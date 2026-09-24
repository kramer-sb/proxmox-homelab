# 12 - Cloud Backups via Backblaze B2

Course: Chapter 5.5 - Cloud Backups

Builds the first off-site leg of the 3/2/1 strategy: PBS on `10.0.0.40`
pushing encrypted backups out to a Backblaze B2 bucket over S3.

## Creating the B2 bucket

Created a free Backblaze account at
<https://www.backblaze.com/sign-up/cloud-storage>, then Buckets > **Create
new Bucket**.

Settings used:

- Bucket name: `jht-proxmox-backups` [TODO: confirm actual bucket name
  chosen]
- Bucket type: **Private**
- Default encryption: **disabled** (PBS applies its own client-side
  encryption instead - no reason to pay for/rely on Backblaze's)
- Object lock: **disabled**

Wrote down the bucket name and bucket endpoint, both needed below.

## Backblaze application key

Application Keys page > **Add a New Application Key**:

- Name: [TODO: name chosen]
- Allow access to Buckets: **all**
- Type of Access: **Read and Write**

Clicked **Create New Key**. Saved the `keyID` and `applicationKey` values
immediately since the `applicationKey` is only shown once.

## PBS S3 endpoint

PBS web UI > S3 Endpoints > **Add**:

- S3 Endpoint ID: `remote-backup` [TODO: confirm name used]
- Endpoint: (Backblaze bucket endpoint from above)
- Path Style: checked
- Access Key: Backblaze `keyID`
- Secret Key: Backblaze `applicationKey`
- Skip If-None-Match Header: checked

## PBS datastore

PBS web UI > **Add Datastore**:

- Name: `remote-backup` [TODO: confirm name used]
- Datastore Type: **S3**
- Local Cache: `/remote-backup` (a path on the PBS LXC's own disk used for
  cache data, not the actual backup storage)
- S3 Endpoint ID: `remote-backup` (the endpoint created above)
- Bucket: `jht-proxmox-backups`

Bucket names auto-populate once the S3 endpoint is selected correctly; if
they don't show up, it means something in the S3 endpoint config is wrong.

## Linking the datastore into PVE

Back in the PVE web UI: Datacenter > Storage > Add > **Proxmox Backup
Server**.

- ID: `remote-backup`
- Server: `10.0.0.40`
- Username: `root@pam` (the `@pam` matters, it's a Linux account)
- Password: PBS LXC's root password
- Datastore: `remote-backup` (the PBS datastore name from above)

Needed the PBS certificate fingerprint to finish this: PBS web UI >
Certificates > default certificate > copied the fingerprint > pasted into the
Fingerprint field on the PVE side. This is what lets PVE confirm it's
actually talking to the real PBS instance before sending backups.

### Encryption key

Encryption tab on the same storage creation screen > **Auto-generate a
client encryption key** > **Add**. This is the key that encrypts backups
before they leave the lab for Backblaze; Backblaze (or anyone with access to
their servers) never sees unencrypted data.

Saved the generated key in more than one place, same logic as the 3/2/1 rule
itself:

- Added it to Vaultwarden (`https://passwords.local`)
- Downloaded a local copy [TODO: note where the local copy was actually
  saved - this should NOT go in the repo]

**This key is never committed to the repo.** Losing it means the cloud
backups become unrecoverable even though the encrypted data itself is still
sitting in the bucket.

## Taking and testing a cloud backup

Datacenter > Backup > **Add**, configured against the new `remote-backup`
storage:

- Node: `pve`
- Storage: `remote-backup`
- Schedule: [TODO: note schedule chosen - course example is monthly]
- Selection: [TODO: which VMs/LXCs]

Retention: kept the last [TODO: note retention count - course example keeps
3] backups.

Created the job, then selected it and clicked **Run now** to test
immediately rather than waiting on the schedule.

Confirmed the backup landed by checking two places:

- PBS web UI > `remote-backup` > Content > **Reload** - showed the backed-up
  VMs/LXCs
- Backblaze web UI > **Browse Files** - showed the same data as encrypted,
  unreadable chunks (expected - only PBS/PVE with the encryption key can make
  sense of it)

### Testing file-level restore

Reused the same Gitea `/etc/gitea/app.ini` breakage as the local-backup test
in `10-local-backups-and-jobs.md`, but this time restored just the one file
instead of the whole LXC.

PVE > `remote-backup` > Backups > selected the Gitea backup > **File
Restore**. Browsed to `/etc/gitea/app.ini`, clicked **Download**, and
confirmed the downloaded file had the correct pre-breakage contents.

This confirmed two things at once: file-level restore works, and the cloud
backup round-trip (write to B2, read back from B2) works.

## Where this leaves the 3/2/1 strategy

- 3 copies: production data, local backup, cloud backup
- 2 storage types: local Proxmox disk, Backblaze B2
- 1 off-site copy: the cloud backup

Full 3/2/1 achieved with just the cloud leg. USB backups
(`13-usb-drive-backups.md`) are optional on top of this - see
`functional-docs/backup-strategy.md` for the reasoning on cloud vs. USB.

## Notes / gotchas

[TODO: capture anything that went wrong - bucket name typos, fingerprint
mismatches, endpoint not populating buckets, etc.]
