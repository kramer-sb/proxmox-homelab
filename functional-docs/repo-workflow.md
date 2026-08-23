# Functional Documentation — Repo Workflow (proxmox-homelab)

> The clean, repeatable version of how this documentation repo itself is set up and used day to day. Distilled from the original `docs/03-github-workflow.md` build log (now retired — see the backup copy sent separately if the full narrative is ever needed).

## Repo setup (one-time)

- Created via **GitHub Desktop**: `File → New Repository`, named `proxmox-homelab`, local path `E:\ALL Cybersecurity\proxmox\proxmox-homelab`, initialized with a README.
- Published to GitHub as **Public** — doubles as a portfolio piece and a resource for other homelabbers.
- Day-to-day editing happens in **VSCode**, opened directly from GitHub Desktop via `Repository → Open in Visual Studio Code` (keeps both tools pointed at the same local folder/repo).

## Secret scanning setup (per machine/clone)

`gitleaks` runs automatically before every commit via a `pre-commit` hook, so "never commit secrets" is enforced, not just aspirational.

1. Install `gitleaks`:
   - Windows: binary from the [gitleaks releases page](https://github.com/gitleaks/gitleaks/releases), added to PATH — or `scoop install gitleaks` if Scoop is set up.
   - Ubuntu: `sudo apt install gitleaks` (needs the `universe` repo enabled — usually on by default).
2. Install the `pre-commit` framework: `pip install pre-commit`.
3. From the repo root: `pre-commit install` — writes the hook into `.git/hooks/pre-commit`. **Per-clone**, not per-repo — re-run this if the repo is ever cloned fresh onto another machine.

Config lives in `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks:
      - id: gitleaks
```

Runs on every `git commit` regardless of whether it's made through VSCode's Source Control panel or GitHub Desktop (both call plain `git` underneath). First commit after `pre-commit install` is slower (~30s–1min, builds the hook environment); after that it's fast. A commit is blocked if something secret-shaped is staged.

**Note:** `gitleaks` installed via `apt` on Ubuntu reports `version is set by build process` instead of a version number when running `gitleaks version` — a known quirk of the distro-packaged build, not a broken install. `apt policy gitleaks` shows the real installed version.

## Day-to-day commit workflow (VSCode)

1. Open the folder — via GitHub Desktop's `Repository → Open in Visual Studio Code`, or `File → Open Folder` / `File → Open Recent` directly.
2. Click the **Source Control icon** in the Activity Bar (branch-shaped icon, far-left edge) — **not** `Ctrl+Shift+G` (see gotcha below). `View → Source Control` from the menu also works.
3. Stage everything via the checkmark/**+** icon next to "Changes" (or stage files individually).
4. Type a commit message, click **Commit**.
5. Push via **"Sync Changes"** (or the sync icon at the bottom) — committing does **not** automatically push. Always double-check.

## Day-to-day workflow (GitHub Desktop)

GitHub Desktop and VSCode operate on the exact same local repo, so commits made in either show up in the other. If VSCode shows a clean state but GitHub Desktop's main screen shows **"No local changes"** with a banner like *"You have 1 local commit waiting to be pushed to GitHub"* — a commit happened but the push step didn't. Click **"Push origin"** to finish.

## Gotchas (so they don't cost time twice)

- **`Ctrl+Shift+G` conflict:** AMD Software's overlay grabs this shortcut before VSCode can, opening AMD's app instead of Source Control. Use the sidebar icon or `View → Source Control` menu instead.
- **VSCode opening a second window:** opening the repo folder can spawn a *new* VSCode window instead of reusing an empty open one. Harmless — close the extra "Untitled (Workspace)" window. (`window.openFoldersInNewWindow` setting can change this behavior permanently.)
- **Committed ≠ Pushed:** always glance at GitHub Desktop (or the sync icon in VSCode) to confirm a commit actually made it to GitHub, especially before closing everything down.
