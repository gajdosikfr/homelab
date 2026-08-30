# Storage Migration: 2 TB NTFS to 8 TB ext4

## Overview

The original 2 TB NTFS media disk was almost full, so I replaced it with an 8 TB Seagate IronWolf drive. The old disk originally came from my gaming PC, which is why it was formatted as NTFS. My Jellyfin library was getting bigger, so replacing it was only a matter of time.

My HP ProDesk has two SATA ports but only one SATA power connector. I could have bought a SATA power splitter, but the original wires are quite thin and I could not find their current rating. Both 3.5-inch drives would be under continuous load for several hours during the transfer, with an additional short current spike when they spin up. It would probably work without any problem, but I was not comfortable risking the cable or the motherboard connector just to save some transfer time. On this model, power for the SATA drives comes from the motherboard rather than directly from the power supply, so transferring the data over the network seemed like the safer option.

The old disk using NTFS was actually helpful in this case because I could connect it directly to my Windows desktop and share it over the network. Another option was an old SATA-to-USB adapter from AliExpress, but it only supports USB 2.0 and transferring almost 2 TB through it would take a very long time. I also did not really trust the cheap adapter with such a long transfer, and I had lost the 12 V power supply required for a 3.5-inch drive anyway.

The goal was to keep the existing `/mnt/data` mountpoint. Jellyfin mounts this directory as `/media`, so keeping the same host path meant that no changes to the Jellyfin libraries or Docker Compose configuration were required.

## Migration plan

1. Stop Jellyfin and disable its automatic restart.
2. Disable the old disk entry in `/etc/fstab`. - 
3. Replace the physical disk.
4. Create a GPT partition table and an ext4 filesystem.
5. Mount the new disk again at `/mnt/data`.
6. Copy the media from the old disk over the local network.
7. Verify the transfer and start Jellyfin again.

## Stopping Jellyfin

Jellyfin was stopped before removing the old disk. Its restart policy was temporarily disabled to prevent it from starting with an empty `/mnt/data` directory after reboot.

```bash
sudo docker update --restart=no jellyfin
sudo docker stop jellyfin
```

## Removing the old disk

The old disk used a device-based entry in `/etc/fstab`:

```fstab
/dev/sda1 /mnt/data ntfs defaults 0 0
```

This entry was temporarily commented out before shutting down and replacing the disk.If you don´t do this you can cause system won´t boot, trust me I have experinced this one and take me some time to recover system. I was trying to set automount with this ntfs disk on RHEL and there were no NTFS drivers. After reboot the system won´t boot.


This time i find out you can verify fstab and also try to mount.

## Preparing the new disk

The new disk was identified before making any changes:

```bash
lsblk -o NAME,MODEL,SERIAL,SIZE,FSTYPE,MOUNTPOINTS
```

A GPT partition table and one properly aligned partition were created across the whole disk:

```bash
sudo parted --script --align optimal /dev/sda \
  mklabel gpt \
  mkpart primary ext4 1MiB 100%
```

The partition was formatted as ext4:

```bash
sudo mkfs.ext4 -L media -m 0 /dev/sda1
```

The `-m 0` option disables the default reserved space for root. This is appropriate for a dedicated media disk where reserving five percent would waste several hundred gigabytes.

## Mount configuration

The filesystem UUID was obtained with:

```bash
sudo blkid /dev/sda1
```

The new disk was added to `/etc/fstab` using its UUID instead of a device name:

```fstab
UUID=bd19ec2d-9fea-490a-902b-854b5b705ec9 /mnt/data ext4 defaults 0 2
```

The systemd configuration was verified, reloaded and the disk was mounted:

```bash
sudo findmnt --verify --verbose
sudo systemctl daemon-reload
sudo mount -a
sudo chown gg:gg /mnt/data
```

Using a UUID makes the mount independent of whether the kernel assigns the disk a different device name in the future. This happend to me in VirtualBox while making course so UUID is better choice. 

## Data transfer

The old NTFS disk was connected to a Windows PC and shared over SMB with read-only access. A temporary Windows account was used because anonymous SMB access was rejected, even if password protected sharing was turned off. This happends in Windows all the time.

The Windows share was mounted read-only using SMB 3.0. The uid and gid options ensured that the mounted files were accessible to the gg user:

```bash
sudo mount -t cifs //<windows-ip>/<share-name> /mnt/olddata \
  -o username=<transfer-user>,ro,vers=3.0,uid=1000,gid=1000,iocharset=utf8
```

The transfer was run inside a named tmux session so that it could continue after disconnecting from SSH:

```bash
tmux new -s disk-copy
```

Only the media directories were copied:

```bash
rsync -avh --info=progress2 --partial --append-verify --stats \
  "/mnt/olddata/Film" \
  "/mnt/olddata/Seriály" \
  /mnt/data/
```

The transfer speed over the gigabit network was approximately 90 MB/s.

## Final steps

After the transfer, a second rsync dry run was used to confirm that no files were missing:

```bash
rsync -avhn \
  "/mnt/olddata/Film" \
  "/mnt/olddata/Seriály" \
  /mnt/data/
```

The temporary SMB mount and Windows transfer account were then removed.

Jellyfin's original restart policy was restored and the container was started:

```bash
sudo docker update --restart=unless-stopped jellyfin
sudo docker start jellyfin
```

Because the new disk uses the same `/mnt/data` mountpoint, Jellyfin continued using the existing library paths without any Compose changes.


## Troubleshooting

During the migration, an earlier rsync configuration preserved permissions from the CIFS/SMB source. Some directories therefore ended up without write permission for the owner, even though gg remained the owner.

The affected permissions were corrected after the transfer. 

```bash
sudo chmod -R u+rwX "/mnt/data/Film" "/mnt/data/Seriály"
```

For future migrations from Windows/SMB sources, permissions should not be preserved unless there is a specific reason to do so.