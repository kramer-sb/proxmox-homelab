# 18 - Backup Review, Retention Fix and Encryption Key Backup

Course: follow-up to Chapter 5 (backups), done 2026-10-12 and 2026-10-13 while adding VMs to the lab.

Reviewed what the backup jobs actually cover after adding the Kali and Windows VMs, backed up VM 201 (kali-python) to the USB drive for the first time, fixed a missing retention policy on the local job, and found and saved a second encryption key that was not in my notes.

## 1. First USB backup of VM 201 (kali-python)

VM 201 > Backup > **Backup now**, Storage `usb-backup`, Mode **Snapshot** so the VM could stay up. It took about 14 hours for an 80 GiB disk. The USB drive is slow, so I started it as the last thing of the day.

**Why it backs up blank space:** Proxmox reads the whole virtual disk block by block and does not ask the guest which parts are in use. As far as I understand it, PBS stores all-zero chunks as almost nothing, so the stored backup stays small, but the read still takes time. Later backups of a running VM should be faster because PBS only reads blocks that changed, though stopping the VM in between resets that.

**What I checked:** VM 201 > Hardware > Hard Disk (scsi0) already had `discard=on` and `ssd=1`, so nothing needed changing there.

**What I ran in the Kali VM before the next backup:**

```
sudo fstrim -av
```

Result: `/: 54.4 GiB (58366566400 bytes) trimmed on /dev/sda1`. About two thirds of the 80 GiB disk was free space the host had not been told about. Run this before each backup of a Linux VM.

**VM 202 (security-onion) was not backed up.** I have not customized it, a reinstall is about an hour, and the traffic mirroring fix lives on the Proxmox host (see 16b), not inside the VM. A 200 GiB disk to this USB drive would take far longer than 201. Revisit once real work is in it.

## 2. What the backup jobs cover

Checked in Datacenter > Backup:

| Job | Covers | Retention |
| --- | --- | --- |
| Local, daily at 21:00 | Everything except Security Onion and the Windows VM | None set (this was the problem, see below) |
| Cloud (`remote-backup`), monthly | LXCs only, VMs excluded on purpose | Keep last 3 |
| USB (`usb-backup`), on demand | Same selection as local | Keep last 3 |

Security Onion and the Windows VM are missing from the local and USB jobs because they were added after the jobs were created, not because of a decision.

## 3. Retention fix on the local job

**Problem:** note 10 says the local job keeps the last 7, but the job had no retention at all. With no retention on the job, Proxmox falls back to the storage setting, which keeps everything.

**Size check.** `local` is a 100.86 GB directory storage with 64.21 GB used, and it also holds ISOs and container templates. One nightly run is about 13.5 GiB:

| Guest | Size per night |
| --- | --- |
| kali-python (201) | 7.29 GiB |
| proxmox-backup-server (999) | 1.97 GiB |
| jelu (1003) | 1.96 GiB |
| uptimekuma (1002) | 676 MiB |
| vaultwarden (1001) | 611 MiB |
| ts-router (2001) | 421 MiB |
| gitea (1000) | 410 MiB |
| coredns (2000) | 257 MiB |

Three nights were already on disk (about 40 GiB). Without retention, `local` would have filled around 2026-10-15. Keeping 7 would not fit either, at roughly 95 GiB.

**Fix:** Datacenter > Backup > local job > Edit > Retention > **Keep Last = 3**. Nothing is deleted when saving. The prune runs after each guest's next backup.

**Windows VM left out of the local job:** a 100 GiB disk backed up daily would not fit in what is left. It stays on the USB backup (and cloud, if I ever want it there).

**Lesson:** size retention from the nightly total times the number kept, and check the storage's free space before picking a number.

## 4. A second encryption key, not in my notes

`proxmox-backup-server.md` said the USB datastore had no encryption key. That was wrong. The VM 203 backup on `usb-backup` showed as Encrypted, and the September 24 container backups on the same storage did not.

On the Proxmox host:

```
ls -l /etc/pve/priv/storage/
sha256sum /etc/pve/priv/storage/*.enc
```

There are two `.enc` files, `remote-backup.enc` (created 2026-09-22) and `usb-backup.enc` (created 2026-10-02), and their fingerprints are different. So there are two separate keys, and the USB one was added after the first USB backups. Backups made before 2026-10-02 are not encrypted, and backups made after it are.

Both keys live only on the Proxmox host, and Vaultwarden is also on that host, so a copy there would be lost with it. I saved both keys to a password manager that is not hosted on the lab, then verified each copy against the original by comparing SHA-256 fingerprints with newlines stripped. Both matched. The steps are in [pbs-encryption-key-backup.md](../functional-docs/pbs-encryption-key-backup.md).

## 5. Docs corrected

- `proxmox-backup-server.md`: the USB datastore does have its own key, added 2026-10-02.
- `backup-strategy.md`: local retention is now keep 3, with the reasoning, and a copy of each key is needed off the host.
- `windows-endpoint-lab-vm.md`: lists all eight snapshots and notes that the only USB backup of VM 203 predates enrollment.
- `README.md`: Chapter 5 summary says keep 3.

## Still to do

- Take a fresh USB backup of VM 203 once the app work is done. The current one has no policies, BitLocker or Sysmon in it. Expect it to take many hours, so start it overnight.
- Decide whether Security Onion should ever go into a backup job.
- Confirm after tonight's 21:00 run that the local job pruned down to three per guest.
