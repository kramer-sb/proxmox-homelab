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
  functional-docs/
    proxmox.md
    repo-workflow.md
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

### Services

| Service | IP / DNS | Access | Machine Type | Installation Notes | Functional Doc |
|---|---|---|---|---|---|
| Proxmox VE (host) | `10.0.0.52` (DHCP reservation on router) | Web UI `https://10.0.0.52:8006` (MFA), SSH (key-only) | Bare metal — GMKtec NucBox G10 Pro | [01-installing-proxmox.md](installation-notes/01-installing-proxmox.md), [02-security-hardening.md](installation-notes/02-security-hardening.md) | [proxmox.md](functional-docs/proxmox.md) |
| Ubuntu 24.04 (CT 100) | `10.0.0.218` (DHCP) | SSH / Proxmox console | LXC on Proxmox | — | — |

New services get a row here as they come online, alongside their own files in `installation-notes/` and `functional-docs/`.

## Status

Actively growing as the lab expands — new docs and configs get added as new services come online.
