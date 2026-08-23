# Installation Notes — Installing Vaultwarden (Password Manager) + Caddy

Following JHT *Home Lab: Beginner Buildout*, Section 2.7. Second of the "Big 3" starter apps, and the most involved install of the three — Vaultwarden strictly requires HTTPS, and the all-in-one community script didn't work, so this went through Docker instead.

**Result:** Vaultwarden running at `10.0.0.93`, served over HTTPS via Caddy at `https://passwords.local`.

## Overview of steps

1. Install a Docker-enabled LXC via community script (not the Vaultwarden all-in-one script — broken at time of writing).
2. Run Vaultwarden as a Docker service via Docker Compose.
3. Run Caddy as a Docker service via Docker Compose to terminate HTTPS in front of Vaultwarden.
4. Add a temporary custom DNS entry (hosts file) for `passwords.local` pointing at the LXC — a stand-in until real DNS is set up in Chapter 3.
5. Configure Vaultwarden via its web UI.

## Step 1: Install a Docker LXC

Proxmox Community Scripts site → **Docker LXC** script (a plain LXC with the Docker engine installed — the general-purpose script to reach for anytime something needs to run in Docker on Proxmox, not just Vaultwarden).

Ran the script in the PVE shell, Advanced Install, DHCP networking. Assigned address: `10.0.0.93`. At the end, the script asks about Portainer and socket settings — left all three as default.

## Step 2: Vaultwarden via Docker Compose

On the new LXC:

```bash
mkdir vaultwarden
cd vaultwarden
touch docker-compose.yml
```

Base config from the [Vaultwarden docs](https://github.com/dani-garcia/vaultwarden#docker-compose):

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
```

Two changes from the sample:
- **Removed the `ports:` section** — Caddy + Docker's internal networking handles routing, so Vaultwarden isn't exposed directly.
- **Set `DOMAIN` to `https://passwords.local`** — a `.local` domain by convention, to avoid any conflict with real public domains (note: `.local` is technically reserved for mDNS, which can occasionally cause resolution quirks on macOS — not an issue here).

## Step 3: Add Caddy to the same Docker Compose file

Caddy's own Docker Compose template, from [caddyserver.com/docs/running#docker-compose](https://caddyserver.com/docs/running#docker-compose). Merged into the same file as Vaultwarden (one `services:` line total), with `caddy:latest` and the volumes trimmed to just the Caddyfile mount:

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

Added the `Caddyfile` alongside `docker-compose.yml`:

```
passwords.local {
    tls internal
    reverse_proxy vaultwarden:80
}
```

- `tls internal` — Caddy generates its own (self-signed) certificate.
- `reverse_proxy vaultwarden:80` — forwards HTTPS traffic to the Vaultwarden container on its internal port 80.

Brought the stack up:

```bash
docker compose up
# pressed d to exit the interactive log view once containers were healthy
docker ps          # confirmed both vaultwarden and caddy containers running
ss -antpl          # confirmed ports 80 and 443 bound to docker-proxy
```

## Step 4: Temporary DNS entry

Grabbed the LXC's real IP with `ip a` (looked for the `eth0`-style interface, ignoring Docker's internal interfaces) — `10.0.0.93`.

Added an entry to the local machine's hosts file (`/etc/hosts` on Linux, `C:\Windows\System32\drivers\etc\hosts` on Windows) resolving `passwords.local` → `10.0.0.93`. Temporary — proper DNS comes in Chapter 3.

Browsed to `https://passwords.local`, accepted the self-signed cert warning (expected — Caddy generated it itself via `tls internal`), and reached the Vaultwarden setup page.

## Step 5: Configure Vaultwarden

Created the admin account (email doesn't need to be real) via the web UI. Skipped the Bitwarden browser-extension prompt for now (web app only, initially).

Populated initial password entries per the course's "catch up" checklist:
- Vaultwarden LXC root password
- Proxmox Web UI username/password (linked to the Proxmox web URL for autofill)
- Gitea LXC root password + Gitea app credentials
- Vaultwarden LXC root password + Vaultwarden app credentials

## Notes for next time

- The all-in-one Vaultwarden community script was broken at time of writing — the Docker LXC + Docker Compose route is the reliable path.
- `.local` domains avoid clashing with real internet domains, unlike something like `.com`.
- The hosts-file DNS entry is a stopgap — will be replaced by real home lab DNS in Chapter 3.
- Optional next step: set up the Bitwarden browser extension (Settings → Self Hosted → server URL = `https://passwords.local`) for autofill convenience.
