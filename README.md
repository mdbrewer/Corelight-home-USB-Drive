# Corelight@Home — Moving Logs Off a Full SD Card (Raspberry Pi 5)

**Service:** `corelight-softsensor.service` (Corelight Software Sensor)
**Problem:** The Corelight Software Sensor writes all its data to `/var/corelight` by default. On a Raspberry Pi 5 running from an SD card, that log volume grows steadily and can eat up most of the card's space — risking sensor crashes or a corrupted filesystem once it fills up.
**Approach:** Mount an external USB drive at a dedicated path, migrate the existing logs over, repoint the sensor's `Corelight::disk_space` setting at the new location, and make the mount required before the service starts — so logs can never silently fall back to filling up the SD card.

Run everything below on the Pi5 itself (SSH or console), as a user with `sudo` access.

## 0. Pre-flight checks

```bash
sudo systemctl status corelight-softsensor --no-pager
df -h /
sudo ls -la /var/corelight
```

Confirm the service is currently running, check how full the SD card is, and note the owner/permissions on `/var/corelight` (the migration preserves these).

## 1. Stop the sensor

```bash
sudo systemctl stop corelight-softsensor
```

## 2. Mount the drive at a dedicated path and verify it's safe to use

```bash
sudo mkdir -p /mnt/corelight-data
sudo mount /dev/sda1 /mnt/corelight-data
ls -la /mnt/corelight-data
df -h /mnt/corelight-data
```

Adjust the device/partition (`/dev/sda1`) to match whatever drive you've attached — check with `lsblk` if you're not sure. If you see anything other than an empty directory (or just `lost+found`), stop here — that partition may hold data from a prior use and needs a look before reuse.

## 3. Copy the existing logs over

```bash
sudo rsync -aHAX --info=progress2 /var/corelight/ /mnt/corelight-data/
```

`-aHAX` preserves permissions, ownership, hardlinks, ACLs, and extended attributes. This can take a while over USB depending on how much data has accumulated — let it finish.

Verify the copy:

```bash
sudo du -sh /var/corelight /mnt/corelight-data
```

The two sizes should match closely.

## 4. Make the mount permanent (survives reboot)

```bash
sudo blkid /dev/sda1
```

Copy the `UUID` it prints, then:

```bash
echo 'UUID=<your-partition-uuid> /mnt/corelight-data ext4 defaults,nofail,x-systemd.device-timeout=10 0 2' | sudo tee -a /etc/fstab
```

`nofail` keeps the Pi bootable even if the USB drive is ever unplugged. Test the fstab entry:

```bash
sudo umount /mnt/corelight-data
sudo mount -a
df -h /mnt/corelight-data
```

Confirm your drive is mounted there again before continuing.

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

This prevents a race on boot where the service starts before the drive is mounted — which would otherwise silently recreate `/var/corelight` on the SD card.

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

Your root filesystem usage should drop significantly, freeing the SD card back up for the OS and system files it's actually meant for.
