# Functional Documentation — Gitea (Version Control)

> Distilled from [`installation-notes/03-installing-gitea.md`](../installation-notes/03-installing-gitea.md).

## At a glance

| | |
|---|---|
| **Host** | LXC on Proxmox (community script) |
| **IP** | `10.0.0.15` |
| **Web UI** | `http://10.0.0.15:3000` |
| **Git (SSH)** | `git@10.0.0.15` |

## Install (reproducible steps)

1. On the [Proxmox Community Scripts site](https://community-scripts.github.io/ProxmoxVE/), find **Gitea (LXC)** and copy the install command.
2. Run it in the Proxmox host's **Shell**, choose **Advanced Install**, configure resources/network (DHCP).
3. Note the URL/port printed at the end of the script.
4. Browse to the printed URL (HTTP): select **SQLite3** as the database type, leave the default DB path, set a site title.
5. Click **Install Gitea**, then register the first account on the resulting page — this account is the admin.
6. Note the config file path shown on the install page: `/etc/gitea/app.ini` (needed later for DNS/HTTPS).

## Add SSH key + first push

1. Profile → **Settings → SSH / GPG Keys → Add Key**, paste the **public** key.
2. Click **Verify**, run the OS-specific verification command Gitea provides (substitute the real path to the private key), paste the resulting signature back in, click **Verify** again.
3. Create a repo (**+ → New Repository**).
4. From the repo's Quick Guide (SSH variant):
   ```bash
   touch README.md
   git init
   git checkout -b main
   git add README.md
   git commit -m "first commit"
   git remote add origin gitea@10.0.0.15:<username>/<repo-name>.git
   git push -u origin main
   ```

## Notes

- No default password — the first registered account becomes admin; later accounts need to be promoted.
- Repos here don't need to be private — this is a single-user, self-hosted instance.
- HTTPS + a real DNS name get added in Chapter 3 via `/etc/gitea/app.ini`.
