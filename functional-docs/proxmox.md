# Functional Documentation — Proxmox VE (Host)

> **Functional documentation** is the short, orderly, repeatable version — no errors, no exploration, just the steps that worked, ready to copy-paste. Distilled from [`installation-notes/01-installing-proxmox.md`](../installation-notes/01-installing-proxmox.md) and [`installation-notes/02-security-hardening.md`](../installation-notes/02-security-hardening.md).

## At a glance

| | |
|---|---|
| **Host** | GMKtec NucBox G10 Pro (bare metal) |
| **IP** | `10.0.0.52` (DHCP reservation on router, MAC `84:47:09:88:ee:f6`) |
| **Web UI** | `https://10.0.0.52:8006` |
| **Web login** | `admin@pve` (Proxmox VE authentication server realm) + TOTP MFA — root web login disabled |
| **SSH** | `ssh root@10.0.0.52 -i ssh_key` — key-only, password auth disabled |

## Fresh install (reproducible steps)

1. Download the latest **ISO Installer** from the [Proxmox downloads page](https://www.proxmox.com/en/downloads).
2. Flash it to a USB drive with [Rufus](https://rufus.ie/en/#download) — let Rufus auto-select **DD Image mode** when it detects the ISOHybrid image.
3. **Before booting the installer, connect an ethernet cable to the target hardware.** This avoids the installer guessing a fictitious static IP.
4. Boot the target hardware from the USB (GMKtec NucBox: tap **Esc** at power-on).
5. Run **Install Proxmox VE (Graphical)**: accept EULA → select target disk (full wipe) → set location/timezone → set root password → confirm network settings pick up a real DHCP lease → Install.
6. Remove the USB drive before the post-install reboot.
7. Confirm you can reach `https://<assigned-ip>:8006` from another machine on the network.
8. Set a DHCP reservation on the router for the host's MAC address so its IP never changes.

## Post-install hardening (run immediately after first login)

Run the community PVE Post Install script from the host's web **Shell**:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

Choices for a single-node, personal-use lab: disable `pve-enterprise` + `ceph-enterprise` repos, add `pve-no-subscription` repo, disable the subscription nag, skip `pvetest`, disable HA + Corosync, update, reboot.

## Security hardening (run once, after post-install)

1. **Datacenter > Users > Add** — create `admin@pve` (realm: Proxmox VE authentication server), set password.
2. **Datacenter > Permissions > Add > User Permission** — Path `/`, User `admin@pve`, Role `Administrator`, Propagate checked.
3. **Datacenter > Two Factor > Add > TOTP** for `admin@pve` — scan QR with an authenticator app that supports encrypted backup/export.
4. **Datacenter > Users > root > Edit** — uncheck **Enabled** to disable root's web login.
5. Generate an SSH key pair, add the public key to `~/.ssh/authorized_keys` on the host, confirm key login works, then set `PasswordAuthentication no` in `/etc/ssh/sshd_config` and `systemctl restart sshd`.
6. Back up the private key to a password manager + offline drive. Never commit it to this repo.

## Day-to-day operations

### Accessing Proxmox

Browse to `https://10.0.0.52:8006` from a machine on the network. You'll hit a self-signed certificate warning (expected) — click **Advanced → Proceed**. Log in as `admin@pve` with MFA (Realm: **Proxmox VE authentication server**).

> If this IP stops responding, the DHCP reservation for the host's MAC (`84:47:09:88:ee:f6`) may not have taken, or something changed it. Check the router's DHCP client list for the host's current address.

### Shutting down / rebooting the host

**From the web UI (recommended):** click the node name (`proxmox`) in the sidebar → **Shutdown** or **Reboot** (top-right).

Proxmox gracefully stops running VMs/LXCs as part of a host shutdown/reboot. For anything doing active writes (databases, file transfers, etc.), shut those down manually first.

**From the console/shell**, if the web UI is unreachable:

```bash
shutdown -h now   # power off
reboot            # restart
```

### Starting the host back up

No software "start" button — Proxmox runs on bare metal, so power-on means physically pressing the power button on the mini PC. Give it a minute or two to boot, then the web UI should be reachable again.

> **Possible future improvement:** Wake-on-LAN (WoL) so the host can be powered on remotely instead of needing a physical button press. Not configured yet.

### Starting / stopping individual VMs and LXCs

Select the VM/LXC in the sidebar, then use the top-right buttons:

- **Start** — boots it up.
- **Shutdown** — graceful shutdown signal (like pressing the power button on a real machine). **Use this by default.**
- **Stop** — hard power-off, equivalent to pulling the plug. Only use if Shutdown hangs — risks data loss/filesystem corruption on anything mid-write.

**Auto-start on host boot:** each VM/LXC has a **"Start at boot"** toggle under its **Options** tab. Enable for anything that should come back up automatically when the host reboots.

### Recommended shutdown order (as the lab grows)

1. Gracefully **Shutdown** any running VMs/LXCs from the web UI first (or let the host handle it automatically, if nothing stateful is running).
2. **Shutdown** the Proxmox host itself.
3. To bring it all back: power on the mini PC → wait for boot → log into the web UI → anything with "Start at boot" enabled comes up automatically; start anything else manually.

## Running services on this host

See the [master App Table](../README.md#app-table) for the current list (Gitea, Vaultwarden, Uptime Kuma).

The original Ubuntu 24.04 LXC (CT 100, `10.0.0.218`) built in [`installation-notes/01-installing-proxmox.md`](../installation-notes/01-installing-proxmox.md) has since been deleted, along with the practice Ubuntu/Debian hosts from Chapter 2 — per the course's end-of-chapter cleanup.
