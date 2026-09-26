# 15 - Kali Python VM (kali-python)

Not part of the home lab course. This is a second Kali VM built as the coding machine for JHT's *Coding for Cybersecurity: Python Fundamentals* course. The original Kali VM (`10.0.0.25`) stays as-is; this one gets more disk so it can grow as the course adds packages and projects.

## Goal

A dedicated Linux box for the Python course that I can code on from VS Code on Windows, with the course repo living on the VM and pushing to GitHub. It should follow the same lab conventions as everything else: static IP, `.lab` name, backups, monitoring.

## Summary

| Item | Value |
|---|---|
| OS | Kali Linux 2026.2 (Installer ISO, x86_64) |
| VM ID | `201` (VMs live in the 200s) |
| Name / hostname | `kali-python` |
| IP | `10.0.0.50` (static) |
| DNS server | `10.0.0.30` (CoreDNS) |
| DNS name | `kali-python.lab` |
| CPU | 4 cores, type `host` |
| RAM | 4096 MB, ballooning minimum 2048 MB |
| Disk | 80 GB on `local-lvm` (SCSI, Discard + SSD emulation + IO thread) |
| Network | `vmbr0`, VirtIO |
| Python | 3.14.7 (shipped with Kali 2026.2) |
| Access | SSH key-only from Windows; VS Code Remote-SSH |
| Repo on VM | `~/python-fun-for-cybersecurity` |
| Login | Stored in Vaultwarden |

### Sizing check before building

Host at build time: 8 CPU threads, 12.6 GiB RAM (about 2.8 GiB in use), and `local-lvm` at 37 GB used of 875 GB. So 4 cores, 4 GB RAM and 80 GB of thin-provisioned disk were all comfortable. The 94 GB "HD space" on the node Summary page is only the Proxmox system partition; VM disks live on `local-lvm`, which is the rest of the drive.

## Step 1: Get the ISO onto Proxmox

Downloaded straight to Proxmox instead of uploading from Windows.

1. kali.org > Get Kali > Installer Images > **x86_64** tab > **Installer** card. Right-click the download arrow (next to "4.4G") > Copy link address. Clicked **sum** to get the SHA256.
2. Proxmox > `local (proxmox)` > **ISO Images** > **Download from URL**. Pasted the URL, **Query URL**, then under Advanced set Hash algorithm to SHA-256 and pasted the checksum.
3. Downloaded `kali-linux-2026.2-installer-amd64.iso`, got `TASK OK`.

Skipped the torrent link; Proxmox can't use it.

## Step 2: Create the VM

**Create VM** with these settings:

| Tab | Setting |
|---|---|
| General | VM ID `201`, Name `kali-python` |
| OS | ISO `kali-linux-2026.2-installer-amd64.iso`, Type Linux, Version `7.x - 2.6 Kernel` |
| System | Defaults (SeaBIOS, i440fx, VirtIO SCSI single), **Qemu Agent** checked |
| Disks | Bus **SCSI** 0, `local-lvm`, 80 GB, Discard, SSD emulation, IO thread |
| CPU | 4 cores, Type `host` |
| Memory | 4096, Advanced: minimum 2048, Ballooning on |
| Network | `vmbr0`, VirtIO (firewall=1 is the default and does nothing unless the Proxmox firewall is on) |
| Confirm | "Start after created" unchecked |

Gotchas:

- **OS version dropdown only offers "7.x - 2.6 Kernel".** That's the current label for modern Linux ("2.6 and newer"), not a mismatch. Correct for Kali.
- **SSD emulation was greyed out.** The disk bus had been left on **VirtIO Block**, which doesn't support that option. Switching Bus/Device to **SCSI** unlocked it. VirtIO SCSI is the *controller* (System tab); SCSI is the *disk bus* (Disks tab). Easy to mix up.

## Step 3: Install Kali

Start > Console > **Graphical install**.

- Network: DHCP during install (static IP set afterward, same as the first Kali VM).
- Hostname `kali-python`, domain `lab`.
- Partitioning: Guided, use entire disk, all files in one partition. The confirm screen lists `sda` getting an ext4 partition and a swap partition. Safe: `sda` is the new 80 GB virtual disk, the only disk this VM can see.
- Software: defaults (Xfce + top-10 tools).
- GRUB to `/dev/sda`.

