# Backup Strategy (3/2/1 Rule)

Not tied to a single service; this is the reasoning behind how backups are
laid out across the whole lab. See `proxmox-backup-server.md` for the tool
that implements it, and `installation-notes/10-local-backups-and-jobs.md`,
`12-cloud-backups-backblaze-b2.md`, and `13-usb-drive-backups.md` for how
each piece was actually set up.

## The rule

Popularized by photographer Peter Krogh: keep **3** total copies of the
data, on **2** different types of storage media, with **1 or more** copies
kept off-site.

The modern reading of the "2": rather than two different *types* of media
(the original 2005 concern, when it wasn't clear which storage formats would
still be usable in 20 years), it now means two different *physical devices*.
All-SSD or all-HDD is fine, as long as it's not all the same box - three
backups on one machine are all lost if that machine is lost.

## How this maps onto the lab

| Copy | What it is | Device | Off-site? |
|---|---|---|---|
| Production | The live VM/LXC virtual disks | Proxmox host | No |
| Local backup | Built-in Proxmox backup, `local` storage | Proxmox host | No |
| Cloud backup | PBS -> Backblaze B2, client-side encrypted | Backblaze (remote) | Yes |
| USB backup (optional) | PBS -> removable USB drive | USB drive | Only when physically taken elsewhere |

Production + local backup alone is a 2/1/0 strategy: two copies, one
device, zero off-site. Adding **either** the cloud backup or the USB backup
closes the gap to a full 3/2/1, because either one adds a second physical
device and (with the USB drive, once it's actually moved elsewhere) an
off-site copy at the same time.

## Cloud vs. USB, and why both exist here

A USB drive kept in the same room as the Proxmox server is not really
off-site; a single event (fire, flood, theft) can take out both at once. It
only becomes a proper off-site copy once it's physically moved somewhere
else (e.g. a friend's house).

Cloud backup is a "properly" off-site copy by default, at the cost of a
recurring fee and trusting a third party with (encrypted) data. USB backup
keeps everything under direct control and has no recurring cost, at the
cost of needing to remember to actually take the drive somewhere else.

Both are implemented in this lab:

- **Cloud** (Backblaze B2) is the primary off-site copy, running on a
  schedule.
- **USB** is a secondary, fully offline copy, taken on-demand rather than on
  a schedule (see `installation-notes/13-usb-drive-backups.md` for why it's
  on-demand instead of scheduled).

## Retention

Local, cloud, and USB backups each have their own retention policy so
storage doesn't grow unbounded:

- Local: keep last [TODO: confirm count] backups
- Cloud: keep last [TODO: confirm count] backups
- USB: keep last [TODO: confirm count] backups

Retention counts are per-datastore and configured on the backup job itself
in Proxmox (Retention tab).

## Encryption

Cloud backups are encrypted client-side by PBS before they ever leave the
lab, using a key generated when the `remote-backup` storage was linked. This
key is stored in Vaultwarden and as a local downloaded copy; it is **never**
committed to this repo. Losing the key means the encrypted data in Backblaze
is unrecoverable even though it's still physically present in the bucket -
back the key up like it's as important as the data itself, because it is.
