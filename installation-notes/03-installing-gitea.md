# Installation Notes — Installing Gitea (Version Control)

Following JHT *Home Lab: Beginner Buildout*, Section 2.6. First of the "Big 3" starter apps.

**Result:** Gitea running at `10.0.0.15:3000`.

## Step 1: Install the LXC via community script

Proxmox Community Scripts site → searched for **Gitea (LXC)**. The install card's Notes section flags an important detail: select **SQLite3** as the database type on first run — noted before starting.

Copied the install command and ran it in the Proxmox shell, chose **Advanced Install**, walked through the prompts (hostname, resources, network — DHCP). Assigned address: `10.0.0.15`.

When the script finished, it printed the URL/port Gitea is running on directly in the terminal output — `10.0.0.15:3000`.

## Step 2: Configure Gitea (first-run setup)

Navigated to `http://10.0.0.15:3000` (HTTP, not HTTPS — that comes in Chapter 3).

- **Database Type:** SQLite3 (per the install card's note), default path left as-is.
- Site title left/changed as preferred.
- Settings file location noted for later reference: `/etc/gitea/app.ini` — will need this in Chapter 3 for DNS/HTTPS config.

Clicked **Install Gitea**. Landed on the registration page — first account created here is automatically an admin (same pattern as most self-hosted apps: first user = admin, later users need to be promoted).

## Step 3: Create the home lab notebook repository

Created a new repository (top-right **+** → **New Repository**) to eventually hold home lab notes. Left non-name settings as default.

> Note: unlike GitHub, a self-hosted Gitea repo doesn't need to be marked private here — nobody else can see it. It's fine if credentials/tokens end up in notes here, as long as they never leave the self-hosted environment.

## Step 4: Add and verify an SSH key

Preferred SSH over HTTPS auth for push/pull (no re-entering credentials every time).

1. Profile → **Settings** → **SSH / GPG Keys** → **Add Key**.
2. Named the key, pasted the **public** key generated back in Chapter 1 (`ssh_key.pub` — same key used for Proxmox host SSH access).
3. Clicked **Verify**, copied Gitea's provided verification command (OS/shell-specific), ran it locally with the real path to the key substituted in for `-f /path_to_key`, copied the resulting signature.
4. Pasted the signature into the **Armored SSH signature** field, clicked **Verify** — success.

## Step 5: First push via SSH

From the repo's Quick Guide (switched to the SSH variant):

```bash
touch README.md
git init
git checkout -b main
# added content to README.md
git add README.md
git commit -m "first commit"
git remote add origin gitea@10.0.0.15:<username>/<repo-name>.git
git push -u origin main
```

Confirmed the push succeeded and the file appeared in Gitea after a refresh.

## Notes for next time

- Gitea's SSH push/pull runs on the LXC's own port 22 — separate from the Proxmox host's SSH.
- Going forward, home lab notes/code get stored in Gitea per the course's recommendation (this repo on GitHub currently fills that role until Gitea is fully wired into the day-to-day workflow).
- Will revisit this app in Chapter 3 (`/etc/gitea/app.ini`) to add DNS + HTTPS.
