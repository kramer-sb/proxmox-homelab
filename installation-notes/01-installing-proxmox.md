# Installation Notes — Installing Proxmox VE on the GMKtec NucBox G10 Pro

> **Installation notes** capture the process as it actually happened — links, commands, screenshots, and the mistakes along the way.

## Overview

Turning a GMKtec NucBox G10 Pro mini PC (previously running Windows 11) into a dedicated Proxmox VE hypervisor — the foundation of the home lab. Following the Just Hacking Training "Home Lab: Beginner Buildout" course, Chapter 1.

**Hardware:**
- GMKtec NucBox G10 Pro mini PC (target/dedicated hardware — wiped Windows 11 entirely)
- A separate Windows machine, used to flash the USB installer and access the Proxmox web interface
- A 64GB USB drive (SanDisk)

**Software:**
- Proxmox VE 9.2 (`proxmox-ve_9.2-1.iso`)
- Rufus 4.15 for flashing the installer USB

---

## Step 1: Flashing the installer with Rufus

Downloaded the Proxmox VE ISO from the official downloads page, flashed it to the USB drive with Rufus.

Rufus detected the ISO as an **ISOHybrid image** and auto-enforced **DD Image mode** (expected/correct — Proxmox's hybrid image needs a raw sector-for-sector write to boot properly on both BIOS and UEFI).

**Gotcha:** Rufus refused to start with "Unsupported image location" the first time — the downloaded ISO was sitting on the same USB drive being flashed. Moved the ISO to the internal drive first; worked immediately. Total write time: ~18 seconds.

## Step 2: Booting the installer on the mini PC

Plugged the freshly flashed USB into the GMKtec NucBox (not the Windows flashing machine). Boot menu key for GMKtec NucBox: **Esc**, tapped repeatedly at power-on.

**Heads up:** some GMKtec boards ship a Realtek RTL8125 2.5GbE NIC that Proxmox's default kernel doesn't always recognize out of the box (working install, no network). Didn't need this fix on this unit, but worth knowing.

## Step 3: Running the installer

Chose **Install Proxmox VE (Graphical)**:

1. **EULA** — accepted.
2. **Target disk** — internal drive, full 953.0 GB, full irreversible wipe of Windows 11 (intentional — dedicated Proxmox box).
3. **Location and timezone** — set accordingly.
4. **Password and email** — root password set; email is only for Proxmox's internal notifications, not a login credential.
5. **Network configuration** — mattered a lot more than expected (see Step 5). No ethernet cable was connected at install time, so the installer pre-filled a plausible-looking but fictitious static config: `192.168.100.2/24`, gateway `192.168.100.1`. Not realized as a guess until later.
6. **Summary → Install.**

**Gotcha:** Forgot to remove the USB before the post-install reboot — booted back into the installer menu. No harm done. Powered off, removed USB, powered back on — booted into the fresh install correctly.

## Step 4: First login — console vs. web interface

On first boot, the physical console shows `https://192.168.100.2:8006/` and a login prompt. Logging in locally as `root` drops into a plain Linux shell on the host (useful for diagnostics) — different from the web interface used day-to-day for VMs/LXCs.

## Step 5: The networking rabbit hole

**Symptom:** Browser couldn't reach `https://192.168.100.2:8006` — `ERR_CONNECTION_TIMED_OUT`.

**Investigation:**
1. `ip a` on the console showed only loopback — no ethernet interface (cable had never been plugged in; the box had only ever used Wi-Fi under Windows).
2. After plugging in ethernet, `ip a` showed `nic0` (bridged into `vmbr0`) flip from `NO-CARRIER` to `UP,LOWER_UP`.
3. Still couldn't ping the gateway or internet from either machine.
4. Root cause (via `ipconfig` on the Windows machine): the real home network is `10.0.0.0/24` (gateway `10.0.0.1`) — `192.168.100.x` doesn't exist anywhere on this network. The installer's config was a placeholder guess, not a real DHCP lease.

**Fix:** Edited `/etc/network/interfaces` on the Proxmox host — changed `iface vmbr0 inet static` (with the bogus address/gateway) to `inet dhcp`, then:

```bash
systemctl restart networking
```

`vmbr0` picked up a real lease: `10.0.0.52/24`. Web interface reachable at `https://10.0.0.52:8006` immediately after.

**Lesson learned:** if installing with no ethernet cable connected, don't trust the installer's pre-filled network config — confirm the real subnet from a device already on the network first.

**Follow-up:** set a DHCP reservation on the router for the mini PC's MAC (`84:47:09:88:ee:f6`) so `10.0.0.52` stays fixed — Proxmox hard-codes its web interface to whatever IP it had at setup/last change, so a future DHCP change would lock access out again.

## Step 6: Post-install hardening (PVE Post Install script)

Logged in, got the expected **"No valid subscription"** popup (normal — Proxmox VE is fully functional without one).

Ran the community **PVE Post Install** helper script from the host's web Shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

Choices made (single-node, no cluster, personal use):

| Prompt | Choice | Why |
|---|---|---|
| Disable `pve-enterprise` repo | Yes | No paid subscription |
| Disable `ceph-enterprise` repo | Yes | Same reason |
| Add `pve-no-subscription` repo | Yes | Free repo, needed for updates |
| Disable subscription nag | Yes | Removes the popup permanently |
| Add Ceph (no-sub) repo | Already present | N/A |
| Add `pvetest` repo | No | Beta/testing channel, not wanted |
| Disable High Availability | Yes | Single node, no cluster |
| Disable Corosync | Yes | Cluster heartbeat, unneeded solo |
| Update system now | Yes | |
| Reboot | Yes | |

Confirmed after reboot: subscription popup no longer appears.

## Step 7: First LXC — Ubuntu 24.04

Created via the community Helper Scripts, from the host's Shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/ubuntu.sh)"
```

Chose **Default Settings**: Ubuntu 24.04, 1 vCPU, 512MB RAM, 2GB disk, unprivileged. Script validated storage, downloaded the template, created/started Container ID `100`, confirmed network reachability automatically.

Result: `10.0.0.218` assigned via DHCP, IPv4 confirmed. (IPv6 not connected — expected, home network doesn't route IPv6 end-to-end.)

No default password on Helper Script container root logins — set one immediately: `passwd`.

---

## Lessons learned / notes for next time

- DD mode in Rufus is correct for Proxmox — don't second-guess it.
- Wire up ethernet before running the installer, or don't trust its pre-filled network values if no cable was connected.
- Private/internal IPs (`10.0.0.x`) aren't secret — fine to document. Keep out credentials, API tokens, SSH keys.
- DHCP reservations matter — Proxmox hard-codes its web interface IP.
- Run the PVE Post Install script early, before building VMs/LXCs.

## What's next

Chapter 2 of the course: running self-hosted services as VMs/LXCs/Docker containers — gitea, vaultwarden, Uptime Kuma, Kali Linux, CoreDNS, and Tailscale.
