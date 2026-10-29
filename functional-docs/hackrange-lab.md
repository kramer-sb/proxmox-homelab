# Functional Documentation - HackRange Local Lab (hackrange-lab)

> The clean, repeatable version. Reference for VM 206, which runs the HackRange Mentorship labs locally instead of the hosted browser labs.

## At a glance

| | |
|---|---|
| **Purpose** | Runs the seven HackRange Mentorship labs (Docker, one image per lab) on my own Proxmox host |
| **VM ID** | `206` |
| **IP / DNS name** | `10.0.0.80` (static, set inside Ubuntu) / none, accessed by IP |
| **OS** | Ubuntu 24.04.5.1 Desktop (amd64) |
| **Specs** | 4 cores (`host`), 8192 MiB RAM (ballooning off), 100 GiB on `local-lvm` (backup off) |
| **Lab control panel** | `https://10.0.0.80:8456/control/` (self-signed certificate warning is expected), login `student` |
| **Shell and desktop** | `ssh brie@10.0.0.80`, `mstsc` to `10.0.0.80` as `brie` (Ubuntu Desktop Sharing), console via Proxmox |
| **Lab's own ports** | 2223 (lab SSH), 13489 (lab RDP) |
| **Lab source** | https://github.com/hackrange/lab_hackrangementorship, installed under `/opt/hackrange-mentorship` |

## Rebuild from scratch

### 1. ISO

Proxmox > `local` > ISO Images > **Download from URL**. Use `https://releases.ubuntu.com/24.04/ubuntu-24.04.5.1-desktop-amd64.iso` (about 5.8 GiB). Use the Desktop ISO, not Server. Stay on an LTS release the course names (24.04 or 26.04), not an interim release.

### 2. Free up RAM

The VM needs 8 GiB. Shut down Security Onion, Wazuh and the Windows VMs first. Proxmox Backup Server, CoreDNS and ts-router can stay on.

### 3. Create the VM

- General: ID `206`, name `hackrange-lab` (check that the ID is free)
- OS: the Ubuntu ISO, Linux, `6.x - 2.6 Kernel`
- System: machine `q35`, SCSI controller VirtIO SCSI single, **Qemu Agent** on
- Disks: Bus **SCSI**, `local-lvm`, 100 GiB (minimum 60), Discard + SSD emulation on, **Backup off**
- CPU: 4 cores, type `host`
- Memory: 8192, ballooning off
- Network: `vmbr0`, VirtIO

### 4. Install Ubuntu

Install Ubuntu, Interactive installation, Default selection of apps, Erase disk and install Ubuntu. Computer name `hackrange-lab`, leave **Use Active Directory** unticked. Afterward, set the CD/DVD drive to **Do not use any media** and reboot.

### 5. Update and services

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y qemu-guest-agent curl openssh-server
```

Proxmox showing the VM's IP on the Summary tab confirms the guest agent works. A "no installation config" message from `systemctl enable` on the agent is normal.

### 6. Static IP

Find the connection name, then set the address, gateway and DNS:

```bash
nmcli con show
sudo nmcli con mod netplan-enp6s18 ipv4.method manual ipv4.addresses 10.0.0.80/24 ipv4.gateway 10.0.0.1 ipv4.dns 10.0.0.30
sudo nmcli con up netplan-enp6s18
```

`netplan-enp6s18` is the name on this VM. Use whatever `nmcli con show` lists. Verify with `ip -4 addr show enp6s18` and `ping -4 -c 2 google.com`, then reboot once and confirm the address held.

### 7. Remote access

SSH from the Windows host (PowerShell), which also makes pasting work:

```powershell
ssh brie@10.0.0.80
```

RDP to the Ubuntu desktop: Settings > System > Remote Desktop > **Desktop Sharing** on, then set the RDP username and password under Login Details (separate from the Ubuntu login). In `mstsc`, put `10.0.0.80` in Computer and `brie` in User name (not `brie@10.0.0.80`). If Windows offers the wrong account, choose **More choices > Use a different account**. Set the resolution under Settings > Displays so the window fills the screen. Desktop Sharing only works while someone is logged in on the Proxmox console.

### 8. Snapshot before the lab install

Proxmox > VM 206 > Snapshots > **Take Snapshot**, name `fresh-ubuntu`. Description: Clean Ubuntu 24.04.5.1 Desktop install, fully updated. Static IP 10.0.0.80, qemu-guest-agent and openssh-server installed, RDP Desktop Sharing enabled. Taken before the HackRange lab install.

### 9. Install the lab

```bash
curl -fsSL https://raw.githubusercontent.com/hackrange/lab_hackrangementorship/main/install.sh | sudo bash
```

It took 24 minutes here. Rerun the same command if it stops partway. Then check it, and log out and back in (or reboot) so the lab commands work without `sudo`:

```bash
hackrange-lab status
```

Every line should show `[ok]` and `no lab running` is expected. Take a second snapshot, `labs-installed`.

### 10. Open the panel

From the VM, open `https://localhost:8456` in Firefox. From the Windows host, open `https://10.0.0.80:8456/control/`. Use `https://`, because plain `http://` returns a 400 error. Accept the certificate warning, and sign in as `student`. The lab password comes from:

```bash
/opt/hackrange-mentorship/show_information.sh
```

Save the `student` password and the RDP password in Vaultwarden. Never put them in this repo, a snapshot description or a chat.

## Day-to-day

```bash
hackrange-lab list
hackrange-lab start kali
hackrange-lab stop
hackrange-lab update
hackrange-lab diag
```

- Only one lab runs at a time. Stopping a lab deletes everything inside it, so copy out anything worth keeping first.
- When done: `hackrange-lab stop`, shut down VM 206, then start the other VMs again.
- `hackrange-lab` may ask for the Ubuntu password (sudo) until the session picks up the lab's group.
- The lab only answers computers on the private network. Do not forward its ports on the router.
- The VM sits on the home LAN, so only scan the lab's own machines and check each target before a scan.
- If Weeks 8 to 10 (Splunk) run slowly, raise the VM to 12 GiB with the other VMs off.

## Lab integration

- Backups: not included in the local, cloud or USB jobs, on purpose. The disk is created with Backup off because the VM is rebuildable and `local` storage is small. The two snapshots are the restore points, and they live on the same `local-lvm` pool, so they are not a backup. See [proxmox-backup-server.md](proxmox-backup-server.md).
- Monitoring: none yet. The VM is shut down between sessions, so a monitor would show it down most of the time.
- DNS: no `.lab` entry. Accessed by IP, like Security Onion and Wazuh.
