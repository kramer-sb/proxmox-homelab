# Functional Documentation — Vaultwarden (Password Manager)

> Distilled from [`installation-notes/04-installing-vaultwarden-and-caddy.md`](../installation-notes/04-installing-vaultwarden-and-caddy.md).

## At a glance

| | |
|---|---|
| **Host** | LXC on Proxmox, Docker (community "Docker LXC" script) |
| **IP** | `10.0.0.93` |
| **Web UI** | `https://passwords.local` (self-signed cert via Caddy) |
| **Exposed ports** | 80, 443 (Caddy) — Vaultwarden itself is not exposed directly |
| **DNS** | Temporary hosts-file entry: `passwords.local` → `10.0.0.93` (real DNS comes in Chapter 3) |

## Install (reproducible steps)

1. On the Proxmox Community Scripts site, use the **Docker LXC** script (not the Vaultwarden all-in-one script — broken as of writing). Advanced Install, DHCP. Accept the default answers for the Portainer/socket prompts.
2. On the new LXC:
   ```bash
   mkdir vaultwarden && cd vaultwarden
   touch docker-compose.yml
   ```
3. Populate `docker-compose.yml`:
   ```yaml
   services:
     vaultwarden:
       image: vaultwarden/server:latest
       container_name: vaultwarden
       restart: unless-stopped
       environment:
         DOMAIN: "https://passwords.local"
       volumes:
         - ./vw-data/:/data/

     caddy:
       image: caddy:latest
       restart: unless-stopped
       ports:
         - "80:80"
         - "443:443"
         - "443:443/udp"
       volumes:
         - ./Caddyfile:/etc/caddy/Caddyfile
         - caddy_data:/data
         - caddy_config:/config

   volumes:
     caddy_data:
     caddy_config:
   ```
4. Create `Caddyfile` alongside it:
   ```
   passwords.local {
       tls internal
       reverse_proxy vaultwarden:80
   }
   ```
5. Bring the stack up:
   ```bash
   docker compose up
   docker ps        # confirm both containers are running
   ss -antpl        # confirm ports 80/443 are bound to docker-proxy
   ```
6. On the machine you'll browse from, add a hosts-file entry pointing `passwords.local` at the LXC's IP (`10.0.0.93`).
7. Browse to `https://passwords.local`, accept the self-signed cert warning, create the admin account.

## Populate initial entries

Per the course's "catch up" checklist, add these to Vaultwarden right away:
- Proxmox IP + root password
- Proxmox Web UI URL, username, password
- Gitea LXC root password + Gitea app credentials
- Vaultwarden LXC root password + Vaultwarden app credentials

## Notes

- Use a `.local` domain (not `.com`/similar) to avoid clashing with real public domains.
- Vaultwarden requires HTTPS — there is no working HTTP-only path.
- Optional: Bitwarden browser extension → Settings → **Self Hosted** → Server URL = `https://passwords.local`.
