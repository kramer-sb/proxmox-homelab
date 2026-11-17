# Functional Documentation - tcm-ubuntu (Ubuntu VM for the TCM Security courses)

> The clean, repeatable version. Distilled from [`installation-notes/24-tcm-ubuntu-vm.md`](../installation-notes/24-tcm-ubuntu-vm.md). Same pattern as [kali-linux.md](kali-linux.md).

## At a glance

| | |
|---|---|
| **Purpose** | Ubuntu VM for the TCM Security courses (SOC 101 redo), reused for later courses |
| **VM ID** | `208` |
| **IP / DNS name** | `10.0.0.95` / `tcm-ubuntu.lab` (once the CoreDNS entry in step 6 is added) |
| **OS** | Ubuntu 24.04 LTS (Noble), from the official cloud image |
| **Specs** | 4 cores (`host`), 8192 MiB RAM, 80 GiB on `local-lvm`, q35 |
| **Access** | `ssh tcm-ubuntu` from Windows (key-only), RDP through an SSH tunnel, console via Proxmox (xterm.js) |

## Rebuild from scratch

### 1. Run the script

Open community-scripts.org, search for the Ubuntu VM script (`scripts/ubuntu-vm`) and copy its install command. Paste it into the **Shell** tab of the Proxmox node in the web UI (not over SSH). Choose **Advanced**:

- Ubuntu **24.04**, VM ID `208`, machine **q35**, disk `80` GiB, disk cache **None**
- Hostname `tcm-ubuntu`, CPU model **host**, `4` cores, `8192` MiB RAM, `vmbr0`, the generated MAC
- Cloud-Init **yes**, username `brie`, **DHCP** (the static IP is set in step 3), DNS `10.0.0.30`
- When asked for SSH keys, tick any of them. They are replaced in step 3. The list can be cut off by the Proxmox menu: arrow keys and Space still work

Delete the credentials file the script prints, and keep the generated password in Vaultwarden:

```
rm /tmp/tcm-ubuntu-208-cloud-init-credentials.txt
```

### 2. Make a key (Windows PowerShell)

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\tcm-ubuntu"
type $env:USERPROFILE\.ssh\tcm-ubuntu.pub
```

Save the passphrase in Vaultwarden.

### 3. Static IP and key (Proxmox)

Shut down VM 208. On its **Cloud-Init** tab:

- IP Config: static `10.0.0.95/24`, gateway `10.0.0.1`
- DNS servers: `10.0.0.30`
- SSH public key: paste the line from `tcm-ubuntu.pub` (this replaces any imported keys)

Click **Regenerate Image**, then start the VM and give it a minute or two. Cloud images use the serial console, so use **xterm.js** (not noVNC) if you need the console.

### 4. SSH config (Windows PowerShell)

```powershell
Add-Content -Encoding ascii $env:USERPROFILE\.ssh\config "`nHost tcm-ubuntu`n    HostName 10.0.0.95`n    User brie`n    IdentityFile ~/.ssh/tcm-ubuntu"
ssh tcm-ubuntu
```

If SSH warns that the host key changed, run `ssh-keygen -R 10.0.0.95` and connect again. Verify inside the VM: `ip a`, `ping 8.8.8.8`, `ping gitea.lab`.

### 5. Set the account password

xrdp logs in with the account password, not the SSH key. Cloud images do not give you one you know, so set it:

```bash
sudo passwd brie
```

Save it in Vaultwarden.

### 6. CoreDNS

On the CoreDNS LXC (2000, not the Proxmox node shell), add to `/etc/coredns/lab.hosts`:

```
10.0.0.95	tcm-ubuntu.lab
```

`systemctl restart coredns.service`, then from Windows: `ipconfig /flushdns` and `ping tcm-ubuntu.lab`.

### 7. Desktop over RDP (xfce4 and xrdp)

xrdp listens on loopback only and is reached through an SSH tunnel, so port 3389 is never open on the network. On the VM, over SSH:

```bash
sudo apt update && sudo apt install -y xfce4 xfce4-goodies xrdp
echo xfce4-session > ~/.xsession
sudo adduser xrdp ssl-cert
sudo systemctl enable --now xrdp
sudo sed -i 's|^port=3389|port=tcp://127.0.0.1:3389|' /etc/xrdp/xrdp.ini
sudo systemctl restart xrdp
ss -tln | grep 3389                    # should show 127.0.0.1:3389, not 0.0.0.0
```

If the install asks for a display manager, pick `lightdm`.

### 8. Course files and bookmarks

```bash
git clone https://github.com/MalwareCube/SOC101.git ~/SOC101
sudo snap install firefox
```

Open Firefox in the RDP session, press `Ctrl+Shift+O`, choose Import and Backup, then Import Bookmarks from HTML, and select `~/SOC101/resources/bookmarks/soc_bookmarks.html`. Course file passwords come from the TCM Security Academy account, not the repo.

## Day-to-day

- Start the VM from Proxmox if it is off. Do not log in on the Proxmox console while you RDP, because it causes a black screen.
- Open the tunnel from Windows and leave the window open (it prints nothing when it works):
  ```
  ssh -N -L 3391:localhost:3389 tcm-ubuntu
  ```
- Run `mstsc` and connect to `localhost:3391`. Leave Session on **Xorg** and log in as `brie` with the account password. That is separate from the SSH key passphrase.
- Closing the RDP window only disconnects. Use Log Out in Xfce to end the session.
- Away from home, Tailscale has to be connected first so the `.lab` name resolves.
- Stay on 24.04. Ignore the `26.04` upgrade offer in the login banner. Reboot when the banner says `System restart required`, then reopen the tunnel.

## Backups

- [ ] VM 208 is in the local, cloud and USB backup jobs (not done yet)
