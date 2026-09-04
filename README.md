# proxmox-homelab

A documented build log of my home lab, running on Proxmox VE. This repo exists for two reasons: to thoroughly document the setup as I go (so I don't have to relearn things by trial and error), and to serve as a portfolio piece / learning resource for other homelabbers following a similar path.

<small>*Note: I am creating my home lab by following Joram Stith's [Home Lab: Beginner Buildout](https://www.justhacking.com/course/home-lab-beginner-buildout/) on Just Hacking Training's site. I strongly encourage you to take the course! You can pay what you can afford, which is a blessing for so many.*</small>

## Repo structure

```
proxmox-homelab/
  README.md
  .gitignore
  .gitattributes
  .pre-commit-config.yaml
  installation-notes/
    01-installing-proxmox.md
    02-security-hardening.md
    03-installing-gitea.md
    04-installing-vaultwarden-and-caddy.md
    05-installing-uptime-kuma.md
    06-static-ip-networking.md
    07-home-lab-dns.md
    08-ssl-termination.md
    09-tailscale-remote-access.md
    09a-tailscale-icmp-troubleshooting-addendum.md
    09b-mobile-remote-access-testing.md
  functional-docs/
    proxmox.md
    repo-workflow.md
    gitea.md
    vaultwarden.md
    uptime-kuma.md
    coredns.md
    tailscale.md
    trusting-caddy-local-ca-windows.md
  configs/
```

- **`installation-notes/`** - raw notes as I go: links, commands, screenshots, and the errors. Written in the moment, for each thing I try.
- **`functional-docs/`** - short, clean, ordered steps distilled from installation notes once something works. No errors, just what to copy-paste to reproduce it.
- **`configs/`** - config and script files (docker-compose files, LXC setup notes, etc.) as more services get added to the lab.

This repo previously used a `docs/` folder of numbered, chronological build logs. It's been consolidated into the two-folder structure above (see the **Home Lab Notebook** section below) so installation notes and functional/reference docs don't live mixed together in one folder.

## Security / redaction philosophy

This is a **public** repo, so a few ground rules are followed throughout:

- **Fine to show:** internal/private IP addresses (`10.0.0.x`, `192.168.x.x`) - meaningless outside the LAN, and common in public homelab repos.
- **Never committed:** root/account passwords, API tokens, SSH private keys, or any public-facing IP/domain.
- Configs that need a credential use a placeholder (e.g. `TAILSCALE_AUTHKEY=your-key-here`) or an environment variable instead of a real value.
- A `.gitignore` blocks common secret-shaped files (`.env`, `*.key`, `*.pem`, `credentials.json`, `secrets.yml`, `*.tfstate`) from ever being staged.

## Secret scanning (gitleaks pre-commit hook)

To make the "never commit secrets" rule enforced rather than just aspirational, `gitleaks` runs automatically before every commit via a `pre-commit` hook.

**Setup (per machine/clone):**

1. Install `gitleaks` itself:
   - Windows: binary from the [gitleaks releases page](https://github.com/gitleaks/gitleaks/releases), added to PATH - or `scoop install gitleaks` if Scoop is set up.
   - Ubuntu: `sudo apt install gitleaks` (needs the `universe` repo enabled - usually already on by default).
2. Install the `pre-commit` framework: `pip install pre-commit`.
3. From the repo root, run `pre-commit install` - this writes the actual hook into `.git/hooks/pre-commit`. **This step is per-clone**, not per-repo; if the repo is ever cloned fresh onto another machine, it needs to be run again there.

**Config:** a `.pre-commit-config.yaml` in the repo root defines the hook:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks:
      - id: gitleaks
```

**How it behaves:**

- Runs on every `git commit`, regardless of whether the commit is made through VSCode's Source Control panel or GitHub Desktop - both call plain `git` underneath, so the hook fires either way.
- The very first commit after `pre-commit install` is slower (~30s-1min), since it downloads and builds the gitleaks hook environment. After that it's fast and reused.
- Scans staged changes for hardcoded secrets before the commit is allowed to complete. A commit is blocked if something secret-shaped is found.

**Note:** gitleaks installed via `apt` on Ubuntu reports `version is set by build process` instead of an actual version number when running `gitleaks version` - this is a known quirk of the distro-packaged build, not a broken install. `apt policy gitleaks` shows the real installed version if needed.

## Tooling

- **GitHub Desktop** for repo management and sync (push/pull).
- **VSCode** for day-to-day editing, opened directly from GitHub Desktop so both tools stay pointed at the same local folder.

## Home Lab Notebook

This repo follows the three-part "Home Lab Notebook" documentation method taught in the JHT *Home Lab: Beginner Buildout* course:

- **`installation-notes/`** - what actually happened while getting something working, gotchas and dead ends included. Written first, in the moment.
- **`functional-docs/`** - the clean, repeatable version distilled from installation notes once a service is working the way I want. No errors, just the steps and commands that worked, ready to copy-paste.
- **This README** - the master document. The table below tracks every service running in the lab: what it is, its address, how it's reached, and links to its installation notes + functional doc.

`functional-docs/repo-workflow.md` covers this repo's own Git/GitHub setup and day-to-day commit workflow - not a home lab service, but documented the same way since it's a repeatable process worth having steps for.

### App Table

Every app/service in the lab gets a row. Add one as soon as something new comes online - don't wait.

As of Chapter 3, all machines run on **static IPs** (increments of 5, starting at `10.0.0.5` for the Proxmox host) instead of DHCP reservations, and are reachable by name via the CoreDNS server (`10.0.0.30`) rather than raw IP. Apps fronted by Caddy use `https://` with no port number; Caddy terminates TLS with a self-signed local cert (`tls internal`) and forwards internally.

| App Name | IP Address | Service Ports | Access Type | Installation Method | Machine Type | Installation Notes | Functional Doc |
|---|---|---|---|---|---|---|---|
| Proxmox VE (host) | `10.0.0.5` (static) | 8006 (web UI), 22 (SSH) | Web UI (MFA) for management, SSH (key-only) for shell | ISO install (manual) | Bare metal - GMKtec NucBox G10 Pro | [01-installing-proxmox.md](installation-notes/01-installing-proxmox.md), [02-security-hardening.md](installation-notes/02-security-hardening.md), [06-static-ip-networking.md](installation-notes/06-static-ip-networking.md) | [proxmox.md](functional-docs/proxmox.md) |
| Gitea | `10.0.0.10` (static) | 3000 (HTTP, loopback only), 80/443 via Caddy, 22 (SSH push/pull) | `https://gitea.lab` for web UI (Caddy reverse proxy, self-signed cert), SSH (key-based) for git push/pull | Community script (Proxmox Helper Scripts, LXC, Advanced Install) | LXC on Proxmox | [03-installing-gitea.md](installation-notes/03-installing-gitea.md), [06-static-ip-networking.md](installation-notes/06-static-ip-networking.md), [08-ssl-termination.md](installation-notes/08-ssl-termination.md) | [gitea.md](functional-docs/gitea.md) |
| Uptime Kuma | `10.0.0.20` (static) | 3001 (HTTP, loopback only), 80/443 via Caddy | `https://kuma.lab` for web UI (Caddy reverse proxy, self-signed cert) | Community script (Proxmox Helper Scripts, LXC, Advanced Install) | LXC on Proxmox | [05-installing-uptime-kuma.md](installation-notes/05-installing-uptime-kuma.md), [06-static-ip-networking.md](installation-notes/06-static-ip-networking.md), [08-ssl-termination.md](installation-notes/08-ssl-termination.md) | [uptime-kuma.md](functional-docs/uptime-kuma.md) |
| Vaultwarden | `10.0.0.15` (static) | 80, 443 (HTTPS via Caddy reverse proxy) | `https://vaultwarden.lab` via browser (Caddy reverse proxy, self-signed cert), or Bitwarden app/extension | Docker LXC (community script) + Docker Compose (Vaultwarden + Caddy) | LXC on Proxmox (Docker) | [04-installing-vaultwarden-and-caddy.md](installation-notes/04-installing-vaultwarden-and-caddy.md), [06-static-ip-networking.md](installation-notes/06-static-ip-networking.md), [08-ssl-termination.md](installation-notes/08-ssl-termination.md) | [vaultwarden.md](functional-docs/vaultwarden.md) |
| CoreDNS | `10.0.0.30` (static) | 53 (DNS) | Internal DNS only - no web UI. Resolves `.lab` hostnames for the lab, forwards everything else upstream | Manual install | LXC on Proxmox | [07-home-lab-dns.md](installation-notes/07-home-lab-dns.md) | [coredns.md](functional-docs/coredns.md) |
| Kali Linux | `10.0.0.25` (static) | N/A (workstation VM, no persistent service) | Console/desktop access via Proxmox | Manual ISO install | VM on Proxmox | [06-static-ip-networking.md](installation-notes/06-static-ip-networking.md) | - |
| Tailscale Subnet Router (ts-router) | `10.0.0.35` (static) <!-- confirm actual IP assigned --> | N/A (relay only, no web UI) | No direct access; advertises the `10.0.0.0/24` route to the tailnet so any Tailscale-connected device (including off-network devices, e.g. a phone on cellular data) can reach every lab service by IP or `.lab` domain | Community script (Proxmox Helper Scripts, LXC) + `tailscale up` / `tailscale set --advertise-routes` | LXC on Proxmox | [09-tailscale-remote-access.md](installation-notes/09-tailscale-remote-access.md), [09a-tailscale-icmp-troubleshooting-addendum.md](installation-notes/09a-tailscale-icmp-troubleshooting-addendum.md), [09b-mobile-remote-access-testing.md](installation-notes/09b-mobile-remote-access-testing.md) | [tailscale.md](functional-docs/tailscale.md) |

*Removed per the course's Chapter 2 cleanup: the Ubuntu 24.04 LXC (CT 100) built in Chapter 1, and the practice Ubuntu/Debian VMs/LXCs from Section 2.3/2.4 - no longer running, so they don't have rows here.*

**Note on DNS:** the Xfinity gateway's admin UI does not expose a DHCP-wide DNS override field (confirmed locked down - see `07-home-lab-dns.md`), so `10.0.0.30` is set as the DNS server manually on each static-IP lab machine (Proxmox host itself, and the Windows management host, both use gateway/Xfinity DNS instead - only the lab VMs/LXCs point at CoreDNS directly). As of Chapter 4, devices connected to the tailnet (the Windows host, and any other Tailscale-connected device such as a phone) resolve `.lab` domains through a Split DNS entry pushed from the Tailscale admin console, rather than needing CoreDNS set as their local DNS server. This is what makes `.lab` names resolve correctly even when away from home - see `functional-docs/tailscale.md`.

### New App To-Do List

Steps to repeat every time a new app gets added to the lab. This list grows as later chapters bring in networking, remote access, and backups.

- [ ] Create a dedicated non-root user/account for the app (root only where the app leaves no other choice).
- [ ] Install the app (community script, Docker, or manual) - capture the raw process, including any errors, in `installation-notes/`.
- [ ] Set a **static IP** for the machine (Proxmox DNS tab for LXCs, guest OS network settings for VMs).
- [ ] Add a DNS entry for the machine in CoreDNS (`.lab` domain).
- [ ] Add HTTPS with Caddy if applicable (Docker Compose method for Docker-based apps, native/systemd install otherwise).
- [ ] Confirm the app is reachable at its assigned DNS name over HTTPS (or documented access method).
- [ ] Distill the working steps into `functional-docs/<app>.md`.
- [ ] Add a row for the app in the **App Table** above.
- [ ] Add a monitor for the app in [Uptime Kuma](functional-docs/uptime-kuma.md), pointed at its `.lab` domain name (ping at minimum; HTTP(S) if it serves a web UI).
- [ ] Push the installation notes + functional doc to [Gitea](functional-docs/gitea.md) - GitHub, via the [repo workflow](functional-docs/repo-workflow.md), currently serves as the primary remote for this repo.
- [ ] Store any credentials in [Vaultwarden](functional-docs/vaultwarden.md) - never in this repo.

## Status

Chapter 3 of the course (home lab networking - static IPs, CoreDNS, and SSL termination with Caddy) is fully documented as of this update. All apps are now reachable by `.lab` domain name over HTTPS.

Chapter 4 (remote access via Tailscale) is in progress. A dedicated subnet router LXC (`ts-router`) advertises the full `10.0.0.0/24` lab subnet to the tailnet, and Split DNS is configured so `.lab` domains resolve correctly from any Tailscale-connected device, including devices away from home. Verified working from a phone on cellular data with Wi-Fi disabled. Remaining Chapter 4 sections (sharing services with others, further remote-access hardening) not yet started.

Actively growing as the lab expands - new docs and configs get added as new services come online.
