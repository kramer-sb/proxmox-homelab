# 12 - Cloud Backups via Backblaze B2

Course: Chapter 5.5 - Cloud Backups

Builds the first off-site leg of the 3/2/1 strategy: PBS on `10.0.0.40`
pushing encrypted backups out to a Backblaze B2 bucket over S3.

## Creating the B2 bucket

Created a free Backblaze account at
<https://www.backblaze.com/sign-up/cloud-storage>, then Buckets > **Create
new Bucket**.

Settings used:

- Bucket name: `jht-proxmox-backups`
- Bucket type: **Private**
- Default encryption: **disabled** (PBS applies its own client-side
  encryption instead - no reason to pay for/rely on Backblaze's)
- Object lock: **disabled**

Wrote down the bucket name and the bucket's endpoint, both needed below. The endpoint for this bucket was `s3.us-east-005.backblazeb2.com`, note the region code embedded in it (`us-east-005`), **it matters later.**

## Backblaze application key

Application Keys page > **Add a New Application Key**:

- Name: `pbs-key`
- Allow access to Buckets: **all**
- Type of Access: **Read and Write**

Clicked **Create New Key**. Saved the `keyID` and `applicationKey` values
immediately since the `applicationKey` is only shown once.

## PBS S3 endpoint

PBS web UI > S3 Endpoints > **Add**:

- S3 Endpoint ID: `remote-backup`
- Endpoint: `s3.us-east-005.backblazeb2.com`
- **Region: `us-east-005`** (matching the region code in the endpoint
  hostname, do not leave this on the default)
- Path Style: checked
- Access Key: Backblaze `keyID`
- Secret Key: Backblaze `applicationKey`
- Skip If-None-Match Header: checked

## PBS datastore

PBS web UI > **Add Datastore**:

- Name: `remote-backup`
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
- Downloaded a local copy, kept outside this repo (on the Windows management
  host, not in any synced/cloud folder)

**This key is never committed to the repo.** Losing it means the cloud
backups become unrecoverable even though the encrypted data itself is still
sitting in the bucket.

## Taking and testing a cloud backup

Datacenter > Backup > **Add**, configured against the new `remote-backup`
storage:

- Node: `pve`
- Storage: `remote-backup`
- Schedule: monthly
- Selection: all current VMs/LXCs (Gitea, Vaultwarden, Uptime Kuma, CoreDNS,
  Kali, ts-router, PBS itself)

Retention: kept the last 3 backups.

Created the job, then selected it and clicked **Run now** to test
immediately rather than waiting on the schedule.

Confirmed the backup landed by checking two places:

- PBS web UI > `remote-backup` > Content > **Reload** - showed the backed-up VMs/LXCs
- Backblaze web UI > **Browse Files** - showed the same data as encrypted, unreadable chunks (expected - only PBS/PVE with the encryption key can make sense of it)

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

**"Bad Request (400), failed to list buckets" when adding the datastore.**

<small>*Note: when I left the session before, I shut down every app, including the CoreDNS. When I started this session, it was still off, so here is a reminder that I need to make sure where things stand operationally when I start up!*</small>

Hit this the first time through the Add Datastore screen, with the bucket dropdown just spinning and failing. Worked through this roughly in order:

1. Checked the application key's bucket scope in Backblaze. It was correctly
   set to all buckets with full read/write, so this wasn't it.
2. Compared the Endpoint field in the PBS S3 Endpoint config against the
   actual bucket endpoint shown in Backblaze; they matched.
3. **The actual cause: the Region field on the S3 Endpoint was left on its
   default (`us-west-1`) instead of being set to match the bucket's real
   region (`us-east-005`, visible in the endpoint hostname itself).** S3
   request signing includes the region, so a mismatched region gets
   rejected by Backblaze as a bad request, this produces the exact same
   error as a bad key would, which is what made it confusing to track down.
   Fixed by explicitly typing `us-east-005` into the Region field on the S3
   Endpoint (PBS doesn't infer it from the endpoint hostname automatically).
4. Setting the region fixed the signing issue, but the error persisted
   because of a second, unrelated problem: DNS. See below.

**DNS resolution was broken inside the PBS container the whole time.**
`curl -I https://s3.us-east-005.backblazeb2.com` from the PBS LXC shell
returned `Could not resolve host`. Root cause was two-fold:

- The PBS LXC's nameserver had been set to `10.0.0.45` instead of
  CoreDNS's actual address (`10.0.0.30`) during install, see the DNS gotcha
  in `11-proxmox-backup-server-install.md`. Fixed with
  `pct set 999 --nameserver 10.0.0.30` and a reboot.
- Even after that fix, resolution still failed with `host unreachable`, a
  routing-level error rather than a DNS answer. Turned out the CoreDNS LXC
  itself (`10.0.0.30`) was **stopped**, visible as a greyed-out icon next to
  it in the Proxmox server view versus the solid icons on every running
  container. Nothing was listening at that address at all. Started it back
  up in the Proxmox web UI, confirmed "Start at boot" was also enabled so
  a host reboot doesn't silently take it down again.

Once both of those were fixed, `curl -I` against the Backblaze endpoint
returned a normal HTTP response, and the datastore picked up the bucket
immediately with no further changes needed. The application key and bucket
config had been correct from the start; every symptom traced back to the
region field and the DNS chain.

**Lesson for next time:** a generic-looking "Bad Request" from an S3-style
API can hide more than one problem stacked on top of each other (a config
mismatch plus a network issue). Test raw connectivity (`curl`) from the
actual container involved before assuming the credentials or bucket
settings are wrong.
