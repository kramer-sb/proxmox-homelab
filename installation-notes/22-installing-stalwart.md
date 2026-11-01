# Installation Notes: Installing Stalwart (Mail Server)

My own project, outside the JHT course: run mail on my own domain, hosted in the lab. Built 2026-11-01.

**Result:** Stalwart v0.16.27 running in LXC 1004 at `10.0.0.85`, web admin at `http://stalwart.lab:8080/admin`, key-only SSH as `ssh stalwart`. **Mail is not flowing yet.** There are no relays, no public DNS records and no router port forwards, and none should be added until that is planned (see "Not done yet" at the bottom).

This repo is public, so the domain here is a placeholder: `cyber.example.dev` stands in for my real subdomain, and `mail.cyber.example.dev` for the server hostname.

## Step 1: Install the LXC via community script

Command copied from the Stalwart page on community-scripts.org, run in the Proxmox shell. Chose **Advanced Install**, like the other app containers.

The page pins **v0.16.27** because Stalwart v1 changes the data store layout and needs an offline migration. **Do not run the container's update until I have read the migration guide.** From a summary of the script's update function, it appears to pull the latest upstream release, which could be v1.x on a 0.16 data store. That reading was not confirmed.

| Setting | Value |
|---|---|
| Container ID | `1004` |
| Hostname | `stalwart` |
| Disk | 40 GB (script default is 20) |
| IPv4 | static `10.0.0.85/24`, gateway `10.0.0.1` |
| DNS server | `10.0.0.30` (CoreDNS) |
| DNS search domain | left blank (a bare `lab` was rejected as invalid in the Security Onion wizard, see note 16, and I did not try it here) |
| Nesting | left at the app default (enabled) |
| GPU passthrough, APT cacher, HTTP proxy, mknod | no / none / none / off (default) |
| Host hook script | left empty |
| IPv6, container protection, root SSH | answers not recorded (confirm before committing) |

Everything else (2 cores, 4096 MB RAM, Debian 13, unprivileged) is from the script's defaults; I did not change them.

LXC root password generated and saved in my vault.

**Gotcha:** the installer's SSH keys screen would not accept the space bar, so I could not tick my key. Fixed after install in Step 2. Sometimes the screens are behind my proxmox info. Frustrating!

## Step 2: Key-only SSH

`ssh.service` was already active after the install, even though I did not turn on the installer's root SSH option.

1. On Windows, made a new ed25519 key named `stalwart` with a passphrase (one key per destination, like `kali-python`). Passphrase is in Vaultwarden.
2. In the container (`pct enter 1004`), wrote the public key to `/root/.ssh/authorized_keys`, `chmod 600`.
   - **Gotcha:** I ran the `echo` command with the placeholder text still in it, so the file held the placeholder line. Fixed by overwriting with `>` and the real public key line, then `cat` to check.
3. Made SSH key-only with a drop-in, `/etc/ssh/sshd_config.d/10-keys-only.conf`:
   ```
   PasswordAuthentication no
   PermitRootLogin prohibit-password
   ```
   - **Gotcha:** pasting the multi-line heredoc into the Proxmox web console garbled the text (characters dropped, stray `>` prompts). Rewrote the file with a one-line `printf` instead, then ran `sshd -t` and restarted `ssh`.
4. Verified with `sshd -T`: `permitrootlogin without-password` and `passwordauthentication no`.
5. Added a `stalwart` host entry to `~/.ssh/config` from PowerShell (`-Encoding ascii`, as with Kali). `ssh stalwart` asks for the key passphrase and logs in.

This is root with a key, not a non-root user with root login off as on Kali.

## Step 3: First login and setup wizard

1. `journalctl -u stalwart | grep -A8 'bootstrap mode'` shows the temporary login. Account name is `admin`. Password saved to Vaultwarden, not written here.
2. Opened `http://10.0.0.85:8080/admin` and ran the 5-step wizard:

