# 01 — Installing Proxmox VE on a GMKtec NucBox G10 Pro

## Overview

This is the build log for turning a GMKtec NucBox G10 Pro mini PC (previously running Windows 11) into a dedicated Proxmox VE hypervisor — the foundation of my home lab. Following the [Just Hacking Training](https://justhackingtraining.com) "Home Lab: Beginner Buildout" course.

**Hardware:**
- GMKtec NucBox G10 Pro mini PC (target/dedicated hardware — wiped Windows 11 entirely)
- A separate Windows machine, used to flash the USB installer and access the Proxmox web interface
- A 64GB USB drive (SanDisk)

**Software:**
- Proxmox VE 9.2 (`proxmox-ve_9.2-1.iso`)
- [Rufus](https://github.com/pbatard/rufus) 4.15 for flashing the installer USB

---

## Step 1: Flashing the installer with Rufus

Downloaded the Proxmox VE ISO installer from the [official downloads page](https://www.proxmox.com/en/downloads), then flashed it to the USB drive with Rufus.

Rufus detected the Proxmox ISO as an **ISOHybrid image** and automatically enforced **DD Image mode** rather than the default ISO/File-copy mode:

> "The image you have selected is an ISOHybrid, but its creators have not made it compatible with ISO/File copy mode. As a result, DD image writing mode will be enforced."

This is expected and correct — Proxmox's hybrid image needs a raw sector-for-sector write to boot properly. Partition scheme and target system fields grey out automatically in DD mode, since the image already has its own boot layout baked in (supports both BIOS and UEFI).

**Gotcha:** Rufus refused to start with an "Unsupported image location" error the first time, because the downloaded ISO was sitting on the same USB drive I was trying to flash — "the same thing as trying to saw the branch you're sitting on." Moved the ISO to the internal drive first and it worked immediately. Total write time: ~18 seconds.

## Step 2: Booting the installer on the mini PC

Plugged the freshly flashed USB into the GMKtec NucBox (not the Windows machine used to flash it — Proxmox needs to live on dedicated hardware). Boot menu key for GMKtec NucBox machines: **Esc**, tapped repeatedly right at power-on.

**Heads up for other GMKtec owners:** some GMKtec boards ship a Realtek RTL8125 2.5GbE NIC that Proxmox's default kernel doesn't always recognize out of the box, which can leave you with a working install but no network. Didn't end up needing this fix on this specific unit, but worth knowing about in advance.

## Step 3: Running the installer

Chose **Install Proxmox VE (Graphical)**. Walked through:

1. **EULA** — accepted.
2. **Target disk** — the mini PC's internal drive, `hdsize` at its full 953.0 GB. This is a full, irreversible wipe of the existing Windows 11 install — confirmed that was intentional, since the whole point is a dedicated Proxmox box.
3. **Location and timezone** — set accordingly.
4. **Password and email** — set root password; email is only used for Proxmox's internal notifications, not a login credential.
5. **Network configuration** — this ended up mattering a lot more than expected (see Step 6). At install time the mini PC had no ethernet cable connected (it had only ever used Wi-Fi under Windows), so the installer had no real DHCP lease to pull from. It pre-filled a plausible-looking but ultimately fictitious static config: `192.168.100.2/24`, gateway `192.168.100.1`. I didn't realize this was a guess rather than a real value until later.
6. **Summary** → Install.

**Gotcha:** Forgot to remove the USB drive before the post-install reboot, which just booted back into the installer menu. No harm done — the install itself was untouched. Powered off, physically removed the USB, powered back on, and it booted into the fresh Proxmox install correctly.

## Step 4: First login — console vs. web interface

On first boot, Proxmox shows a banner on the physical console (`https://192.168.100.2:8006/`) and a login prompt right there on the same screen. Logged in locally with `root` / the password set during install — this drops you into a plain Linux shell on the host itself (useful for diagnostics), which is different from the actual Proxmox **web interface** you use day-to-day for managing VMs/LXCs.

## Step 5: The networking rabbit hole

This was the most involved part of the whole install. Documenting it in full because the debugging process is probably the most useful part of this write-up for anyone else who hits it.

**Symptom:** Browser couldn't reach `https://192.168.100.2:8006` — `ERR_CONNECTION_TIMED_OUT`.

**Investigation:**
1. `ip a` on the console showed only the loopback interface (`lo`) — no ethernet interface at all initially. Turned out the mini PC's ethernet port had never actually had a cable plugged into it; it had only ever used Wi-Fi under Windows.
2. After plugging in an ethernet cable to the router, `ip a` showed the `nic0` interface (bridged into `vmbr0`) flip from `NO-CARRIER` to `UP,LOWER_UP` — physical link established.
3. Still couldn't ping the gateway (`192.168.100.1`) *or* the internet, from either the mini PC itself or from the Windows machine trying to reach `192.168.100.2`.
4. Root cause, found by running `ipconfig` on the Windows machine: the real home network is `10.0.0.0/24` (gateway `10.0.0.1`) — nothing on `192.168.100.x` exists at all. The `192.168.100.2/24` config from the installer was a placeholder guess made because no cable was connected at install time, not a real DHCP-issued value.

**Fix:** Edited `/etc/network/interfaces` on the Proxmox host:

```diff
 iface vmbr0 inet dhcp
     bridge-ports nic0
     bridge-stp off
     bridge-fd 0
```

(changed from `inet static` with the bogus `address`/`gateway` lines to `inet dhcp`), then:

```bash
systemctl restart networking
```

`vmbr0` immediately picked up a real, valid lease: `10.0.0.52/24`. Web interface was reachable at `https://10.0.0.52:8006` right after.

**Lesson learned:** if you're installing Proxmox with no ethernet cable connected yet, don't trust whatever network config the installer pre-fills — it may be a non-functional guess rather than a real DHCP lease. Confirm your actual router's subnet (`ipconfig` / `ip a` on a device that's definitely on the real network) before assuming the installer got it right.

Follow-up: set a DHCP reservation on the router for the mini PC's MAC address (`84:47:09:88:ee:f6`) so `10.0.0.52` stays fixed permanently — Proxmox hard-codes whatever IP it's given into its web interface, so a future DHCP change would lock me out again.

## Step 6: Post-install hardening

Logged into the web interface and got the expected **"No valid subscription"** popup — normal, not an error. Proxmox VE is fully free/functional without one; this just nags about the paid enterprise support tier.

Ran the community **PVE Post Install** helper script, run from the Proxmox host's **Shell** in the web UI:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

Choices made (single-node, no cluster, personal use):

| Prompt | Choice | Why |
|---|---|---|
| Disable `pve-enterprise` repo | Yes | No paid subscription |
| Disable `ceph-enterprise` repo | Yes | Same reason |
| Add `pve-no-subscription` repo | Yes | Free repo, needed for updates to work |
| Disable subscription nag | Yes | Removes the popup permanently |
| Add Ceph (no-sub) repo | Already present | N/A |
| Add `pvetest` repo | **No** | Beta/testing channel, not wanted |
| Disable High Availability | Yes | Single node, no cluster — reclaims resources |
| Disable Corosync | Yes | Same — cluster heartbeat service, unneeded solo |
| Update system now | Yes | |
| Reboot | Yes | |

After reboot, confirmed the subscription popup no longer appears.

## Step 7: First LXC — Ubuntu 24.04

Created the first container using the community Helper Scripts, again from the host's Shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/ubuntu.sh)"
```

Chose **Default Settings**: Ubuntu 24.04, 1 vCPU, 512MB RAM, 2GB disk, unprivileged container. The script validated storage, downloaded the template, created and started Container ID `100`, and confirmed network reachability automatically.

Result: `10.0.0.218` assigned via DHCP, IPv4 internet connectivity confirmed. (IPv6 showed as not connected — expected and fine, since the home network doesn't have IPv6 routed end-to-end.)

Logged into the container console — root login has **no password set by default** on these Helper Script containers, so set one immediately:

```bash
passwd
```

---

## Lessons learned / notes for next time

- **DD mode in Rufus is correct for Proxmox** — don't second-guess it if it gets auto-selected.
- **Wire up ethernet before running the installer**, or at minimum don't trust the network values it pre-fills if no cable was connected at install time.
- **Private/internal IPs aren't secret** — fine to document real ones like `10.0.0.x` here; the real things to keep out of this repo are credentials, API tokens, and SSH keys.
- **DHCP reservations matter** — Proxmox hard-codes its web interface to whatever IP it had during setup/last config change; a reservation on the router avoids getting locked out later.
- The **PVE Post Install script** should be one of the very first things run after a fresh install, before building any VMs/LXCs.

## What's next

Chapter 2 of the course: running actual self-hosted services as VMs/LXCs/Docker containers — gitea, vaultwarden, Uptime Kuma, Kali Linux, CoreDNS, and Tailscale.
