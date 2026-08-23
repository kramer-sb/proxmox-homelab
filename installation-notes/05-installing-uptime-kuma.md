# Installation Notes — Installing Uptime Kuma (App Monitor)

Following JHT *Home Lab: Beginner Buildout*, Section 2.8. Last of the "Big 3" starter apps — straightforward after Vaultwarden.

**Result:** Uptime Kuma running at `10.0.0.199:3001`.

## Step 1: Install the LXC via community script

Proxmox Community Scripts site → **Uptime Kuma (LXC)** — no special notes on the install card beyond running the installer. Copied the command into the PVE shell, chose **Advanced Install**.

> Used Vaultwarden to generate and store the LXC OS credentials and the web app admin credentials created during this install, per the course's recommendation.

DHCP networking — assigned address: `10.0.0.199`. Script printed the configuration URL at completion — `10.0.0.199:3001`.

## Step 2: First-run configuration

Navigated to `http://10.0.0.199:3001` (HTTP). Selected **SQLite** as the database type (lightweight, appropriate for a home lab). Created the admin account (credentials generated/stored in Vaultwarden).

## Step 3: Add monitors

Baseline **Ping** monitors for the other core hosts:

| Monitor Type | Friendly Name | Hostname |
|---|---|---|
| Ping | Vaultwarden Ping | `10.0.0.93` |
| Ping | Gitea Ping | `10.0.0.15` |
| Ping | Proxmox Ping | `10.0.0.52` |

No ping monitor needed for Uptime Kuma itself — if its own web page loads, it's up.

Added an **HTTP(S)** monitor for Proxmox specifically, since a ping monitor alone wouldn't catch the web UI (or a crashed service) going down while the host stays up:

- **Monitor Type:** HTTP(S)
- **Friendly Name:** Proxmox Web UI HTTPS
- **URL:** `https://10.0.0.52:8006`
- **Advanced → Ignore TLS/SSL errors:** checked (needed since Proxmox's cert is self-signed)

## Step 4: Custom status page

Created a status page (**Status Pages → + New Status Page**) for Proxmox, added both Proxmox monitors (Ping + HTTPS) to it.

## Notes for next time

- Add ping and/or HTTP(S) monitors for every new app going forward — this is now step 6 in the repo's [New App To-Do List](../README.md#new-app-to-do-list).
- Consider a combined status page across all Big 3 apps once there's more to show.
