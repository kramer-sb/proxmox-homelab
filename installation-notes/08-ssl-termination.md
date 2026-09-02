# 08 - SSL Termination (Complete)

Status: **Complete** — Course Section 3.5 (SSL Termination). All lab services
now reachable over `https://` (no port number) using Caddy as a reverse
proxy, instead of plain `http://` with a manually-remembered port.

## Goal

Put Caddy in front of each service to:
1. Terminate HTTPS (encrypt traffic between browser/client and the LXC)
2. Forward decrypted traffic internally to the app's real, unencrypted port
3. Let each app be reached at a clean URL (`https://gitea.lab`) instead of
   `http://gitea.lab:3000`

Two install patterns used depending on how the underlying app runs:
- **Docker apps** (Vaultwarden) — Caddy runs as a container in the same
  Docker Compose file, talking to the app container by container name.
- **Native/systemd apps** (Gitea, Uptime Kuma) — Caddy installed directly on
  the LXC's OS, listening on the LAN IP (ports 80/443), forwarding to the
  app on `127.0.0.1` (loopback only, not exposed to the network directly).

## Vaultwarden — Completed (Docker method)

Already running Caddy in `docker-compose.yml` from the original Vaultwarden
setup (see `06-static-ip-networking.md`). Updated for the new DNS name:

1. Edited `Caddyfile` — changed the site block to `vaultwarden.lab`
2. Edited `docker-compose.yml` — changed the `vaultwarden` container's
   `DOMAIN` environment variable to `https://vaultwarden.lab`
3. Applied with:
   ```
   docker compose down
   docker compose up -d
   ```
4. Verified `https://vaultwarden.lab/#/vault` loads (self-signed cert
   warning expected — see note below)
5. Removed the old `passwords.local` entry from the Windows hosts file —
   no longer needed now that CoreDNS + Caddy handle it

## Gitea — Completed (native/systemd method)

1. Checked `ss -antpl` — confirmed Gitea was listening on `*:3000` (all
   interfaces)
2. Edited `/etc/gitea/app.ini`, `[server]` section:
   - `SSH_DOMAIN` → `gitea.lab`
   - `DOMAIN` → `gitea.lab`
   - Added `HTTP_ADDR = 127.0.0.1` below `HTTP_PORT`
   - `ROOT_URL` → `https://gitea.lab`
3. `systemctl restart gitea`
4. Re-checked `ss -antpl` — confirmed Gitea now listens on `127.0.0.1:3000`
   only
5. Installed Caddy (see troubleshooting note below — took two attempts)
6. Configured `/etc/caddy/Caddyfile`:
   ```
   gitea.lab {
       tls internal
       reverse_proxy 127.0.0.1:3000
   }
   ```
7. `systemctl restart caddy`
8. Re-checked `ss -antpl` — confirmed Caddy listening on `*:80` and `*:443`,
   Gitea still on `127.0.0.1:3000` only
9. Verified `https://gitea.lab` loads (self-signed cert warning expected)

### Troubleshooting note: Caddy install appeared to do nothing (`/usr/bin/caddy` missing)

After running through the official Caddy install steps (adding the
Cloudsmith GPG keyring + repo, then `sudo apt install caddy`), `caddy
version` and `which caddy` both came back empty, and `dpkg -l | grep caddy`
showed nothing installed either — meaning the package genuinely never
installed, not just a PATH issue.

**Cause:** the `apt install caddy` confirmation prompt (`Continue? [Y/n]`)
was not answered/confirmed on the first attempt, so the install silently did
nothing.

**Fix:** simply re-ran `sudo apt install caddy` and confirmed the `[Y/n]`
prompt this time. Installed successfully.

**Side note — which repo it actually installed from:** the working install
pulled `caddy` from Debian's own `trixie-security` repo (version `2.6.2`),
not from the Cloudsmith repo added earlier in the official install steps.
Debian Trixie ships Caddy natively now, so `apt` found and preferred that
copy. Functionally equivalent for this course (`tls internal` and
`reverse_proxy` work identically) — just a slightly older point release than
whatever Cloudsmith's latest build would be. No action needed.

### Troubleshooting note: root mail about a failed `sudo` from `caddy` user (harmless)

After Caddy started, a system mail appeared (`You have mail in
/var/mail/root`) containing a security log entry:

```
uptimekuma : Sep  2 10:06:48 : caddy : user NOT in sudoers ; PWD=/ ; USER=root ; COMMAND=/usr/bin/tee /usr/local/share/ca-certificates/Caddy_Local_Authority...crt
```

**What this means:** on startup, Caddy (with `tls internal`) tries to
install its self-signed local root CA into the LXC's *own* OS-level trust
store, so command-line tools on that same machine (e.g. `curl` run locally)
would also trust its certs. The `caddy` service account isn't in `sudoers`,
so this attempt was denied and logged.

**Why it doesn't matter:** this only affects trust for tools running
*inside* that LXC. It has no effect on browser trust from an actual client
machine (Windows), which is handled separately by accepting the self-signed
cert warning in-browser. No fix applied — left as-is, cosmetic only.

## Uptime Kuma — Completed (native/systemd method)

Followed the same pattern as Gitea (Caddy installed natively, `tls
internal`, `reverse_proxy 127.0.0.1:<uptime-kuma-port>`). Verified
`https://kuma.lab` loads.

### Follow-up: monitors showed red after the domain change

Per the course, updated existing Uptime Kuma monitors to point at the new
`.lab` domain names (e.g. `gitea.lab`, `https://proxmox.lab:8006`) instead
of the old raw IPs/ports, since the monitors were still pointed at
pre-DNS/pre-SSL addresses.

## General note: self-signed certificate warnings are expected

All three services (`vaultwarden.lab`, `gitea.lab`, `kuma.lab`) show a
browser "Not secure" / self-signed certificate warning. This is expected
and correct behavior with Caddy's `tls internal` directive — internal `.lab`
domains can't get a publicly-trusted certificate (no public CA can verify
control of a non-public domain), so Caddy self-signs using its own local CA.
The connection is still encrypted; the browser just doesn't trust the issuer
by default. Accepting the warning per-site is sufficient and is what the
course expects. (Optional, not required: Caddy's local root CA cert could be
imported into the Windows Trusted Root store to remove the warning
entirely — not done as part of this course.)

## What's Next

- **Chapter 3 wrap-up items** (per course "Home Lab Notebook"):
  - CoreDNS added to master services list — done via `07-home-lab-dns.md`
  - App IPs/DNS names kept current in notes — reflected across
    `06-static-ip-networking.md`, `07-home-lab-dns.md`, this file
  - Functional doc for CoreDNS install — `functional-docs/proxmox-lxc-dns-config.md`
  - New-service checklist updated to include: static IP → CoreDNS entry →
    Caddy (if applicable) → Uptime Kuma monitor → documentation
- **Chapter 4**: remote access to the home lab via Tailscale
