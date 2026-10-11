# 17a - Windows VM Gotchas Addendum

*Addendum to [17-windows-endpoint-lab-vm.md](17-windows-endpoint-lab-vm.md)*

Three problems hit while building VM 203. Each one looked like something else.

## 1. Remote Desktop error 0x204 on a fresh VM

**Symptom:** `mstsc` could not connect, with error code 0x204.

**What I suspected first:** Tailscale or the ESET firewall on the Windows host, because the lab is reached through the subnet router.

**Diagnostic path:**

1. From the Kali VM, `nc -zvn -w 3 <vm-ip> 3389` timed out, so the port was not answering from inside the LAN either.
2. `ip neigh` on Kali showed the VM in the neighbor table, so the network path was fine and the VM was up.
3. On the Windows VM, `qwinsta` showed no `rdp-tcp` listener, and `netstat` showed nothing on 3389.
4. `Restart-Service TermService -Force` failed to stop the service.

**Cause:** Remote Desktop had been turned on, but the Remote Desktop service did not create its listener until after a full reboot.

**Fix:** rebooted the VM. `qwinsta` then showed `rdp-tcp` in Listen state and the connection worked.

**Related behavior:** Windows allows one interactive session. While a Remote Desktop session is active, the Proxmox console shows the lock screen. Do not sign in at the console while using Remote Desktop.

## 2. A leftover ISO blocked BitLocker

**Symptom:** the Intune BitLocker policy showed Succeeded for over an hour, but `manage-bde -status C:` showed Protection Off and no key protectors.

**Cause:** the Windows install ISO and the VirtIO ISO were still attached as virtual DVD drives. The BitLocker Management event log repeated event 853: "Failed to enable Silent Encryption. TPM is not available", followed by "BitLocker Drive Encryption detected bootable media (CD or DVD) in the computer". The first half of that message is misleading. The second half is the cause.

**Fix:** in Proxmox, VM 203, Hardware, edit each CD/DVD drive and choose Do not use any media. Proxmox does this while the VM is running. The policy retried by itself within minutes, protection turned on, and event 845 confirmed the recovery key was backed up to Entra ID.

**Lesson:** detach installer ISOs once Windows is installed. It is also good hygiene for any VM.

## 3. `.lab` names failed to resolve after setting a static IP

**Symptom:** after setting DNS to `10.0.0.30`, `ping gitea.lab` returned "could not find host", although `ping google.com` worked.

**Cause:** IPv6 was still on, and Windows asked the ISP's IPv6 DNS server, which does not know `.lab`. `google.com` had resolved to an IPv6 address, which gave it away. This is the same issue documented for the Windows host in [07-home-lab-dns.md](07-home-lab-dns.md).

**Fix:** `Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6`, then `ipconfig /flushdns`. Both `gitea.lab` and `google.com` then resolved over IPv4.
