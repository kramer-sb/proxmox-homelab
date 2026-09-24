# 11 - Installing the Proxmox Backup Server (PBS)

Course: Chapter 5.4 - The Proxmox Backup Server

PBS is the piece that actually lets backups leave the Proxmox host. It sits
between PVE and wherever the backup actually lives (cloud bucket, USB drive,
etc.) and adds incremental backups, deduplication, and file-level restore on
top of what the built-in local backup feature offers.

## Install via community script

Grabbed the PBS install script from the community scripts page:

<https://community-scripts.org/scripts/proxmox-backup-server>

Two things the script's warning banner calls out that need to be handled
during the install:

- Set a root password
- Disable IPv6

Ran the script from the Proxmox shell and stepped through the installer,
making sure to hit both of the above.

Used the **Advanced Install** option so the LXC's settings could be set
directly instead of accepting the script's defaults.

Assigned IP: `10.0.0.40` (static), continuing the increment-of-5 convention
used for every other machine in the lab.

CTID: `999`

Hostname: `proxmox-backup-server`

Once the install finished, the script printed a URL for the PBS web UI.

## First login

Signed in to the PBS web UI with:

- Username: `root`
- Realm: **Linux PAM standard authentication**

Same pattern as the PVE login (root user's password, set during the LXC's
own install, authenticating through the underlying Linux account).

## Post-install script (remove subscription nag)

After first login, got the same "no subscription" pop-up seen on PVE. There's
a matching community post-install script for PBS:

<https://community-scripts.org/scripts/post-pbs-install>

Ran this **from the PBS LXC's own shell**, not from PVE (Administration >
Shell in the PBS web UI). Answered yes to all prompts. Script finished by
rebooting the PBS LXC; signed back in afterward with no more nag.

## Quick tour of what's in PBS

- **Datastore** tab - where storage destinations for backups get added
  (this is where the cloud and USB backups get configured later).
- **S3 Endpoints** tab - connects PBS to S3-compatible cloud storage
  (used for the Backblaze B2 backup).
- **Certificates** tab - PBS's self-signed HTTPS cert. The fingerprint from
  here is needed later when linking a PBS datastore back into PVE.

## Notes / gotchas

**DNS Server field during Advanced Install.** The advanced installer prompts
for a "DNS Server IP" for the new container. This isn't asking for a new
address to assign following the lab's `+5` static IP scheme, it's asking
which existing DNS server the container should use to resolve hostnames,
the same way every other lab LXC points at CoreDNS (`10.0.0.30`).

Misread this the first time and entered `10.0.0.45` (continuing the
increment-of-5 pattern used for assigning the container its *own* address),
which isn't a real device on the network. This broke all DNS resolution
inside the PBS container from the start, visible in the install log itself:

```
APT repository DNS resolution failed in container, injecting public DNS servers
```

That workaround was enough to finish the package install, but didn't fix DNS
going forward. It surfaced later as a "Bad Request, failed to list buckets"
error when trying to add the S3 datastore (see
`12-cloud-backups-backblaze-b2.md`), which took a while to trace back to
this. Fixed from the Proxmox host shell:

```
pct set 999 --nameserver 10.0.0.30
pct reboot 999
```

Confirmed the fix with `cat /etc/resolv.conf` inside the container (should
show `10.0.0.30`), then `curl -I https://<some external host>` to confirm
resolution actually works.

**Lesson for next time:** the DNS Server field in the advanced installer
always wants `10.0.0.30` (CoreDNS), never the next number in the
static-IP sequence.

## What's next

With PBS running, the next step is adding an actual backup destination.
Starting with a cloud backup to Backblaze B2
(`12-cloud-backups-backblaze-b2.md`), then a removable USB drive
(`13-usb-drive-backups.md`).
