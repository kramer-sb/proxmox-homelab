# 03 — GitHub Workflow for This Project

Documenting how version control got set up for this home lab project, and the working conventions established — mostly so future-me (or anyone else following along) doesn't have to relearn this by trial and error.

## Why version control this project

Partly following the course's own advice to document everything thoroughly, and partly to make this usable as both a learning resource for other homelabbers and a portfolio piece.

## Repo setup

- Created via **GitHub Desktop**: `File → New Repository`, named `proxmox-homelab`, local path `E:\ALL Cybersecurity\proxmox\proxmox-homelab`, initialized with a README.
- Published to GitHub, ultimately set to **Public** — the goal is for this repo to double as a portfolio piece and something other homelabbers can learn from.
- Day-to-day editing happens in **VSCode**, opened directly from GitHub Desktop via `Repository → Open in Visual Studio Code` (keeps both tools pointed at the exact same local folder/repo).

## What goes in this repo — and what doesn't

```
proxmox-homelab/
  README.md
  .gitignore
  .gitattributes
  docs/
    01-proxmox-install.md
    02-starting-stopping-proxmox.md
    03-github-workflow.md      <- this file
  configs/                     <- config/script files, as the lab grows
```

`docs/` holds numbered, chronological build logs. `configs/` will hold actual config and script files (docker-compose files, LXC setup notes, etc.) as more services get added.

A `.gitignore` was added early — **before** any config files existed — specifically so secrets never get staged in the first place:

```
.env
*.key
*.pem
credentials.json
secrets.yml
*.tfstate
```

## Security / redaction philosophy (public repo)

Not everything needs to be hidden, and not everything is worth showing:

- **Fine to leave visible:** internal/private IP addresses (`10.0.0.x`, `192.168.x.x`) — these are meaningless outside the LAN, and most public homelab repos show them openly.
- **Never commit:** root/account passwords, API tokens (this will matter once Tailscale and other services enter the picture), SSH private keys, or any public-facing IP/domain if something ends up exposed to the internet.
- For any config that needs a credential, use a placeholder (e.g., `TAILSCALE_AUTHKEY=your-key-here`) or reference an environment variable rather than hardcoding the real value.
- **Future to-do:** install [`gitleaks`](https://github.com/gitleaks/gitleaks) to automatically scan for anything secret-shaped before it gets pushed — cheap insurance for a public repo that'll keep growing.

## Day-to-day commit workflow (VSCode)

1. Open the folder — via GitHub Desktop's `Repository → Open in Visual Studio Code`, or `File → Open Folder` / `File → Open Recent` directly in VSCode.
2. Click the **Source Control icon** in the Activity Bar (the branch-shaped icon on the far-left edge) — **not** the `Ctrl+Shift+G` shortcut, which AMD Software has hijacked on this machine. Use `View → Source Control` from the menu if in doubt.
3. Stage everything via the **checkmark/+ icon** next to "Changes" (or stage files individually).
4. Type a commit message, click **Commit**.
5. Push via **"Sync Changes"** (or the sync icon at the bottom) — committing does **not** automatically push. Always double-check.

## Day-to-day workflow (GitHub Desktop)

GitHub Desktop and VSCode operate on the exact same local repo, so changes/commits made in either tool show up in the other. If VSCode shows a clean state but GitHub Desktop's main screen displays **"No local changes"** alongside a banner like *"You have 1 local commit waiting to be pushed to GitHub"* — that means a commit happened but the push step didn't. Click **"Push origin"** to finish sending it up.

## Gotchas encountered (so they don't cost time twice)

- **`Ctrl+Shift+G` conflict:** AMD Software's overlay grabs this shortcut before VSCode can, so it opens AMD's app instead of the Source Control panel. Use the sidebar icon or `View → Source Control` menu instead.
- **VSCode opening a second window:** Opening the repo folder (via GitHub Desktop's button or `File → Open Folder`) can spawn a *new* VSCode window rather than reusing an already-open empty one. Harmless — just close the extra empty "Untitled (Workspace)" window. (There's a `window.openFoldersInNewWindow` setting to change this behavior permanently, if it becomes annoying.)
- **Committed ≠ Pushed:** always glance at GitHub Desktop (or the sync icon in VSCode) to confirm a commit actually made it to GitHub, especially before closing everything down.
