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

Assigned IP: `10.0.0.40` (static) [TODO: confirm - following the
increment-of-5 static IP convention used for everything else in the lab;
replace with the real address if a different one was used]

CTID: `999` [TODO: confirm actual CTID assigned]

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

[TODO: note anything that came up during install - IPv6 disable step,
password requirements, any error messages]

## What's next

With PBS running, the next step is adding an actual backup destination.
Starting with a cloud backup to Backblaze B2
(`12-cloud-backups-backblaze-b2.md`), then a removable USB drive
(`13-usb-drive-backups.md`).
