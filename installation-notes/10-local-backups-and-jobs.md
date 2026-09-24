# 10 - Local Backups and Backup Jobs

Course: Chapter 5.2 - Local Backups in Proxmox

Before doing anything with off-site or cloud backups, this covers the built-in
Proxmox backup feature: taking a manual backup, restoring from it, and then
automating the process with a scheduled backup job.

## Taking an on-demand backup

Went to the Gitea LXC (`10.0.0.10`) > Backup tab > **Backup now**.

In the pop-up, selected the `local` storage object (the same one that holds
CT templates and VM ISOs) and left the rest at default. Clicked **Backup**.

Confirmed the task viewer showed `TASK OK` once it finished. The backup then
showed up in two places:

- Gitea LXC > Backup tab
- Datacenter > `local (pve)` > Backups

## Restoring a backup

To test restoring, intentionally broke the Gitea config to have something to
fix:

```
# Opened a shell on the Gitea LXC and edited:
/etc/gitea/app.ini
```

Deleted a large chunk of the file to break the app and saved it.

Restarted the service to make the breakage visible:

```
systemctl restart gitea
```

Confirmed `https://gitea.lab` failed to load.

To restore:

1. Shut down the Gitea LXC.
2. PVE > `gitea` > Backup, selected the backup taken above, clicked **Restore**.
3. Left all restore options at default (note: restoring into a *running*
   LXC/VM creates a new copy instead of overwriting; shutting it down first
   is what makes this an in-place overwrite).
4. Clicked **Restore**, waited for the task to finish, powered the LXC back
   on.

Confirmed `https://gitea.lab` was back to normal after the restore, no
further DNS/Caddy reconfiguration was needed since restoring the LXC
preserved its existing static IP and config.

## Creating a scheduled backup job

Manual "on-demand" backups don't scale once there are more than a couple of
services, so set up a recurring job instead.

Datacenter > Backup tab > **Add**. Configured:

- Node: `pve`
- Storage: `local`
- Schedule: daily at 21:00
- Selection: all current lab services (Gitea, Vaultwarden, Uptime Kuma,
  CoreDNS, Kali, ts-router)

Retention tab: set to keep the last 7 backups (one week of local backups).

Clicked **Create**. Job showed up in the Datacenter Backup tab list.

Tested it immediately with **Run now** rather than waiting for the schedule.
Confirmed `TASK OK` after a few minutes, then checked Datacenter > `local
(pve)` > Backups and saw backups for every selected VM/LXC.

## Notes / gotchas

Nothing unexpected here, this part matched the course exactly. The gotchas
in this chapter all showed up later, once PBS and off-site backups entered
the picture, see `11-proxmox-backup-server-install.md`,
`12-cloud-backups-backblaze-b2.md`, and `13-usb-drive-backups.md`.

## What's next

Local backups only protect against user error and software mistakes, not
against the Proxmox host itself failing, being stolen, or destroyed. Next:
working out a proper 3/2/1 backup strategy (`functional-docs/backup-strategy.md`)
and standing up the Proxmox Backup Server to get an actual off-site copy.
