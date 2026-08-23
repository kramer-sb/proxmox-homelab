# Functional Documentation — Uptime Kuma (App Monitor)

> Distilled from [`installation-notes/05-installing-uptime-kuma.md`](../installation-notes/05-installing-uptime-kuma.md).

## At a glance

| | |
|---|---|
| **Host** | LXC on Proxmox (community script) |
| **IP** | `10.0.0.199` |
| **Web UI** | `http://10.0.0.199:3001` |

## Install (reproducible steps)

1. On the Proxmox Community Scripts site, find **Uptime Kuma (LXC)**, copy the install command, run it in the Proxmox host's Shell, choose **Advanced Install**, DHCP.
2. Browse to the printed URL (HTTP): select **SQLite** as the database type.
3. Create the admin account — generate credentials in Vaultwarden first.

## Standard monitors to add for every host

| Monitor Type | Friendly Name | Target |
|---|---|---|
| Ping | `<App> Ping` | app's IP |
| HTTP(S) | `<App> Web UI` (only if it has HTTPS) | app's URL — check **Ignore TLS/SSL errors** if self-signed |

Current monitors:

- Ping — Vaultwarden (`10.0.0.93`)
- Ping — Gitea (`10.0.0.15`)
- Ping — Proxmox (`10.0.0.52`)
- HTTP(S) — Proxmox Web UI (`https://10.0.0.52:8006`, TLS errors ignored)

No monitor needed for Uptime Kuma itself — reaching its own web page is proof it's up.

## Status pages

**Status Pages → + New Status Page**, name it, add relevant monitors, Save. One created so far for Proxmox (Ping + HTTPS monitors).

## Notes

- Add a monitor for every new app as it comes online — see the repo's [New App To-Do List](../README.md#new-app-to-do-list).
