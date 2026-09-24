# Mirroring a GitHub Repo into Gitea

How to pull a read-only, auto-syncing copy of any GitHub repo into Gitea
(`https://gitea.lab`). Nothing on GitHub changes; GitHub stays the primary
remote and Gitea just keeps a synced backup copy.

## Creating the mirror

1. Sign in at `https://gitea.lab`, click the **+** in the top right, then
   **New Migration**.
2. Select **GitHub** as the source.
3. **Clone Address**: paste the repo's HTTPS URL, e.g.
   `https://github.com/<owner>/<repo>.git`.
4. **Access Token**: leave blank for a public repo. Only needed for a
   private repo, or to avoid GitHub's API rate limit on a very large
   migration. This is a GitHub personal access token, not a Gitea one.
5. **Migration Options**: check **This repository will be a mirror**. This
   is the setting that matters, without it Gitea just does a one-time copy
   instead of an ongoing mirror.
6. Leave **Migrate LFS files** and the Migration Items (Wiki, Labels,
   Issues, etc.) unchecked unless the repo actually uses them and an access
   token is provided.
7. **Owner**: your Gitea account. **Repository Name**: defaults to match
   the source repo, can be changed. **Description**: optional.
8. Click **Migrate Repository**.

Gitea pulls the full repo and history immediately. From then on it's a
mirror: read-only in Gitea, kept in sync from GitHub.

## How syncing works afterward

No manual action needed for normal use. Gitea has a recurring **Update
Mirrors** job (Site Administration > Monitoring > Cron Tasks) that runs
automatically, checking every mirrored repo for new commits on its
schedule (`@every 10m` by default in this instance). Any new commit pushed
to GitHub shows up in Gitea within one cycle of that schedule, no action
needed on your end.

### Forcing an immediate sync

If you don't want to wait for the next scheduled cycle:

- Open the mirrored repo in Gitea, go to **Settings > Mirror Settings**,
  and there's a **Synchronize Now** button.
- Or, as an admin, trigger the **Update Mirrors** cron task directly from
  Site Administration > Monitoring > Cron Tasks.

### Changing the sync interval

Also on the repo's **Settings > Mirror Settings** tab, there's an interval
field (e.g. `8h`, `10m`) that can be set per-repo, independent of the
global cron schedule.

## Troubleshooting

**"The Git data underlying this repository cannot be read."** This means
the migration failed partway through but left an empty repo record behind.
Before retrying:

1. Check Site Administration > Monitoring > Cron Tasks for a specific
   error, though this often won't show much beyond confirming the job ran.
2. From a shell on the Gitea host/LXC, confirm outbound connectivity to
   GitHub works: `curl -I https://github.com` should return `HTTP/2 200`.
3. Check disk space on the Gitea host: `df -h`.
4. If both check out clean, go to Site Administration > Monitoring > Cron
   Tasks and manually run **Delete all repositories missing their Git
   files** (this clears the broken record), or delete the repo manually
   from its own Settings tab.
5. Redo the migration from scratch (New Migration, same settings as
   before). A first attempt sometimes fails on a transient hiccup talking
   to GitHub's API; a clean retry after clearing the broken record usually
   works.

If you see "repository already exists" when trying to migrate again, that
usually means an earlier attempt actually succeeded. Check the repo's file
list and commit history before assuming it's broken.

## Notes

- This is a **one-way, read-only pull**. Gitea never pushes anything back
  to GitHub, and GitHub has no awareness the mirror exists.
- Because it's a scheduled pull rather than an instant push mirror, there's
  a small window (up to the sync interval) where Gitea could be behind the
  latest GitHub commit. For a repo that only changes occasionally (like a
  documentation repo updated once a session), this is a non-issue.
- A commit hash shown in Gitea's UI (e.g. `76f6d6d1e8`) is not sensitive,
  it's just a content-derived identifier and safe to share in screenshots.
