# 02 — Starting and Stopping Proxmox

Quick reference for powering the home lab up and down safely, from the Windows machine's browser.

## Accessing Proxmox

From the Windows machine, browse to:

```
https://10.0.0.52:8006
```

You'll hit a self-signed certificate warning (expected, not an error) — click **Advanced → Proceed**. Log in with `root` / password, Realm set to **Linux PAM standard authentication**.

> If this IP address stops working, it likely means the DHCP reservation for the mini PC's MAC address (`84:47:09:88:ee:f6`) either wasn't set on the router yet, or something changed it. Check the router's DHCP client list for the mini PC's current address if `10.0.0.52` ever stops responding.

## Shutting down / rebooting the Proxmox host itself

**From the web interface (recommended):**

1. Click the node name (`proxmox`) in the left sidebar.
2. In the top-right corner, use the **Shutdown** or **Reboot** buttons.

Proxmox will gracefully stop any running VMs/LXCs as part of a host shutdown or reboot. That said, for anything doing active writes (databases, file transfers, etc. — not a concern yet with just the base Ubuntu LXC, but will matter once services like gitea or vaultwarden are running), it's safer to manually shut those down first before shutting down the host — see below.

**From the console/shell**, if the web UI is unreachable:

```bash
shutdown -h now   # power off
reboot            # restart
```

## Starting the host back up

There's no software "start" button — Proxmox runs on bare metal, so powering it back on means physically pressing the power button on the mini PC itself.

After power-on, give it a minute or two to fully boot, then the web interface at `https://10.0.0.52:8006` should be reachable again from the browser as normal.

> **Possible future improvement:** set up Wake-on-LAN (WoL) so the mini PC can be powered on remotely over the network instead of needing a physical button press. Not configured yet — worth revisiting once the lab is being accessed remotely (Tailscale chapter of the course).

## Starting / stopping individual VMs and LXCs

Once logged into the web interface:

1. Select the VM or LXC in the left sidebar (e.g., container `100 (ubuntu)`).
2. Use the buttons in the top-right corner:
   - **Start** — boots it up.
   - **Shutdown** — sends a graceful shutdown signal (like pressing the power button on a real machine). **Use this by default.**
   - **Stop** — hard power-off, equivalent to pulling the plug. Only use this if Shutdown hangs or doesn't respond — it can risk data loss / filesystem corruption on anything mid-write.

### Auto-start on host boot

Each VM/LXC has a **"Start at boot"** toggle under its **Options** tab. Enable this for anything that should automatically come back up whenever the Proxmox host itself reboots or powers on, so services don't have to be started by hand every time.

## Recommended shutdown order (as the lab grows)

1. Gracefully **Shutdown** any running VMs/LXCs from the web UI first (or let the host handle it automatically, if not running anything stateful).
2. **Shutdown** the Proxmox host itself from the web UI (or console).
3. To bring it all back: physically power on the mini PC → wait for boot → log into the web UI → anything with "Start at boot" enabled comes up automatically; start anything else manually as needed.

## Current setup snapshot

- Host IP: `10.0.0.52` (DHCP reservation recommended/pending on router)
- Container `100` — Ubuntu 24.04 LXC
