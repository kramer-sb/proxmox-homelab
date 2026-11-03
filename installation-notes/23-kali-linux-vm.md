# 23 - Kali Linux VM (kali-linux)

Status: **Working**. A fresh Kali Linux 2026.3 VM runs as VM 207 (`kali-linux`) at `10.0.0.90`, with key-only SSH and RDP through an SSH tunnel. The clean, repeatable version is in [kali-linux.md](../functional-docs/kali-linux.md).

## Goal

HackRange's *Specializing in Linux* course has no hosted lab for the Kali sections. Students install their own Kali VM. I wanted a clean Kali that I can reach the same way as the Python course VM: static IP, `.lab` name, key-only SSH from Windows and a desktop over RDP.

## Decisions

- **A fresh VM, not a clone of VM 201 (`kali-python`).** VM 201 now holds personal state from the Python course: `10.0.0.50`, its SSH keys, a GitHub key, xrdp and the course repo. A full clone would copy all of it, including a second copy of the same private keys and SSH host keys. Cleaning a clone is about as much work as a fresh install, and a fresh install is easier to trust.
- **Not the HackRange Local Lab (VM 206).** The Kali lab there is a Docker container inside VM 206 that is deleted when the lab stops. It is not a standalone VM.
- **A new ISO.** I had deleted my ISO files to save space, so I downloaded the current Kali installer again. I plan to delete it after the build.
- **Same specs as VM 201.** 4 cores, 4096 MB RAM and an 80 GB disk. The host has about 12 GB available, so ballooning stays on with a 2048 MB minimum.
- **`10.0.0.90`.** I picked it believing `10.0.0.85` was still Stalwart's address. Stalwart was later moved to `10.0.0.250` (see note 22 and `stalwart.md`), so `10.0.0.85` is free again. I kept `10.0.0.90`.
- **Key-only SSH and xrdp on loopback**, the same pattern as note 15.

## VM settings

| Setting | Value |
| --- | --- |
| VM ID / name | 207 / `kali-linux` |
| OS | Kali Linux 2026.3 (Installer ISO, x86_64), kernel `7.1.5+kali-amd64` |
| Machine / BIOS | Default (i440fx) / Default (SeaBIOS) |
| SCSI controller | VirtIO SCSI single |
| Disk | 80 GiB on `local-lvm`, SCSI bus, Discard, SSD emulation, IO thread, Backup on |
| CPU / memory | 4 cores (host CPU type), 4096 MiB, ballooning minimum 2048 MiB |
| Network | VirtIO on `vmbr0`, `10.0.0.90/24`, gateway `10.0.0.1`, DNS `10.0.0.30`, IPv6 disabled |
| Other | QEMU guest agent on, hostname `kali-linux`, domain `lab`, user `brie` |

## What I did

1. **Downloaded the ISO** in Proxmox: `local`, ISO Images, Download from URL. I used the Installer card on the x86_64 tab at kali.org, copied the link address and the SHA-256 sum, and set the hash algorithm to SHA-256 under Advanced. The file is `kali-linux-2026.3-installer-amd64.iso`, 4.70 GiB.
2. **Created VM 207** with the settings above. On the Disks tab the disk list on the left shows `scsi0`. That is just the list of the VM's disks, and only the size (default 32), Discard and SSD emulation needed changing. The confirm screen showed `balloon 0`, so I went back and set the memory minimum to 2048.
3. **Started the install.** The first boot landed in a text-mode installer (a numbered language list). I reset the VM and chose Graphical install.
4. **Installed Kali**: hostname `kali-linux`, domain `lab`, guided whole-disk, all files in one partition. The confirm screen listed only `sda` (ext4 and swap). Default software (Xfce and the top-10 tools). GRUB on `/dev/sda`.
5. **Set the CD/DVD drive (ide2) to Do not use any media** before the first reboot.
6. **Updated and installed the basics** from a terminal: `sudo apt update && sudo apt full-upgrade -y`, then `sudo apt install -y qemu-guest-agent openssh-server`. Both packages were already at the newest version.
7. **Set the static IP** under the network icon, Edit Connections (right-click the icon in the top panel): Manual, `10.0.0.90`, netmask `24`, gateway `10.0.0.1`, DNS `10.0.0.30`, IPv6 Disabled.
8. **Fixed DNS.** `gitea.lab` would not resolve at first. `nmcli -g IP4.DNS,IP4.ADDRESS,ipv4.dns,ipv4.method con show "Wired connection 1"` showed the saved profile was right (`manual`, `10.0.0.30`), but the live connection still had the old DHCP lease and the Xfinity DNS. A reset of the VM applied the saved settings and `gitea.lab` resolved. Toggling the connection down and up would probably have done the same, but I did not test that.
9. **Added the CoreDNS entry** on the CoreDNS LXC (2000), `10.0.0.90	kali-linux.lab` in `/etc/coredns/lab.hosts`, then `systemctl restart coredns.service`.
10. **Switched to SSH from Windows**, because the Proxmox console would not paste. First login used the Kali password: `ssh brie@kali-linux.lab`.
11. **Made a key and a config entry** on Windows. A new key `kali-linux` (ed25519, with a passphrase saved in Vaultwarden), the public key copied to `~/.ssh/authorized_keys`, and a `Host kali-linux kali-linux.lab` block added to the Windows SSH config.
12. **Locked down SSH** with a drop-in file, `/etc/ssh/sshd_config.d/10-keys-only.conf`. `sudo sshd -T | grep -Ei "passwordauth|permitroot"` shows both as `no`.
13. **Installed xrdp**, bound it to loopback (`port=tcp://127.0.0.1:3389` in `/etc/xrdp/xrdp.ini`) and confirmed with `ss -tln | grep 3389`.
14. **Added an Uptime Kuma monitor**, a ping monitor named `kali-linux Ping`. It will show down whenever the VM is off, which is expected because I do not leave this VM running.
15. **Connected over RDP** through a tunnel: `ssh -N -L 3390:localhost:3389 kali-linux`, then `mstsc` to `localhost:3390`, Session Xorg, user `brie` with the Kali account password.

## Problems I hit

| Problem | Cause | Fix |
| --- | --- | --- |
| First boot showed a numbered language list in a text installer | A different boot menu entry was selected | Reset the VM and chose Graphical install |
| Left-click on the network icon had no Edit Connections | That menu only lists connections | Right-click the icon, or run `nm-connection-editor` |
| `gitea.lab` would not resolve after saving the static settings | The live connection still used the DHCP lease and Xfinity DNS | Reset the VM so the saved settings applied |
| Proxmox console would not paste | The console has no clipboard | Worked over SSH from Windows |
| `Permission denied (publickey)` on `ssh kali-linux.lab` | The SSH config entry only matched `kali-linux` | Put both names on the Host line |
| `Could not resolve hostname kali-linux` | The multi-line paste never wrote the `kali-linux` block; the config had only the `kali-python` and `stalwart` entries | Wrote the block with a single-line `Add-Content` using `` `n `` for the line breaks |
| My first `nmcli` check returned nothing | Typed `-d` instead of `-g`, a capital C in the connection name, and a second command on the same line | `nmcli -g ... con show "Wired connection 1"` on its own line |

## Open items

- Add VM 207 to the local, cloud and USB backup jobs.
- Delete the Kali ISO from `local` storage once the VM is settled.
