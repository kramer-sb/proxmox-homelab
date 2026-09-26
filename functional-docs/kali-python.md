# Functional Documentation - kali-python (Python course VM)

> The clean, repeatable version. Distilled from [`installation-notes/15-kali-python-vm.md`](../installation-notes/15-kali-python-vm.md).

## At a glance

| | |
|---|---|
| **Purpose** | Coding machine for JHT *Coding for Cybersecurity: Python Fundamentals* |
| **VM ID** | `201` |
| **IP / DNS name** | `10.0.0.50` / `kali-python.lab` |
| **OS** | Kali Linux 2026.2, Python 3.14.7 |
| **Specs** | 4 cores (`host`), 4096 MB RAM (balloon min 2048), 80 GB on `local-lvm` |
| **Access** | `ssh kali-python` from Windows (key-only), VS Code Remote-SSH |
| **Repo** | `~/python-fun-for-cybersecurity` -> GitHub, mirrored to Gitea |

## Rebuild from scratch

### 1. ISO

Proxmox > `local` > ISO Images > **Download from URL**. Use the kali.org **Installer** ISO link (x86_64) and its SHA-256 sum under Advanced.

### 2. Create the VM

- General: ID `201`, name `kali-python`
- OS: the Kali ISO, Linux, `7.x - 2.6 Kernel`
- System: defaults, **Qemu Agent** on
- Disks: Bus **SCSI** (not VirtIO Block, or SSD emulation is unavailable), `local-lvm`, 80 GB, Discard + SSD emulation + IO thread
- CPU: 4, type `host`
- Memory: 4096, minimum 2048
- Network: `vmbr0`, VirtIO

### 3. Install Kali

Graphical install, DHCP for now, hostname `kali-python`, domain `lab`, guided whole-disk partitioning, default software, GRUB to `/dev/sda`. Afterward, set the CD/DVD drive to **Do not use any media**.

### 4. Update and services

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent openssh-server
sudo systemctl enable --now ssh
```

Answer **Yes** to "Restart services during package upgrades without asking?"

### 5. Static IP and DNS

Edit Connections > Wired connection 1:

- IPv4: Manual, `10.0.0.50/24`, gateway `10.0.0.1`, DNS `10.0.0.30`
- IPv6: Disabled

Toggle the connection, then verify: `ping 8.8.8.8`, `ping google.com`, `ping gitea.lab`.

### 6. CoreDNS

On the CoreDNS LXC (2000, not the Proxmox node shell), add to `/etc/coredns/lab.hosts`:

```
10.0.0.50	kali-python.lab
```

`systemctl restart coredns.service`, then from Windows: `ipconfig /flushdns` and `ping kali-python.lab`.

### 7. SSH from Windows (PowerShell)

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\kali-python -C "windows to kali-python"
type $env:USERPROFILE\.ssh\kali-python.pub | ssh brie@kali-python.lab "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
@"

Host kali-python
    HostName kali-python.lab
    User brie
    IdentityFile ~/.ssh/kali-python
"@ | Add-Content -Encoding ascii $env:USERPROFILE\.ssh\config
```

Test with `ssh kali-python`, then on Kali:

```bash
sudo tee /etc/ssh/sshd_config.d/10-keys-only.conf > /dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
sudo sshd -t && sudo systemctl restart ssh
sudo sshd -T | grep -Ei "passwordauth|permitroot"   # both "no"
```

Confirm a second `ssh kali-python` still works before closing the first session.

Windows ssh-agent (admin PowerShell), so the passphrase is entered once:

```powershell
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\kali-python
```

Back up the private key and passphrase to Vaultwarden.

### 8. VS Code

Install **Remote - SSH** on Windows > **Remote-SSH: Connect to Host** > `kali-python` > Linux. Install the **Python** extension "in SSH: kali-python". Open folder `/home/brie/python-fun-for-cybersecurity/`.

### 9. GitHub access and repo

```bash
git config --global user.name "..."
git config --global user.email "<id>+kramer-sb@users.noreply.github.com"
git config --global init.defaultBranch main
ssh-keygen -t ed25519 -f ~/.ssh/github -C "kali-python to github"
printf 'Host github.com\n    User git\n    IdentityFile ~/.ssh/github\n' >> ~/.ssh/config
ssh -T git@github.com
```

Add `~/.ssh/github.pub` to GitHub (Settings > SSH and GPG keys). To restore the repo on a rebuilt VM:

```bash
sudo apt install -y gitleaks pre-commit
git clone git@github.com:kramer-sb/python-fun-for-cybersecurity.git ~/python-fun-for-cybersecurity
cd ~/python-fun-for-cybersecurity && pre-commit install
```

`pre-commit install` is per-clone. Check it with `pre-commit run --all-files`.

## Day-to-day

- Start the VM from Proxmox if it's off, then open VS Code > Remote-SSH > `kali-python`.
- Terminal in VS Code is Kali. A `PS` prompt is Windows; `brie@kali-python` is Kali.
- Python packages go in a per-project venv (`python3 -m venv .venv`). Never `--break-system-packages`.
- "Last login from 10.0.0.35" is normal: Windows reaches the lab through `ts-router`.

## Lab integration

- Backups: included in the local, cloud, and USB jobs (see [proxmox-backup-server.md](proxmox-backup-server.md)).
- Monitoring: Uptime Kuma ping monitor on `kali-python.lab`.
- Gitea: read-only mirror of `python-fun-for-cybersecurity` (see [github-mirroring.md](github-mirroring.md)).