| Step | Setting | Value |
|---|---|---|
| 1 Server identity | Server hostname | `mail.cyber.example.dev` |
| 1 | Default email domain | `cyber.example.dev` (no `mail.`) |
| 1 | Automatically obtain TLS certificate | **off** (Let's Encrypt cannot reach this server yet) |
| 1 | Generate email signing keys (DKIM) | on |
| 2 Storage | Main data storage | RocksDB, path `/opt/stalwart_data/data/` |
| 2 | Blob size, write buffer, block cache | defaults (16.44 KB, 128 MB, 128 MB) |
| 2 | Attachment, search index, cache | "Use data store" for all three |
| 3 Directory | Directory type | internal directory |
| 4 Logging | Destination | log file, path `/opt/stalwart_data/logs/`, prefix `stalwart`, daily rotation, level Info |
| 5 DNS | DNS server type | manual |

**Gotcha:** the wizard's default paths were `/var/lib/stalwart/` and `/var/log/stalwart/`. Neither exists on this container (`ls` returned "No such file or directory"). The script keeps everything in `/opt/stalwart_data` (`etc`, `data`, `logs`), and that is the folder to back up, so I changed both paths to sit inside it.

3. Finishing the wizard showed a new login and password. Both saved to Vaultwarden.

**Gotcha:** `/opt/stalwart_data/logs` stayed empty until I ran `systemctl restart stalwart`. After the restart `stalwart.2026-11-01` appeared. The setting was applied; the log file only starts after a restart.

## Step 4: Verify

- `systemctl status stalwart` is `active (running)`, before and after the restart.
- `/opt/stalwart_data/data` filled with RocksDB files (`CURRENT`, `MANIFEST`, `LOG`, ...) at the time the wizard finished.
- `ss -tlnp | grep stalwart` shows these listening ports, all on every address:

| Port | Service (standard assignment) |
|---|---|
| 25 | SMTP (receiving) |
| 465 | SMTP over TLS (submission) |
| 993 | IMAP over TLS |
| 995 | POP3 over TLS |
| 4190 | ManageSieve |
| 443 | HTTPS |
| 8080 | HTTP web admin |

Ports 587, 143 and 110 are not listening. Nothing is forwarded from the Xfinity gateway, so none of this is reachable from the internet.

## Step 5: DNS name

On the CoreDNS LXC, added `10.0.0.85` and `stalwart.lab` to `/etc/coredns/lab.hosts` (tab-separated), restarted `coredns.service`, and checked with `nslookup stalwart.lab 10.0.0.30`, which answered `10.0.0.85`. `/etc/resolv.conf` on the Stalwart container shows `nameserver 10.0.0.30`. Steps are in [coredns.md](../functional-docs/coredns.md).

## Step 6: Uptime Kuma monitors

- Ping, `stalwart.lab`.
- HTTP(S), `http://stalwart.lab:8080`. Plain HTTP, since the admin page is served that way.

The HTTP monitor showed two red beats early on (88% uptime), as lookups before the CoreDNS entry was live. Seeing red to green is *very* satisfying. 

## Step 7: Backups

Added CT 1004 to all three jobs under Datacenter > Backup: local, cloud (Backblaze via PBS) and USB. Retention unchanged. See [proxmox-backup-server.md](../functional-docs/proxmox-backup-server.md).

## Step 8: Local storage cleanup (same session)

`local` storage was at 80.81% (81.51 of 100.86 GB). Checked each VM's Hardware tab for attached ISOs. Only VM 205 (win11-client) still had them. Set its `ide2` CD drive to "Do not use any media" and left `virtio-win` on `ide0`. Then removed every ISO from `local` except the small `virtio-win` one. Usage dropped to 57.65% (58.15 GB). I wish I had a 2TB drive. 

## Not done yet

- Mail flow. Next decisions: the inbound relay (this connection blocks port 25), the outbound relay, and which DNS records go into my DNS provider for the subdomain.
- Real TLS certificate for the server hostname (ACME is off).
- HTTPS with Caddy for the admin page. It is plain HTTP on the LAN for now.
- Not in `lab.hosts` or the App Table as a `mail.lab` name; the entry is `stalwart.lab`.
- A non-root user. I am using root with key-only SSH.
- Reading the Stalwart v1 migration guide before any update.
