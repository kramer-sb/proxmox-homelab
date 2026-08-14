# proxmox-homelab

A documented build log of my home lab, running on Proxmox VE. This repo exists for two reasons: to thoroughly document the setup as I go (so I don't have to relearn things by trial and error), and to serve as a portfolio piece / learning resource for other homelabbers following a similar path.

## Repo structure

```
proxmox-homelab/
  README.md
  .gitignore
  .gitattributes
  .pre-commit-config.yaml
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

## Status

Actively growing as the lab expands — new docs and configs get added as new services come online.
