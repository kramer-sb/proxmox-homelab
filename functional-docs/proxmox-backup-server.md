# Proxmox Backup Server (PBS)

Reference for the standalone LXC (`10.0.0.40`) that all off-site backups run through. See `installation-notes/11-proxmox-backup-server-install.md` for how it was installed, `12-cloud-backups-backblaze-b2.md` and `13-usb-drive-backups.md` for how each backup destination was wired up, and `backup-strategy.md` for why it's set up this way.

## What it is

PBS is a free, open-source backup server from the Proxmox team. It sits between PVE and wherever backups actually live, and adds:

- Incremental backups and deduplication (less storage used over time compared to full backups every time)
- File-level restore (pull a single file out of a backup instead of restoring the whole VM/LXC)

PVE cannot write directly to something like an S3 bucket or a manually mounted USB drive. A PBS **datastore** is the bridge: it's a link to the actual storage location, and PVE talks to PBS, not to the storage directly.

## Access

- Web UI: `https://10.0.0.40:8007`
- Login: `root`, realm **Linux PAM standard authentication**
- Root password: stored in Vaultwarden

## Installation

Installed via the community script
(<https://community-scripts.org/scripts/proxmox-backup-server>), followed by
the post-install script (<https://community-scripts.org/scripts/post-pbs-install>) run from the PBS LXC's own shell to remove the subscription nag. Full steps in `installation-notes/11-proxmox-backup-server-install.md`.

## Datastores in this lab

| Datastore | Type | Backing | Purpose |
|---|---|---|---|
| `remote-backup` | S3 | Backblaze B2 bucket via S3 endpoint | Primary off-site cloud backup, encrypted client-side |
| `usb-backup` | Local | `/mnt/usb-backup` (passed-through USB mount) | Secondary off-site backup, on-demand only |

Each datastore is linked into PVE as its own storage object (Datacenter > Storage), authenticated with `root@pam` and the PBS certificate fingerprint (PBS web UI > Certificates tab).

## Restoring

Two ways to restore, both available from the PVE side once a datastore is linked:

- **Full restore**: select the backup under the storage object's Backups list, click Restore. The ID field in the dialog decides what happens. A new, unused ID creates a separate copy next to the original. Using the original's ID overwrites that guest and permanently erases its current disks, so the guest must be stopped first (and not protected).
- **File restore**: select the backup, click **File Restore**, browse the filesystem, download just the file(s) needed. Useful when only one thing broke and a full restore would undo other, wanted changes.

Cloud backups are client-side encrypted, so any restore from them needs the encryption key. Keep a copy outside the lab.

### Testing a restore safely

1. Select a backup, click Restore, and set the CT ID to a spare number such as 9000.
2. Tick **Unique** and leave **Start after restore** unticked, so the copy cannot clash with the original's MAC address or IP.
3. Confirm the task ends with TASK OK, then remove the test container (More, then Remove). Double check the ID before removing.

### Containers that need TUN access (2000 and 2001)

Proxmox skips custom LXC options on restore and prints a warning. After restoring either container, add these two lines to `/etc/pve/lxc/<id>.conf` on the host before starting it:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

## Retention

Set per backup job, on the Retention tab when creating/editing the job in PVE (Datacenter > Backup). Each datastore (local, cloud, USB) has its own independent retention count; see `backup-strategy.md` for the counts in use.

## Encryption

The `remote-backup` (cloud) datastore uses a client-side encryption key, generated when the storage was linked in PVE. Backblaze only ever sees encrypted chunks. The key itself lives in Vaultwarden and as a local downloaded copy; it is not committed to this repo. The USB datastore does not use a separate encryption key, it relies on the drive itself being kept physically secure instead.

## Operational dependency worth knowing

PBS needs working DNS to reach `remote-backup` (Backblaze), same as any other lab service. It queries CoreDNS (`10.0.0.30`) like everything else, so if CoreDNS is ever down, cloud backup jobs will fail even though the local and USB datastores are unaffected (they don't need external DNS at all). See `lab-gotchas.md` for the CoreDNS-related gotchas that came up
while setting this up.