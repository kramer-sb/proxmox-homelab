# Functional Documentation - kali-linux (Specializing in Linux course VM)

> The clean, repeatable version. Distilled from [`installation-notes/23-kali-linux-vm.md`](../installation-notes/23-kali-linux-vm.md). Same pattern as [kali-python.md](kali-python.md).

## At a glance

| | |
|---|---|
| **Purpose** | Kali VM for HackRange's *Specializing in Linux* course (students install their own Kali) |
| **VM ID** | `207` |
| **IP / DNS name** | `10.0.0.90` / `kali-linux.lab` |
| **OS** | Kali Linux 2026.3 |
| **Specs** | 4 cores (`host`), 4096 MB RAM (balloon min 2048), 80 GB on `local-lvm` |
| **Access** | `ssh kali-linux` from Windows (key-only), RDP through an SSH tunnel, console via Proxmox |

## Rebuild from scratch

### 1. ISO

Proxmox > `local` > ISO Images > **Download from URL**. On kali.org, Get Kali > Installer Images > **x86_64** > **Installer** card: right-click the download arrow, Copy link address, and click **sum** for the SHA-256. In Advanced, set the hash algorithm to SHA-256 and paste the sum.

### 2. Create the VM

- General: ID `207`, name `kali-linux`
- OS: the Kali ISO, Linux, `7.x - 2.6 Kernel`
- System: defaults (SeaBIOS, i440fx, VirtIO SCSI single), **Qemu Agent** on
- Disks: Bus **SCSI**, `local-lvm`, 80 GB, Discard + SSD emulation + IO thread
- CPU: 4, type `host`
- Memory: 4096, Advanced: minimum 2048, ballooning on
- Network: `vmbr0`, VirtIO
- Tick **Start after created**

### 3. Install Kali

At the boot menu pick **Graphical install** (the other entries open a text-mode or accessibility installer). DHCP for now, hostname `kali-linux`, domain `lab`, guided whole-disk with all files in one partition (only `sda` should be listed), default software, GRUB to `/dev/sda`. Before the first boot, set the CD/DVD drive to **Do not use any media**.

### 4. Update and services

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent openssh-server
sudo systemctl enable --now ssh
```

### 5. Static IP and DNS

Right-click the network icon in the top panel > Edit Connections > Wired connection 1:

- IPv4: Manual, `10.0.0.90/24`, gateway `10.0.0.1`, DNS `10.0.0.30`
- IPv6: Disabled

Save, then restart the VM or toggle the connection so the settings apply. A saved profile does not change a connection that is already up. Verify: `ping 8.8.8.8`, `ping google.com`, `ping gitea.lab`.

### 6. CoreDNS

On the CoreDNS LXC (2000, not the Proxmox node shell), add to `/etc/coredns/lab.hosts`:

```
10.0.0.90	kali-linux.lab
```

`systemctl restart coredns.service`, then from Windows: `ipconfig /flushdns` and `ping kali-linux.lab`.

### 7. SSH from Windows (PowerShell)

Run each command on its own. The first two ask for a passphrase and a password.

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\kali-linux -C "windows to kali-linux"
type $env:USERPROFILE\.ssh\kali-linux.pub | ssh brie@kali-linux.lab "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
Add-Content -Encoding ascii $env:USERPROFILE\.ssh\config "`nHost kali-linux kali-linux.lab`n    HostName kali-linux.lab`n    User brie`n    IdentityFile ~/.ssh/kali-linux"
ssh kali-linux
```

The single-line `Add-Content` avoids a multi-line paste that did not write the block. Then on Kali:

```bash
sudo tee /etc/ssh/sshd_config.d/10-keys-only.conf > /dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
sudo sshd -t && sudo systemctl restart ssh
sudo sshd -T | grep -Ei "passwordauth|permitroot"   # both "no"
```

Confirm a second `ssh kali-linux` still works before closing the first. Back up the private key and passphrase to Vaultwarden. Optional, so the passphrase is entered once: `ssh-add $env:USERPROFILE\.ssh\kali-linux` in PowerShell.

### 8. Desktop over RDP (xrdp)

xrdp listens on loopback only and is reached through an SSH tunnel, so port 3389 is never open on the network. Log out of the Xfce desktop on the Proxmox console first. On Kali:

```bash
sudo apt install -y xrdp
echo xfce4-session > ~/.xsession
sudo adduser xrdp ssl-cert
sudo systemctl enable --now xrdp
sudo sed -i 's|^port=3389|port=tcp://127.0.0.1:3389|' /etc/xrdp/xrdp.ini
grep '^port=' /etc/xrdp/xrdp.ini      # first line should be port=tcp://127.0.0.1:3389
sudo systemctl restart xrdp
ss -tln | grep 3389                    # should show 127.0.0.1:3389, not 0.0.0.0
```

The other `port=` lines in `xrdp.ini` belong to the session sections. Leave them alone.

## Day-to-day

- Start the VM from Proxmox if it is off. Do not log in on the Proxmox console.
- Open the tunnel from Windows and leave the window open (it prints nothing when it works):
  ```
  ssh -N -L 3390:localhost:3389 kali-linux
  ```
- Run `mstsc` and connect to `localhost:3390`. Leave Session on **Xorg** and log in as `brie` with the Kali account password. That is separate from the SSH key passphrase.
- From `kali-python.md`, and not hit on this VM: if the same user is logged in on the Proxmox console, xrdp gives a black screen. `loginctl list-sessions` shows it.
- Closing the RDP window only disconnects. Use Log Out in Xfce to end the session.
- Away from home, Tailscale has to be connected first so `kali-linux.lab` resolves.

## RustScan

RustScan is a fast port scanner that hands the open ports it finds to nmap. Details and the mistakes I made are in [`installation-notes/23a-rustscan-addendum.md`](../installation-notes/23a-rustscan-addendum.md).

Install and check:

```bash
sudo apt update
sudo apt install rustscan -y
rustscan --version
```

Scan a lab machine with version detection:

```bash
rustscan -a 10.0.0.10 --ulimit 5000 -- -sV
```

Everything before the `--` is a RustScan option. Everything after it goes to nmap, and the nmap flag is `-sV` with a capital V. Keep `--ulimit 5000` before the `--`, or the file limit warning appears. If `apt` cannot find the package, run `sudo apt full-upgrade` and try again. Only scan machines in the lab.

## Lab integration

- [ ] Add VM 207 to the local backup job (daily, keep 7)
- [ ] Add VM 207 to the PBS cloud backup job and the USB backup job
- [x] Uptime Kuma: ping monitor `kali-linux Ping`. It shows down whenever the VM is off, by design, since this VM is only started when needed
- [x] CoreDNS entry for `kali-linux.lab`
- [ ] Start at boot: decide (Options tab). Probably off, since it is a workstation.
