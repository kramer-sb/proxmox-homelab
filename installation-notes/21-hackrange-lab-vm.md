# 21 - HackRange Local Lab VM

Status: **Working**. The HackRange Mentorship Local Lab runs as VM 206 (`hackrange-lab`) at `10.0.0.80`. The installer finished in 24 minutes, `hackrange-lab status` showed every service and lab image as `[ok]`, and I signed in to the control panel from inside the VM. The clean, repeatable version is in [hackrange-lab.md](../functional-docs/hackrange-lab.md).

## Goal

Run the labs for the HackRange Mentorship on my own Proxmox host instead of the hosted browser labs. The course offers both. They run the same seven labs, so the Local Lab is a choice about where they run. It also keeps them running between sessions and gives me something to document.

## Decisions

- **A full VM.** The course builds the labs inside an Ubuntu VM. The installer sets up Docker and builds one container image per lab. No nested virtualization is needed.
- **Ubuntu 24.04 LTS, not 26.10.** The course names 24.04 and 26.04 LTS as tested. 26.10 is an interim release it does not name, and the installer pulls in Docker, so I did not risk it. I did not test 26.10.
- **Static IP set inside Ubuntu.** I could not get into the Xfinity gateway. The DHCP pool starts at `10.0.0.101` (see 06), so `10.0.0.80` is in my static range and was the next open increment after `10.0.0.75`.
- **RAM.** The host has about 12 GB available and the VM needs 8 GB, so Security Onion, Wazuh and the Windows VMs have to be off while it runs. PBS, CoreDNS and ts-router stay on.

## VM settings

| Setting | Value |
| --- | --- |
| VM ID / name | 206 / `hackrange-lab` (205 is `win11-client`) |
| OS | Ubuntu 24.04.5.1 Desktop (amd64) |
| Machine / BIOS | q35 / Default (SeaBIOS) |
| SCSI controller | VirtIO SCSI single |
| Disk | 100 GiB on `local-lvm`, Discard on, SSD emulation on, Backup off |
| CPU / memory | 4 cores (host CPU type), 8192 MiB, ballooning off |
| Network | VirtIO on `vmbr0`, `10.0.0.80/24`, gateway `10.0.0.1`, DNS `10.0.0.30` |
| Other | QEMU guest agent on, hostname `hackrange-lab`, user `brie` |

## What I did

1. **Checked the pool.** `local-lvm` had about 560 GiB allocated across the other VMs, so a 100 GiB thin disk fit. I watched actual usage on the Summary tab, not the allocation.
2. **Downloaded the ISO** in Proxmox (`local`, ISO Images, Download from URL) from `releases.ubuntu.com/24.04/`, file `ubuntu-24.04.5.1-desktop-amd64.iso`, 5.82 GiB.
3. **Shut down the other VMs** and created VM 206 with the settings above. Backup is off on the disk because the VM is rebuildable and `local` storage is small.
4. **Installed Ubuntu** from the console: Interactive installation, Default selection of apps, Erase disk. Computer name `hackrange-lab`, Active Directory left unticked. Then I set the CD/DVD drive to Do not use any media. The first boot of the live installer was slow but not stuck.
5. **Updated and installed the basics**: `sudo apt update && sudo apt -y upgrade`, then `sudo apt install -y qemu-guest-agent curl openssh-server`. The agent unit prints a "no installation config" message when enabled with `systemctl`. That is normal. Proxmox then showed the VM's IP on the Summary tab.
6. **Set the static IP** with `nmcli`. The connection name was `netplan-enp6s18`, from `nmcli con show`. I checked with `ip -4 addr show enp6s18` and `ping -4 -c 2 google.com`, rebooted once, and the address held.
7. **Switched to SSH from my PC** (`ssh brie@10.0.0.80` in PowerShell), because the Proxmox console cannot paste.
8. **Turned on RDP** for the Ubuntu desktop (Settings, System, Remote Desktop, Desktop Sharing) with RDP credentials separate from my Ubuntu login. Raised the VM's resolution under Settings, Displays so the `mstsc` window fills the screen.
9. **Took the first snapshot**, `fresh-ubuntu`, before the lab install.
10. **Installed the lab** with the course's `install.sh`: `curl -fsSL https://raw.githubusercontent.com/hackrange/lab_hackrangementorship/main/install.sh | sudo bash`. It ended with the connection details box. I did not copy the password anywhere in this repo.
11. **Checked it** with `hackrange-lab status`. Docker, the web desktop, the control panel and both Guacamole services showed `[ok]`, along with all seven lab images. `no lab running` was expected.
12. **Took the second snapshot**, `labs-installed`, once the lab checked out, so the build never has to be repeated.
13. **Opened the panel** at `https://localhost:8456` inside the VM, accepted the certificate warning, and signed in as `student`.

## Problems I hit

| Problem | Cause | Fix |
| --- | --- | --- |
| `Unable to locate package quemu-guest-agent` | Typo in the package name | The name is `qemu-guest-agent` |
| Console would not paste | The Proxmox console has no clipboard | Installed `openssh-server` and worked over SSH |
| `ssh bire@...` rejected | Typo in the username | `ssh brie@10.0.0.80` |
| `Test-NetConnection` not recognized | Ran it in cmd, not PowerShell | Run it in PowerShell |
| `mstsc`: "Invalid connection file (brie@10.0.0.80)" | SSH-style `user@host` in the Computer field | `10.0.0.80` in Computer, `brie` in User name |
| RDP "credentials did not work" | Windows offered my Microsoft account | More choices, Use a different account, user `brie` and the RDP password |
| RDP desktop small in the middle of the screen | Desktop Sharing uses the VM's existing resolution | Raised the resolution in Ubuntu Settings, Displays |
| `400 The plain HTTP request was sent to HTTPS port` | Browser used `http://localhost:8456` | Use `https://` |
| `hackrange-lab` asks for a sudo password | The installer's group change was not active in my session | Harmless. A full logout or reboot should stop it |

## What I saw after install

The lab panel answers only on the private network, as the course says. The VM's own ports are 8456 (panel and web desktop), 2223 (lab SSH) and 13489 (lab RDP). The lab's RDP port is separate from the Ubuntu RDP I enabled on 3389. When I finish a session, I run `hackrange-lab stop`, shut the VM down and start the other VMs again. Stopping a lab deletes everything inside it.

## Open items

- Load `https://10.0.0.80:8456/control/` from my Windows host once. So far I have only opened the panel from inside the VM.
- Update the README table row so its two link cells point to this note and to `hackrange-lab.md`.
- Save the lab and RDP passwords in Vaultwarden. Optionally run `hackrange-lab reset` to replace the lab password.
- If the Splunk lab (weeks 8 to 10) is slow, raise the VM to 12 GiB with the other VMs off.
