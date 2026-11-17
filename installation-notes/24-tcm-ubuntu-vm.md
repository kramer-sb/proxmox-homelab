# 24 - Ubuntu VM for TCM Security (tcm-ubuntu)

Status: **Working**. VM 208 (`tcm-ubuntu`) is built from the Ubuntu 24.04 cloud image at `10.0.0.95`, with key-only SSH from Windows and an Xfce desktop over RDP through an SSH tunnel. The course repository and Firefox bookmarks are set up. The CoreDNS entry, the Uptime Kuma monitor and the backup jobs are still open (see Open items). The clean, repeatable version is in [tcm-ubuntu.md](../functional-docs/tcm-ubuntu.md).

## Goal

I am redoing the TCM Security SOC 101 course so I can sit for a certification, and the course needs an Ubuntu VM. I want it in the Proxmox lab the same way as the other VMs: a static IP, a `.lab` name, key-only SSH from Windows and a desktop over RDP. I named it `tcm-ubuntu` because I plan to reuse it for 201 later.

## Decisions

- **The community-scripts Ubuntu VM script, not a manual ISO install.** The script lives at community-scripts.org (`scripts/ubuntu-vm`) and builds a VM from Ubuntu's official cloud image. It supports 22.04, 24.04 and 26.04. I chose **24.04 LTS (Noble)** because it is the safe, well documented release. 26.04 was the pre-selected default and is very new.
- **Advanced setup, Cloud-Init on.** The cloud image has no preset login, so Cloud-Init is how the user and the SSH key get in.
- **`10.0.0.95`.** Every `.5` step from `10.0.0.5` to `10.0.0.80` is taken, and so are `10.0.0.90` (kali-linux) and `10.0.0.250` (Stalwart). `10.0.0.95` is the next open step after the newest VM. `10.0.0.85` is also free.
- **Static IP set in the Proxmox Cloud-Init tab**, not inside the guest, because the script skips its static IP prompts when DHCP is chosen (see Problems).
- **A dedicated SSH key (`tcm-ubuntu`)**, the same pattern as the Kali VMs, instead of reusing the keys the script offered to import.
- **Machine type q35** (the script's default). The Kali VMs use i440fx. The HackRange VM (206) is q35 as well.
- **CPU type host, 4 cores, 8 GiB RAM.** I do not plan to run other VMs while this one is on. If Kali is also running, add its RAM to the 8 GiB before checking the host total, because Proxmox will not stop an overcommit.
- **CoreDNS (`10.0.0.30`) as the only DNS server**, the same as every other lab machine, because the Xfinity gateway cannot push a DNS override over DHCP.

## VM settings

| Setting | Value |
| --- | --- |
| VM ID / name | 208 / `tcm-ubuntu` |
| OS | Ubuntu 24.04 LTS (Noble) cloud image (`ubuntu-24.04-server-cloudimg-amd64.img`) |
| Machine | q35 (the script's default) |
| Disk | 80 GiB on `local-lvm`, cache None |
| CPU / memory | 4 cores (host CPU type), 8192 MiB |
| Network | VirtIO on `vmbr0`, VLAN default, MTU default, keyboard `en-us` |
| Address | `10.0.0.95/24`, gateway `10.0.0.1`, DNS `10.0.0.30` (set in the Cloud-Init tab) |
| Cloud-Init | On, user `brie`, SSH public key from the dedicated `tcm-ubuntu` key |
| Other | Verbose mode on. The script started the VM when it finished |

## What I did

1. **Opened the script** at community-scripts.org (`scripts/ubuntu-vm`) and copied the install command from the page. I pasted it into the **Shell** tab of the Proxmox node in the web UI, not over SSH.
2. **Chose Advanced setup** and answered the prompts: Ubuntu 24.04, VM ID 208, machine q35, disk 80 GiB, disk cache None, hostname `tcm-ubuntu`, CPU model host, 4 cores, 8192 MiB RAM, the generated MAC address, Verbose mode.
3. **Chose Cloud-Init yes**, a username (`brie`), DHCP for the network, and DNS `10.0.0.30`.
4. **Ticked the SSH keys** the script offered to import (the Proxmox host's root key and one Windows key). The key list was partly hidden (see Problems), so I selected them blind with Space.
5. **The script built and started the VM.** It printed a summary with the VM's DHCP address (`10.0.0.105`), the Cloud-Init user, a generated password and the path of a credentials file in `/tmp`. Cloud-Init needs about a minute on first boot before a login works.
6. **Deleted the credentials file** from the Proxmox host with `rm` (as the script warns) and kept the password only in Vaultwarden.
7. **Shut the VM down and set the static IP** in the Cloud-Init tab: IP Config, static `10.0.0.95/24`, gateway `10.0.0.1`. I checked that DNS was `10.0.0.30`, then clicked **Regenerate Image** and started the VM.
8. **Made a dedicated key** on Windows (ed25519, with a passphrase saved in Vaultwarden):
   `ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\tcm-ubuntu"`
9. **Replaced the imported keys** with the new public key: Cloud-Init tab, SSH public key, paste the single line from `tcm-ubuntu.pub`, Regenerate Image, start the VM.
10. **Cleared the old host key** (`ssh-keygen -R 10.0.0.95`) because the regenerated image gave the VM new SSH host keys, and connected with `ssh -i $env:USERPROFILE\.ssh\tcm-ubuntu brie@10.0.0.95`.
11. **Added a Windows SSH config entry** so `ssh tcm-ubuntu` works, with a Host block for `tcm-ubuntu` (HostName `10.0.0.95`, User `brie`, IdentityFile `~/.ssh/tcm-ubuntu`). I only found it missing later, when the tunnel command failed (see Problems).
12. **Installed a desktop and xrdp** over SSH: `sudo apt update && sudo apt install -y xfce4 xfce4-goodies xrdp`. The install finished with no services needing a restart.
13. **Set up xrdp** the same way as Kali: `echo xfce4-session > ~/.xsession`, `sudo adduser xrdp ssl-cert`, `sudo systemctl enable --now xrdp`, then bound it to loopback with `sudo sed -i 's|^port=3389|port=tcp://127.0.0.1:3389|' /etc/xrdp/xrdp.ini` and `sudo systemctl restart xrdp`. `ss -tln | grep 3389` shows `127.0.0.1:3389`.
14. **Set the account password** with `sudo passwd brie`. The first RDP attempt reached xrdp but failed at login (see Problems).
15. **Connected over RDP** through a tunnel: `ssh -N -L 3391:localhost:3389 tcm-ubuntu`, then `mstsc` to `localhost:3391`, Session Xorg, user `brie`. I used port 3391 so it does not collide with the Kali tunnel on 3390.
16. **Set up the course files.** `git clone https://github.com/MalwareCube/SOC101.git ~/SOC101`, `sudo snap install firefox`, then in Firefox `Ctrl+Shift+O`, Import and Backup, Import Bookmarks from HTML, and `~/SOC101/resources/bookmarks/soc_bookmarks.html`.

## Problems I hit

| Problem | Cause | Fix |
| --- | --- | --- |
| The SELECT SSH KEYS dialog was cut off, so I could not see the checkboxes | The dialog was wider than the visible shell area and the left edge sat behind the Proxmox menu | Selected with the arrow keys and Space without seeing the markers. The shell can also be popped out into its own window |
| The script never asked for a static IP | I chose DHCP at the Network Mode prompt, and the static IP prompts only appear when DHCP is declined | Let it build on DHCP (`10.0.0.105`), then set the static IP in the Cloud-Init tab and regenerated the image |
| The new password did not work at login | A password set in the Cloud-Init tab only applies after Regenerate Image and a full shutdown and start. Cloud images also turn SSH password login off | Moved to key login, which was the goal anyway |
| `Permission denied (publickey)` on `10.0.0.95` | The keys the script imported (the Proxmox host's root key and one Windows key) did not match any key I use from Windows (`kali-linux`, `kali-python`, `stalwart`) | Made a dedicated `tcm-ubuntu` key and put it in the Cloud-Init tab |
| `ssh 10.0.0.105` timed out | The VM had moved to its static address | Connected to `10.0.0.95` |
| Host key verification failed on `10.0.0.95` | Regenerating the image gave the VM new host keys, and Windows had saved the old fingerprint | `ssh-keygen -R 10.0.0.95`, then accepted the new fingerprint. I made the change myself, so this was expected |
| `ssh-keygen` showed only a blinking cursor at first | Not confirmed. A mangled paste is the likely cause | The passphrase prompt appeared on a later attempt |
| The first summary line said `Cloud-Init: no` | It is the summary printed before the Cloud-Init prompt is answered | No action. The later summary and the final output both show `Cloud-Init: yes` |
| `ssh -N -L ... tcm-ubuntu` said `Permission denied (publickey)` | The `Host tcm-ubuntu` block was missing from the Windows SSH config, so SSH offered only the default keys. The earlier logins had used `-i` with the key path | Added the block with a single-line `Add-Content`, then ran the tunnel from a normal (non-admin) window |
| RDP showed `login failed for user brie` (the log said `sesman connect ok`) | The tunnel and xrdp were fine. The account password I was typing was not the one set on the VM | Ran `sudo passwd brie` over SSH and logged in with the new password |

## Open items

- Reboot the VM. The login banner says `System restart required` (from the apt updates). Stay on 24.04 and ignore the 26.04 upgrade offer. Reopen the tunnel afterwards. xrdp is enabled, so it should start by itself.
- Confirm the static settings from inside the VM: `ip a`, `ping 8.8.8.8`, `ping gitea.lab`.
- Add `10.0.0.95	tcm-ubuntu.lab` to `/etc/coredns/lab.hosts` on the CoreDNS LXC (2000), restart `coredns.service`, then `ipconfig /flushdns` and `ping tcm-ubuntu.lab` from Windows.
- Add an Uptime Kuma ping monitor named `tcm-ubuntu Ping`.
- Add VM 208 to the local, cloud and USB backup jobs.
- Confirm the SSH key passphrase and the account password are saved in Vaultwarden.
- Take a snapshot now that the desktop and course files work (for example `fresh-ubuntu`).
