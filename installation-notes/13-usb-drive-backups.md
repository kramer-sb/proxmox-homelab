# 13 - USB Drive Backups

Course: Chapter 5.6 - USB Drive Backups

Second (optional) off-site backup leg: passing a USB drive through to the PBS
LXC (`10.0.0.40`) as a local datastore, for a fully offline copy of the lab
that never touches the internet.

## Preparing the drive

Drive: Verbatim Pinstripe 128 GB

Partitioned with an ext4 filesystem. Confirmed it was readable from a Linux
system before moving on.

## Finding the drive on Proxmox

From the PVE shell, with the drive plugged in:

```
lsblk        # find the partition name (e.g. sda1)
lsblk -f     # find the UUID of that partition
blkid -U <UUID>   # confirm the /dev path for that UUID
```

Recorded:

- Partition: [TODO: e.g. `sda1`]
- UUID: [TODO: actual UUID - do NOT reuse the course's example UUID
  `56ed1933-ac58-4646-98e4-62649d2c9393`, that's specific to the course
  author's drive]

## Mounting on the Proxmox host

```
mkdir -p /mnt/usb-backup
mount /dev/sda1 /mnt/usb-backup
```

Confirmed the mount worked by listing the directory (saw the default
`lost+found` for a fresh ext4 filesystem).

## Passing the mount into the PBS LXC

Edited the PBS LXC's config on the PVE host:

```
nano /etc/pve/lxc/999.conf   # replace 999 with the actual PBS CTID
```

Added:

```
mp0: /mnt/usb-backup,mp=/mnt/usb-backup
```

Rebooted the LXC to apply it:

```
pct reboot 999
```

Confirmed the mount showed up inside the PBS LXC's own `/mnt`, but reading
from it failed at this point with a permissions error - expected, since
Proxmox hadn't yet been told the PBS LXC is allowed to touch that host
directory.

## Fixing permissions (UID mapping)

Found the UID of the `backup` user *inside* PBS:

```
pct exec 999 -- id backup   # or similar - UID was 34 at time of writing
```

Checked whether the PBS LXC uses the default UID base offset (100,000):

```
grep lxc.idmap /etc/pve/lxc/999.conf
```

No output meant the default 100,000 offset applies. [TODO: confirm this was
the case, or note the actual offset if different]

Calculated the Proxmox-side UID: `100,000 + 34 = 100034`.

Applied ownership and permissions on the host side:

```
chown 100034:100034 /mnt/usb-backup
chmod 750 /mnt/usb-backup
pct reboot 999
```

Verified read/write worked **from inside the PBS LXC shell**, not from PVE:

```
cd /mnt/usb-backup
echo "hello there!" > file.txt
echo "General Kenobi! You are a bold one." >> file.txt
cat file.txt
rm file.txt
```

## Mount/unmount scripts

Since the mount doesn't survive a drive unplug or a host reboot, added two
scripts on the PVE host to make replugging the drive repeatable.

`/usr/local/bin/usb-backup-mount.sh`:

```bash
#!/bin/bash
UUID="<actual USB drive UUID>"
MOUNTPOINT="/mnt/usb-backup"
CTID=999

DEV=$(blkid -U "$UUID")
[ -z "$DEV" ] && { echo "usb-backup: no device for UUID $UUID"; exit 1; }

mountpoint -q "$MOUNTPOINT" && umount -l "$MOUNTPOINT" 2>/dev/null
mount "$DEV" "$MOUNTPOINT" || { echo "usb-backup: mount failed for $DEV"; exit 1; }

echo "usb-backup: mounted $DEV, rebooting CT $CTID"
pct reboot "$CTID"
```

`/usr/local/bin/usb-backup-umount.sh`:

```bash
#!/bin/bash
MOUNTPOINT="/mnt/usb-backup"

mountpoint -q "$MOUNTPOINT" || exit 0
sync
umount "$MOUNTPOINT" || umount -l "$MOUNTPOINT"
echo "usb-backup: unmounted"
```

```
chmod +x /usr/local/bin/usb-backup-mount.sh /usr/local/bin/usb-backup-umount.sh
```

Tested by unmounting, unplugging, replugging, and re-running the mount
script; confirmed read/write still worked from the PBS LXC afterward.

The repeatable process going forward:

1. Plug in the USB drive
2. Run `usb-backup-mount.sh` from the PVE shell
3. Run the backup job
4. Run `usb-backup-umount.sh` from the PVE shell
5. Unplug the drive

## PBS datastore for the USB drive

PBS web UI > **Add Datastore**:

- Name: `usb-backup` [TODO: confirm name used]
- Datastore type: **Local**
- Backing Path: `/mnt/usb-backup`

## PVE storage object for the USB drive

Same pattern as the cloud datastore: Datacenter > Storage > Add > **Proxmox
Backup Server**.

- ID: [TODO: confirm name]
- Server: `10.0.0.40`
- Username: `root@pam`
- Password: PBS LXC's root password
- Datastore: `usb-backup`
- Fingerprint: same PBS certificate fingerprint used for the cloud storage
  link (PBS Certificates tab)

## Taking backups to the USB drive

Deliberately **not** a scheduled job. A scheduled job would mean leaving the
drive plugged in permanently, which defeats the point (no off-site
separation, and a single event like ransomware could take out the local
backup and the "off-site" USB copy at once).

Two ways to trigger a backup on demand:

- Per-VM/LXC: Backup tab > **Backup Now** > select `usb-backup` as the
  target.
- Whole-lab "phantom job": a backup job pointed at `usb-backup` with a
  deliberately long schedule (e.g. once a year) that's never actually meant
  to run on schedule, just triggered manually with **Run now**. Used this
  for backing up everything at once.

Retention on the USB datastore: kept the last [TODO: note retention count -
course example keeps 3] backups.

## Notes / gotchas

[TODO: capture anything that went wrong here - UID mismatches, mount
failures after replug, drive not showing up in `lsblk`, etc. This section in
particular is a good candidate for an addendum file (e.g.
`13a-usb-mount-troubleshooting-addendum.md`) if anything substantial comes
up later]

## Review: repeatable USB backup process

1. Plug in the USB drive to the Proxmox server
2. Run `usb-backup-mount.sh` from a PVE shell
3. Select the backup job in Proxmox and run it with **Run now**
4. Run `usb-backup-umount.sh` from a PVE shell
5. Unplug the USB drive