After first boot: Proxmox > VM 201 > **Hardware** > CD/DVD Drive (ide2) > **Do not use any media**, so it never boots the installer again.

## Step 4: Update and base services

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent openssh-server
sudo systemctl start qemu-guest-agent
sudo systemctl enable --now ssh
python3 --version
```

- During the upgrade, a **"Restart services during package upgrades without asking?"** prompt (from configuring `libc6`). Chose **Yes**. Fine for a workstation VM with nothing critical running, and it stops the question repeating.
- `qemu-guest-agent` and `openssh-server` were already installed by the Kali installer; SSH just needed enabling.
- Python: **3.14.7**, which matches the course's recommended version. No extra Python install needed.
- apt listed a batch of packages as "no longer required" (mostly the old Python 3.13). Cleanup with `sudo apt autoremove -y` is optional.
- "Not Upgrading: 1" is a package Kali holds back on purpose. Ignored.

Side fix: the Xfce screen was locking after a few minutes. Changed in Settings > **Screensaver** (idle time, Lock Screen tab) and Settings > **Power Manager** (Display and Security tabs).

## Step 5: Static IP and DNS

Checked `10.0.0.50` was free first (`ping 10.0.0.50` from Windows, no replies).

GUI, same method as the first Kali VM: network icon > **Edit Connections** > Wired connection 1 > **IPv4 Settings**:

- Method: Manual
- Address `10.0.0.50`, netmask `24`, gateway `10.0.0.1`
- DNS servers: `10.0.0.30`

**IPv6 Settings** > Method: **Disabled**, so the gateway can't hand out an IPv6 DNS server that bypasses CoreDNS (the same problem the Windows host had, see `07a-dns-scope-addendum.md`).

Saved, toggled the connection off and on.

Verified:

```bash
ping -c 3 8.8.8.8      # replies: IP + gateway OK
ping -c 3 google.com   # replies: upstream DNS forwarding OK
ping -c 3 gitea.lab    # resolved to 10.0.0.10: CoreDNS in use
```

## Step 6: CoreDNS entry

On the CoreDNS LXC (2000), added to `/etc/coredns/lab.hosts`:

```
10.0.0.50	kali-python.lab
```

```bash
systemctl restart coredns.service
```

Gotcha: my first attempt at `nano /etc/coredns/lab.hosts` said **"Directory '/etc/coredns' does not exist"**. I was in the **Proxmox node** shell, not the CoreDNS container. Exited without saving, then opened the CoreDNS console (or `pct enter 2000` from the node shell). The path in `coredns.md` was correct.

Verified from Windows:

```
ipconfig /flushdns
ping kali-python.lab
```

Resolved to `10.0.0.50`, 4/4 replies.

## Step 7: Key-only SSH from Windows

All in PowerShell on Windows unless noted.

1. Dedicated key for this VM:

   ```powershell
   ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\kali-python -C "windows to kali-python"
   ```

   Passphrase set and stored in Vaultwarden, along with the private key.

2. Copied the public key over (Windows has no `ssh-copy-id`):

   ```powershell
   type $env:USERPROFILE\.ssh\kali-python.pub | ssh brie@kali-python.lab "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   ```

3. Added a host entry to `~/.ssh/config`.

   Gotcha: **Notepad kept saving the file as `config.txt`**, even with "All files" selected. Fixed by writing it from PowerShell instead:

   ```powershell
   @"

   Host kali-python
       HostName kali-python.lab
       User brie
       IdentityFile ~/.ssh/kali-python
   "@ | Add-Content -Encoding ascii $env:USERPROFILE\.ssh\config
   ```

   `-Encoding ascii` avoids invisible characters at the start of the file that SSH can choke on. (The Notepad workaround is typing the filename in quotes: `"config"`.)

4. `ssh kali-python` asked for the key passphrase, not the Kali password. Key works.

