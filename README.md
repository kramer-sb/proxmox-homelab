# proxmox-homelab

A documented build log of my home lab, running on Proxmox VE. This repo exists for two reasons: to thoroughly document the setup as I go (so I don't have to relearn things by trial and error), and to serve as a portfolio piece / learning resource for other homelabbers following a similar path.

*Note: I am creating my home lab by following Joram Stith's [Home Lab: Beginner Buildout](https://www.justhacking.com/course/home-lab-beginner-buildout/) on Just Hacking Training's site. I strongly encourage you to take the course! You can pay what you can afford, which is a blessing for so many.*

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
  functional-docs/
    proxmox.md
    repo-workflow.md
    gitea.md
    vaultwarden.md
    uptime-kuma.md
  configs/
```

- **`installation-notes/`** — raw notes as I go: links, commands, screenshots, and the errors. Written in the moment, for each thing I try.
- **`functional-docs/`** — short, clean, ordered steps distilled from installation notes once something works. No errors, just what to copy-paste to reproduce it.
- **`configs/`** — config and script files (docker-compose files, LXC setup notes, etc.) as more services get added to the lab.

This repo previously used a `docs/` folder of numbered, chronological build logs. It's been consolidated into the two-folder structure above (see the **Home Lab Notebook** section below) so installation notes and functional/reference docs don't live mixed together in one folder.

## Security / redaction philosophy

This is a **public** repo, so a few ground rules are followed throughout:

- **Fine to show:** internal/private IP addresses (`10.0.0.x`, `192.168.x.x`) — meaningless outside the LAN, and common in public homelab repos.
- **Never committed:** root/account passwords, API tokens, SSH private keys, or any public-facing IP/domain.
- Configs that need a credential use a placeholder (e.g. `TAILSCALE_AUTHKEY=your-key-here`) or an environment variable instead of a real value.
- A `.gitignore` blocks common secret-shaped files (`.env`, `*.key`, `*.pem`, `credentials.json`, `secrets.yml`, `*.tfstate`) from ever being staged.

## Secret scanning (gitleaks pre-commit hook)

To make the "never commit secrets" rule enforced rather than just aspirational, `gitleaks` runs automatically before every commit via a `pre-commit` hook.

**Setup (per machine/clone):**

1. Install `gitleaks` itself:
   - Windows: binary from the [gitleaks releases page](https://github.com/gitleaks/gitleaks/releases), added to PATH — or `scoop install gitleaks` if Scoop is set up.
   - Ubuntu: `sudo apt install gitleaks` (needs the `universe` repo enabled — usually already on by default).
2. Install the `pre-commit` framework: `pip install pre-commit`.
3. From the repo root, run `pre-commit install` — this writes the actual hook into `.git/hooks/pre-commit`. **This step is per-clone**, not per-repo; if the repo is ever cloned fresh onto another machine, it needs to be run again there.

**Config:** a `.pre-commit-config.yaml` in the repo root defines the hook:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks:
      - id: gitleaks
```

**How it behaves:**

- Runs on every `git commit`, regardless of whether the commit is made through VSCode's Source Control panel or GitHub Desktop — both call plain `git` underneath, so the hook fires either way.
- The very first commit after `pre-commit install` is slower (~30s–1min), since it downloads and builds the gitleaks hook environment. After that it's fast and reused.
- Scans staged changes for hardcoded secrets before the commit is allowed to complete. A commit is blocked if something secret-shaped is found.

**Note:** gitleaks installed via `apt` on Ubuntu reports `version is set by build process` instead of an actual version number when running `gitleaks version` — this is a known quirk of the distro-packaged build, not a broken install. `apt policy gitleaks` shows the real installed version if needed.

## Tooling

- **GitHub Desktop** for repo management and sync (push/pull).
- **VSCode** for day-to-day editing, opened directly from GitHub Desktop so both tools stay pointed at the same local folder.

## Home Lab Notebook

This repo follows the three-part "Home Lab Notebook" documentation method taught in the JHT *Home Lab: Beginner Buildout* course:

- **`installation-notes/`** — what actually happened while getting something working, gotchas and dead ends included. Written first, in the moment.
- **`functional-docs/`** — the clean, repeatable version distilled from installation notes once a service is working the way I want. No errors, just the steps and commands that worked, ready to copy-paste.
- **This README** — the master document. The table below tracks every service running in the lab: what it is, its address, how it's reached, and links to its installation notes + functional doc.

`functional-docs/repo-workflow.md` covers this repo's own Git/GitHub setup and day-to-day commit workflow — not a home lab service, but documented the same way since it's a repeatable process worth having steps for.

### App Table

Every app/service in the lab gets a row. Add one as soon as something new comes online — don't wait.

| App Name | IP Address | Service Ports | Access Type | Installation Method | Machine Type | Installation Notes | Functional Doc |
|---|---|---|---|---|---|---|---|
| Proxmox VE (host) | `10.0.0.52` (DHCP reservation on router) | 8006 (web UI), 22 (SSH) | Web UI (MFA) for management, SSH (key-only) for shell | ISO install (manual) | Bare metal — GMKtec NucBox G10 Pro | [01-installing-proxmox.md](installation-notes/01-installing-proxmox.md), [02-security-hardening.md](installation-notes/02-security-hardening.md) | [proxmox.md](functional-docs/proxmox.md) |
| Gitea | `10.0.0.15` | 3000 (HTTP web UI), 22 (SSH push/pull) | HTTP for web UI, SSH (key-based) for git push/pull | Community script (Proxmox Helper Scripts, LXC, Advanced Install) | LXC on Proxmox | [03-installing-gitea.md](installation-notes/03-installing-gitea.md) | [gitea.md](functional-docs/gitea.md) |
| Uptime Kuma | `10.0.0.199` | 3001 (HTTP web UI) | HTTP for web UI | Community script (Proxmox Helper Scripts, LXC, Advanced Install) | LXC on Proxmox | [05-installing-uptime-kuma.md](installation-notes/05-installing-uptime-kuma.md) | [uptime-kuma.md](functional-docs/uptime-kuma.md) |
| Vaultwarden | `10.0.0.93` | 80, 443 (HTTPS via Caddy reverse proxy) | HTTPS via browser at `https://passwords.local`, or Bitwarden app/extension | Docker LXC (community script) + Docker Compose (Vaultwarden + Caddy) | LXC on Proxmox (Docker) | [04-installing-vaultwarden-and-caddy.md](installation-notes/04-installing-vaultwarden-and-caddy.md) | [vaultwarden.md](functional-docs/vaultwarden.md) |

*Removed per the course's Chapter 2 cleanup: the Ubuntu 24.04 LXC (CT 100) built in Chapter 1, and the practice Ubuntu/Debian VMs/LXCs from Section 2.3/2.4 — no longer running, so they don't have rows here. The Kali Linux VM from Section 2.3 was likewise a one-off exercise and was never kept running.*

### New App To-Do List

Steps to repeat every time a new app gets added to the lab. This list grows as later chapters bring in networking, remote access, and backups.

- [ ] Create a dedicated non-root user/account for the app (root only where the app leaves no other choice).
- [ ] Install the app (community script, Docker, or manual) — capture the raw process, including any errors, in `installation-notes/`.
- [ ] Confirm the app is reachable at its assigned IP and service port(s).
- [ ] Set a DHCP reservation for the app's IP if it's a separate VM/LXC.
- [ ] Distill the working steps into `functional-docs/<app>.md`.
- [ ] Add a row for the app in the **App Table** above.
- [ ] Add a monitor for the app in [Uptime Kuma](functional-docs/uptime-kuma.md) (ping at minimum; HTTP(S) if it serves a web UI).
- [ ] Push the installation notes + functional doc to [Gitea](functional-docs/gitea.md) — GitHub, via the [repo workflow](functional-docs/repo-workflow.md), currently serves as the primary remote for this repo.
- [ ] Store any credentials in [Vaultwarden](functional-docs/vaultwarden.md) — never in this repo.

## Status

Chapter 2 of the course (Proxmox VE + the "Big 3" starter apps — Gitea, Vaultwarden, Uptime Kuma) is fully documented as of this update. Actively growing as the lab expands — new docs and configs get added as new services come online.
