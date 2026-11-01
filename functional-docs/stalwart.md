# Stalwart (Mail Server)

All-in-one mail server (SMTP, IMAP, POP3, JMAP and a web admin) running in LXC 1004. See `installation-notes/22-installing-stalwart.md` for how it was built, including the gotchas.

**Status:** installed and reachable on the LAN. Mail is not flowing yet; relays, public DNS records and a TLS certificate are still to do.

- **IP:** `10.0.0.85` (static), name `stalwart.lab`
- **Web admin:** `http://stalwart.lab:8080/admin` (plain HTTP, LAN only)
- **SSH:** `ssh stalwart` (root, key-only)
- **Version:** v0.16.27, pinned by the community script. Do not update until I have read the Stalwart v1 migration guide.
- **Data, config and logs:** `/opt/stalwart_data` (`data`, `etc`, `logs`). This is the folder to back up.
- **Credentials:** admin login, LXC root password and SSH key passphrase are in Vaultwarden.

## Ports (all listening, LAN only)

| Port | Service |
|---|---|
| 25 | SMTP |
| 465 | SMTP over TLS (submission) |
| 993 | IMAP over TLS |
| 995 | POP3 over TLS |
| 4190 | ManageSieve |
| 443 | HTTPS |
| 8080 | HTTP web admin |

## Install

1. Copy the command from the Stalwart page on community-scripts.org into the Proxmox shell and choose **Advanced Install**.
2. Settings: ID `1004`, hostname `stalwart`, 40 GB disk, static IP `10.0.0.85/24`, gateway `10.0.0.1`, DNS server `10.0.0.30`, search domain blank. Leave the other prompts at their defaults.
3. Save the LXC root password in Vaultwarden.

## Key-only SSH

1. On Windows (PowerShell), make a key and print the public half:
   ```
   ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\stalwart -C "windows to stalwart"
   type $env:USERPROFILE\.ssh\stalwart.pub
   ```
2. In the container (`pct enter 1004`), replace the placeholder in the first command with the public key line, then run:
   ```
   mkdir -p /root/.ssh && chmod 700 /root/.ssh
   echo "PASTE-PUBLIC-KEY-LINE-HERE" > /root/.ssh/authorized_keys
   chmod 600 /root/.ssh/authorized_keys
   ```
3. Turn off password login. Run each line on its own (multi-line pastes get garbled in the Proxmox console):
   ```
   printf 'PasswordAuthentication no\nPermitRootLogin prohibit-password\n' > /etc/ssh/sshd_config.d/10-keys-only.conf
   sshd -t
   systemctl restart ssh
   sshd -T | grep -Ei "passwordauth|permitroot"
   ```
   Expect `passwordauthentication no` and `permitrootlogin without-password`.
4. Add a host entry on Windows and connect:
   ```
   @"
   Host stalwart
       HostName 10.0.0.85
       User root
       IdentityFile ~/.ssh/stalwart
   "@ | Add-Content -Encoding ascii $env:USERPROFILE\.ssh\config
   ssh stalwart
   ```

## First login and wizard

1. In the container, get the temporary login:
   ```
   journalctl -u stalwart | grep -A8 'bootstrap mode'
   ```
   Account name is `admin`. Save the password in Vaultwarden.
2. Open `http://10.0.0.85:8080/admin`, log in, and run the wizard with these values:
   - Server hostname `mail.cyber.example.dev`, default email domain `cyber.example.dev`, automatic TLS **off**, signing keys on.
   - Storage: RocksDB, path `/opt/stalwart_data/data/`, defaults for the rest, "Use data store" for the other three.
   - Directory: internal.
   - Logging: log file, path `/opt/stalwart_data/logs/`, daily rotation, level Info.
   - DNS: manual.
3. The wizard's default paths (`/var/lib/stalwart/`, `/var/log/stalwart/`) do not exist on this container. Change both as above.
4. Finishing the wizard shows a new login and password. Save them in Vaultwarden.
5. Restart once so the log file starts:
   ```
   systemctl restart stalwart
   ```

## Checks

```
systemctl status stalwart --no-pager
ls -la /opt/stalwart_data/data /opt/stalwart_data/logs
ss -tlnp | grep stalwart
```

## Around the container

- **DNS:** `10.0.0.85` and `stalwart.lab` in `/etc/coredns/lab.hosts` on CoreDNS, then `systemctl restart coredns.service`. See [coredns.md](coredns.md).
- **Uptime Kuma:** a ping monitor on `stalwart.lab` and an HTTP monitor on `http://stalwart.lab:8080`.
- **Backups:** CT 1004 is in the local, cloud and USB jobs. See [proxmox-backup-server.md](proxmox-backup-server.md).

## Before any mail flows

Do not open router ports or add public DNS records until these are decided: the inbound relay (this connection blocks port 25), the outbound relay, and which records belong in the DNS provider for the subdomain.