5. Disabled password and root logins with a drop-in file on Kali (drop-ins survive package updates; editing `sshd_config` directly might not).

   Gotcha: the first try with `sudo nano` **never saved the file**, so `sshd -T` still showed `passwordauthentication yes`. Checked with `ls /etc/ssh/sshd_config.d/` (file missing) and confirmed `sshd_config` has `Include /etc/ssh/sshd_config.d/*.conf`. Rewrote it with `tee`:

   ```bash
   sudo tee /etc/ssh/sshd_config.d/10-keys-only.conf > /dev/null <<'EOF'
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin no
   EOF
   sudo sshd -t && sudo systemctl restart ssh
   sudo sshd -T | grep -Ei "passwordauth|permitroot"
   ```

   Now shows `permitrootlogin no` and `passwordauthentication no`. (Nano reminder: Ctrl+O, **Enter**, Ctrl+X. The Enter is the step that gets missed.)

   Also gotcha: I ran the Windows `Set-Service` commands inside the Kali SSH session by mistake ("command not found"). A `PS` prompt is Windows; a `brie@kali-python` prompt is Kali.

6. Windows ssh-agent, so VS Code doesn't ask for the passphrase on every reconnect (admin PowerShell):

   ```powershell
   Set-Service ssh-agent -StartupType Automatic
   Start-Service ssh-agent
   ssh-add $env:USERPROFILE\.ssh\kali-python
   ```

   After this, `ssh kali-python` logs straight in.

Note: the login banner says **"Last login ... from 10.0.0.35"**. That's `ts-router`. Traffic from the Windows host to the lab goes through the Tailscale subnet router, which is also why pings from Windows show TTL=63 instead of 64. Expected in this lab, not a problem.

## Step 8: VS Code Remote-SSH

1. Installed the **Remote - SSH** extension (Microsoft) on Windows.
2. Ctrl+Shift+P > **Remote-SSH: Connect to Host** > `kali-python` (picked up from the SSH config). Platform: Linux.
3. Bottom-left shows **SSH: kali-python**; the integrated terminal prompt is `brie@kali-python`.
4. Installed the **Python** extension with "Install in SSH: kali-python", so Pylance, the debugger, etc. run on the VM.
5. File > Open Folder > `/home/brie/python-fun-for-cybersecurity/` (the folder picker is on Kali, not Windows).

## Step 9: Course repo on the VM

Git identity (GitHub noreply address, since the repo is public):

```bash
git config --global user.name "..."
git config --global user.email "<id>+kramer-sb@users.noreply.github.com"
git config --global init.defaultBranch main
```

Separate GitHub key for the VM, so it can be revoked without touching the Windows key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github -C "kali-python to github"
cat ~/.ssh/github.pub   # pasted whole line into GitHub > Settings > SSH and GPG keys
cat >> ~/.ssh/config <<'EOF'
Host github.com
    User git
    IdentityFile ~/.ssh/github
EOF
ssh -T git@github.com   # "Hi kramer-sb! You've successfully authenticated"
```

Created the empty public repo `python-fun-for-cybersecurity` on GitHub (no README/.gitignore/license, to avoid a conflict on first push), then on Kali:

```bash
sudo apt install -y gitleaks pre-commit    # apt, not pip: Kali blocks global pip installs
mkdir -p ~/python-fun-for-cybersecurity && cd ~/python-fun-for-cybersecurity
git init
# module folders with .gitkeep, notes.md, .gitignore, .pre-commit-config.yaml (same gitleaks config as this repo)
pre-commit install
git add . && git commit -m "Initial repo structure"
git remote add origin git@github.com:kramer-sb/python-fun-for-cybersecurity.git
git push -u origin main
```

Didn't see the gitleaks line during the first commit (scrolled past), so confirmed manually:

```bash
pre-commit run --all-files
# Detect hardcoded secrets.....Passed
```

Added `CLAUDE.md` and `README.md` in later commits. First `git add CLAUDE.md` failed with "pathspec did not match": the file hadn't been saved into the folder yet.

## Step 10: Lab integration

- [ ] Add VM 201 to the local backup job (daily, keep 7)
- [ ] Add VM 201 to the PBS cloud backup job (Backblaze B2, monthly, keep 3)
- [ ] Add VM 201 to the USB backup job
- [ ] Uptime Kuma: ping monitor for `kali-python.lab`
- [ ] Gitea: mirror `python-fun-for-cybersecurity` (see `functional-docs/github-mirroring.md`)
- [ ] Start at boot: decide (Options tab). Probably off, since it's a workstation and not a service.

## What's next

Course Module 0.3 Quick Test (`setup_test.py`), then Module 1.
