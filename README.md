# Corelight@Home (netmon) — Moving Logs to the Attached Hard Drive

**Host:** `netmon` (Raspberry Pi 5), user `pi`
**Service:** `corelight-softsensor.service` (Corelight Software Sensor)
**Problem:** All sensor data lives in `/var/corelight` (16 GB and growing) on the 30 GB SD card, which is already at 79% used.
**Drive:** WD Elements USB HDD, `/dev/sda`, 931 GB total — `sda1` is a 4 GB unformatted swap-type partition, `sda2` is a 927.5 GB ext4 partition (UUID `c378d2f9-2cca-43ba-b6cf-efd36f471cb5`), not yet mounted.
**Approach:** Mount `sda2` at a dedicated path (`/mnt/corelight-data`), migrate the existing 16 GB, repoint the sensor's `Corelight::disk_space` setting at it, and make the mount required before the service starts.

Run everything below on the Pi5 itself (SSH or console), as `pi` with `sudo`.

## 0. Pre-flight checks

```bash
sudo systemctl status corelight-softsensor --no-pager
df -h /
sudo ls -la /var/corelight
```

Confirm the service is currently running and note the owner/permissions on `/var/corelight` (the migration preserves these).

## 1. Stop the sensor

```bash
sudo systemctl stop corelight-softsensor
```

## 2. Mount the drive at a dedicated path and verify it's safe to use

```bash
sudo mkdir -p /mnt/corelight-data
sudo mount /dev/sda2 /mnt/corelight-data
ls -la /mnt/corelight-data
df -h /mnt/corelight-data
```

If you see anything other than an empty directory (or just `lost+found`), stop here — that partition may hold data from a prior project and needs a look before reuse.

## 3. Copy the existing 16 GB over

```bash
sudo rsync -aHAX --info=progress2 /var/corelight/ /mnt/corelight-data/
```

`-aHAX` preserves permissions, ownership, hardlinks, ACLs, and extended attributes. This will take a while over USB for 16 GB — let it finish.

Verify the copy:

```bash
sudo du -sh /var/corelight /mnt/corelight-data
```

The two sizes should match closely.

## 4. Make the mount permanent (survives reboot)

```bash
echo 'UUID=c378d2f9-2cca-43ba-b6cf-efd36f471cb5 /mnt/corelight-data ext4 defaults,nofail,x-systemd.device-timeout=10 0 2' | sudo tee -a /etc/fstab
```

`nofail` keeps the Pi bootable even if the USB drive is ever unplugged. Test the fstab entry:

```bash
sudo umount /mnt/corelight-data
sudo mount -a
df -h /mnt/corelight-data
```

Confirm `/dev/sda2` is mounted there again before continuing.

## 5. Point the sensor at the new location

Edit `/etc/corelight-softsensor.conf` and change the (currently commented-out) disk_space line:

```bash
sudo sed -i 's|^#Corelight::disk_space.*|Corelight::disk_space        /mnt/corelight-data|' /etc/corelight-softsensor.conf
grep disk_space /etc/corelight-softsensor.conf
```

You should see `Corelight::disk_space        /mnt/corelight-data` uncommented. All the relative paths in that config (`batch_log_disk_path ./logs`, `extracted_files_disk_directory ./extracted_files`, etc.) resolve against this base directory automatically — no other config changes needed.

## 6. Guarantee the mount is up before the service starts

```bash
sudo systemctl edit corelight-softsensor.service
```

In the editor that opens, add:

```ini
[Unit]
RequiresMountsFor=/mnt/corelight-data
```

Save and exit, then:

```bash
sudo systemctl daemon-reload
```

This prevents a race on boot where the service starts before the USB drive is mounted (which would otherwise silently recreate `/var/corelight` on the SD card).

## 7. Start the sensor and verify

```bash
sudo systemctl start corelight-softsensor
sudo systemctl status corelight-softsensor --no-pager
sudo journalctl -u corelight-softsensor -n 50 --no-pager
```

Then confirm new logs are actually landing on the drive:

```bash
ls -la /mnt/corelight-data/zeek /mnt/corelight-data/suricata
ls -la /mnt/corelight-data/logs/$(date +%F)/
```

Wait 5-10 minutes and check that new files continue appearing there, and that nothing new is being written to the old location:

```bash
find /var/corelight -newer /etc/corelight-softsensor.conf
```

That should return nothing once the cutover is confirmed.

## 8. Reclaim the SD card space

Once you've confirmed the sensor is healthy on the new mount (give it 15-30 minutes, or a full log rotation cycle):

```bash
sudo rm -rf /var/corelight/*
df -h /
```

Root filesystem usage should drop from 79% down to roughly the low 20s (16 GB freed on a 30 GB card).

## Optional: put the spare 4 GB swap partition to use

`sda1` is already partitioned as a Linux swap type but has never been formatted:

```bash
sudo mkswap /dev/sda1
sudo blkid /dev/sda1   # copy the UUID it prints
echo 'UUID=<paste-uuid-here> none swap sw,nofail 0 0' | sudo tee -a /etc/fstab
sudo swapon -a
swapon --show
```

Not required for the log migration, just a freebie since the partition already exists on the drive. USB swap is slow, so treat it as a safety net rather than a performance feature — with the sensor's 6500 MB memory_limit_mb setting, an extra 4 GB of swap buys headroom before an OOM kill rather than daily use.
</content>
