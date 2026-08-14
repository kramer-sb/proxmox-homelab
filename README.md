# proxmox-homelab

A documented build log of my home lab, running on Proxmox VE. This repo exists for two reasons: to thoroughly document the setup as I go (so I don't have to relearn things by trial and error), and to serve as a portfolio piece / learning resource for other homelabbers following a similar path.

## Repo structure

```
proxmox-homelab/
  README.md
  .gitignore
  .gitattributes
  docs/
    01-proxmox-install.md
    02-starting-stopping-proxmox.md
    03-github-workflow.md
  configs/
```

- **`docs/`** — numbered, chronological build logs. Each file documents a stage of the build in the order it happened.
- **`configs/`** — config and script files (docker-compose files, LXC setup notes, etc.) as more services get added to the lab.

## Docs index

| # | Doc | Covers |
|---|-----|--------|
| 01 | [proxmox-install.md](docs/01-proxmox-install.md) | Initial Proxmox VE installation |
| 02 | [starting-stopping-proxmox.md](docs/02-starting-stopping-proxmox.md) | Powering the host up/down safely |
| 03 | [github-workflow.md](docs/03-github-workflow.md) | How this repo is version-controlled day to day |

## Security / redaction philosophy

This is a **public** repo, so a few ground rules are followed throughout:

- **Fine to show:** internal/private IP addresses (`10.0.0.x`, `192.168.x.x`) — meaningless outside the LAN, and common in public homelab repos.
- **Never committed:** root/account passwords, API tokens, SSH private keys, or any public-facing IP/domain.
- Configs that need a credential use a placeholder (e.g. `TAILSCALE_AUTHKEY=your-key-here`) or an environment variable instead of a real value.
- A `.gitignore` blocks common secret-shaped files (`.env`, `*.key`, `*.pem`, `credentials.json`, `secrets.yml`, `*.tfstate`) from ever being staged.
- **Planned:** [`gitleaks`](https://github.com/gitleaks/gitleaks) to scan for secrets automatically before they get pushed.

## Tooling

- **GitHub Desktop** for repo management and sync (push/pull).
- **VSCode** for day-to-day editing, opened directly from GitHub Desktop so both tools stay pointed at the same local folder.

## Status

Actively growing as the lab expands — new docs and configs get added as new services come online.