# Windows 11 Endpoint Lab VM

Reference for VM 203 (`win11-intune-lab`). The raw build log is in [17-windows-endpoint-lab-vm.md](../installation-notes/17-windows-endpoint-lab-vm.md), and the problems hit are in [17a](../installation-notes/17a-windows-vm-gotchas-addendum.md). The Intune work is in the [windows-endpoint-lab](https://github.com/kramer-sb/windows-endpoint-lab) repo.

| Item | Value |
| --- | --- |
| IP | `10.0.0.60` (static) |
| Access | Remote Desktop (`mstsc`) to `10.0.0.60`, or the Proxmox console |
| DNS name | none (accessed by IP) |
| Evaluation license | 90 days |

## Create the VM

1. Upload the Windows 11 Enterprise Evaluation ISO and the VirtIO driver ISO to the node's ISO images. Check free space on the node first.
2. Create a VM with: machine q35, BIOS OVMF (UEFI) with an EFI disk, a v2.0 TPM, 4 cores, up to 8 GiB RAM, a 100 GiB VirtIO SCSI single disk on `local-lvm`, and an E1000 NIC on `vmbr0`. Attach both ISOs as CD/DVD drives.
3. Start the VM and run Windows setup. When the disk list is empty, choose Load driver and select the VirtIO SCSI driver (`vioscsi`, Windows 11, amd64) from the VirtIO ISO.
4. After Windows installs, run Windows Update.
5. **Detach both ISOs:** VM, Hardware, edit each CD/DVD drive, Do not use any media. This is required for BitLocker to work.

## Remote Desktop

1. In the VM, Settings, System, Remote Desktop, turn it on.
2. Reboot the VM. The listener may not exist until after a full reboot.
3. From the Windows host, open `mstsc` and connect to the VM's IP with the local admin account (`.\<username>`).

## Static IP and DNS

In the guest (Settings, Network & internet, Ethernet, IP assignment, Edit, Manual):

| Setting | Value |
| --- | --- |
| IPv4 address | `10.0.0.60` |
| Subnet mask | `255.255.255.0` |
| Gateway | `10.0.0.1` |
| DNS | `10.0.0.30` only |
| IPv6 | disabled on the adapter |

PowerShell equivalent, as administrator:

```
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Disabled
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.60 -PrefixLength 24 -DefaultGateway 10.0.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.30
Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6
```

Run it from the Proxmox console, since the connection drops when the address changes. Verify with `ping 8.8.8.8`, `ping google.com` and `ping gitea.lab`.

## Monitoring

Uptime Kuma, TCP Port monitor: friendly name `Windows Endpoint Lab RDP`, hostname `10.0.0.60`, port `3389`. Use a TCP monitor and not a ping monitor, because Windows blocks inbound ping by default. The monitor shows down while the VM is shut off, for example during snapshots.

## Snapshots and backups

Snapshot names must start with a letter. Current snapshots, oldest first: `clean-install-before-enroll`, `enrolled`, `bitlocker-static-ip`, `before-firewall-settings`, `firewall-settings-applied`, `before-sysmon`, `sysmon-installed`, `before-conditional-access`. The only on-demand backup to the `usb-backup` datastore (Proxmox Backup Server, encrypted) is from before enrollment, so it does not include the policies, BitLocker or Sysmon. The VM is not part of the scheduled local job because the `local` storage is too small for a 100 GiB disk. See [proxmox-backup-server.md](proxmox-backup-server.md).
