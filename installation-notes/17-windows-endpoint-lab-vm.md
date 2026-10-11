# 17 - Windows 11 Endpoint Lab VM

Status: **Working**. VM 203 (`win11-intune-lab`) is built, backed up, on a static IP, and enrolled in Microsoft Intune. The Intune, compliance and BitLocker work lives in its own repo, [windows-endpoint-lab](https://github.com/kramer-sb/windows-endpoint-lab). This note covers the Proxmox and network side only.

## Goal

A Windows 11 endpoint to practice endpoint management on (Intune first, other tools later). A Windows VM was also a gap in the lab, which so far only had Linux machines. The evaluation edition runs for 90 days.

## VM settings

| Setting | Value |
| --- | --- |
| VM ID / name | 203 / `win11-intune-lab` |
| Machine | q35 |
| BIOS | OVMF (UEFI), with an EFI disk |
| TPM | v2.0 virtual TPM (needed for Windows 11 and BitLocker) |
| CPU / memory | 4 cores (host CPU type), up to 8 GiB RAM |
| Disk | 100 GiB on `local-lvm` (thin pool), VirtIO SCSI single |
| Network | E1000 on `vmbr0`, so Windows has a working NIC without extra drivers |
| Other | QEMU agent option enabled |
| OS | Windows 11 Enterprise Evaluation, build 26100 |

## What I did

1. **Uploaded the ISOs.** Windows 11 Enterprise Evaluation and the VirtIO driver ISO went into the node's ISO images. Uploads land temporarily in `/var/tmp`, so I checked free space with `df` first (37 GB available at the time) before uploading the large file.
2. **Created the VM** with the settings above. I chose 100 GiB for the disk because the thin pool had plenty of room.
3. **Installed Windows.** The disk list was empty at first, so I loaded the VirtIO SCSI driver from the VirtIO ISO, then the disk appeared. I created a local account for setup.
4. **Finished the VM.** I checked the TPM and Secure Boot status from PowerShell and ran Windows Update.
5. **Took a snapshot and a backup before enrolling.** The snapshot is `clean-install-before-enroll`. Gotcha: a snapshot name has to start with a letter, so my first choice, `00-clean-install-before-enroll`, was rejected. The backup went to the `usb-backup` datastore on the Proxmox Backup Server, so it is client-side encrypted. The USB drive is slow, so the first backup took a long time even though the VM only held about 4 GB of data.
6. **Set up Remote Desktop.** The Proxmox console has no clipboard, so copy and paste does not work in it. I turned on Remote Desktop in the VM and connected with `mstsc`. The first attempt failed. See [17a](17a-windows-vm-gotchas-addendum.md).
7. **Joined the VM to Microsoft Entra ID and enrolled it in Intune.** Details are in the windows-endpoint-lab repo.
8. **Took two more snapshots** as rollback points: `enrolled` (after enrollment) and `bitlocker-static-ip` (after BitLocker and the static IP).
9. **Gave it a static IP.** `10.0.0.60` was the next open increment after Security Onion. To confirm it was free, I pinged it from the Windows host. The replies came from the Tailscale router (`10.0.0.35`) as "Destination host unreachable", meaning nothing answered the router's lookup. In the guest I set IPv4 manually: `10.0.0.60`, mask `255.255.255.0`, gateway `10.0.0.1`, DNS `10.0.0.30`. I also disabled IPv6 on the adapter, as I did on Kali, so the gateway cannot hand out an IPv6 DNS server that bypasses CoreDNS. See [17a](17a-windows-vm-gotchas-addendum.md).
10. **Verified.** `ping 8.8.8.8` (IP and gateway), `ping google.com` (upstream DNS through CoreDNS), `ping gitea.lab` (resolved to `10.0.0.10`, so CoreDNS is in use). Intune still checked in afterward.
11. **Added an Uptime Kuma monitor.** A ping monitor does not work for this VM. From the Kali VM, `ping -c 3 10.0.0.60` timed out, because Windows blocks inbound ping by default. I used a TCP Port monitor instead: friendly name `Windows Endpoint Lab RDP`, hostname `10.0.0.60`, port `3389`. It went green within a minute. The monitor shows down whenever the VM is shut off, for example during a snapshot.

## Open items

- No CoreDNS entry for this VM. It is reached by IP over Remote Desktop, the same as Security Onion.
- Not yet added to the scheduled backup jobs.
- The local admin password is saved in my personal password manager rather than Vaultwarden, since Vaultwarden is not always running.

See also: [functional doc](../functional-docs/windows-endpoint-lab-vm.md).
