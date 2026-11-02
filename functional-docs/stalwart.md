# Stalwart (Mail Server)

All-in-one mail server (SMTP, IMAP, POP3, JMAP and a web admin) in LXC 1004. See `installation-notes/22-installing-stalwart.md` for how it was built and `installation-notes/22a-stalwart-certificate-relay-and-clients.md` for the certificate, relay, firewall and mail clients, including the gotchas.

**Status:** receives mail from the internet through a forwarding service, stores it in mailboxes, and sends through an outbound relay. Mail is read in Thunderbird on the PC and in Thunderbird for Android on the phone, over a real Let's Encrypt certificate that renews itself.

- **IP:** `10.0.0.250` (static, and reserved on the gateway so a port forward can target it), name `stalwart.lab`
- **Web admin:** `http://stalwart.lab:8080/admin` (plain HTTP, LAN only). The account name is the full address, `admin@cyber.example.dev`.
- **SSH:** `ssh stalwart` (root, key-only)
- **Version:** v0.16.27, pinned by the community script. Do not update until I have read the Stalwart v1 migration guide.
- **Data, config and logs:** `/opt/stalwart_data` (`data`, `etc`, `logs`). This is the folder to back up.
- **Credentials:** admin login, mailbox password, LXC root password, SSH key passphrase and the dynamic DNS update password are all in Vaultwarden.

## Why this design

Mail runs on a subdomain, with the main address staying at its current provider. Inbound mail arrives through a forwarding service and outbound mail leaves through a relay, because of the ISP and the home IP address:

- **Port 25 is blocked.** An Xfinity employee says on their community forum that Xfinity blocks port 25 for residential email traffic ([Xfinity: email port 25 no longer supported](https://www.xfinity.com/support/articles/email-port-25-no-longer-supported)). I did not test inbound on my line, but a direct outbound delivery to Gmail on port 25 timed out here, which is consistent. So inbound mail comes in on port 26 from a forwarding service, and outbound mail goes through a relay.
- **A home IP is distrusted.** Receiving servers distrust residential addresses even with correct SPF, DKIM and DMARC, and a residential line cannot set a PTR (reverse DNS) record.
- **Receiving is the forgiving direction.** Mail servers retry, so a short outage costs little. Sending is what depends on reputation, which is why it goes through a relay.
- **Upkeep.** Watch blocklists and uptime. Upgrades can be disruptive: one community report says moving from Stalwart 0.15 to 0.16 needed a fresh install, so check the migration notes before updating.
- **Treat it as secondary** until delivery has proven reliable over time.

## Ports

| Port | Service | Reachable from |
|---|---|---|
| 26 | SMTP, receives forwarded mail (STARTTLS) | the internet, through a router forward, but a firewall in the container only accepts Dynu's published address ranges |
| 25 | SMTP | LAN only (nothing forwarded) |
| 465 | SMTP over TLS (submission) | LAN |
| 993 | IMAP over TLS | LAN |
| 995 | POP3 over TLS | LAN |
| 4190 | ManageSieve | LAN |
| 443 | HTTPS | LAN |
| 8080 | HTTP web admin | LAN |

## Install

1. Copy the command from the Stalwart page on community-scripts.org into the Proxmox shell and choose **Advanced Install**.
2. Settings: ID `1004`, hostname `stalwart`, 40 GB disk, DNS server `10.0.0.30`, search domain blank. Leave the other prompts at their defaults. Network: see "Address" below.
3. Save the LXC root password in Vaultwarden.

## Address

The Xfinity app can only forward a port to a device the gateway knows, and it only learns reserved or DHCP devices. Reservations are only accepted inside the DHCP pool (`10.0.0.101` to `10.0.0.253`). So Stalwart is an exception to the numbering pattern:

1. In the gateway admin tool, Connected Devices, add a device with a reserved IP: name `stalwart`, the container's MAC (`ip -br link show` inside it), reserved IP `10.0.0.250`.
2. In Proxmox, set CT 1004 net0 to `10.0.0.250/24`, gateway `10.0.0.1`, and reboot it.
3. Add `10.0.0.250` and `stalwart.lab` to `/etc/coredns/lab.hosts` on CoreDNS, restart `coredns.service`, and check `nslookup stalwart.lab 10.0.0.30`.

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
       HostName 10.0.0.250
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
2. Open the admin page, log in, and run the wizard:
   - Server hostname `mail.cyber.example.dev`, default email domain `cyber.example.dev`, automatic TLS **off**, signing keys on.
   - Storage: RocksDB, path `/opt/stalwart_data/data/`, defaults for the rest, "Use data store" for the other three.
   - Directory: internal.
   - Logging: log file, path `/opt/stalwart_data/logs/`, daily rotation, level Info.
   - DNS: manual.
3. The wizard's default paths (`/var/lib/stalwart/`, `/var/log/stalwart/`) do not exist on this container. Change both as above.
4. Finishing the wizard issues a new login and password. **The account name is now the full address** (`admin@cyber.example.dev`). Save both in Vaultwarden.
5. Restart once so the log file starts: `systemctl restart stalwart`.

## Inbound mail through a forwarding service

My connection blocks inbound port 25, so mail arrives through a store-and-forward service that delivers to port 26.

1. **Service.** Dynu Email Store/Forward for the subdomain. In its Email Service page: ETRN host = the dynamic DNS hostname, ETRN port `26`, retry 10, connection security None, anti-spam on.
2. **Dynamic DNS name and updater.** Create a free hostname in Dynu's control panel and turn off the wildcard and IPv6 toggles. Create an IP update password, separate from the account password, and save it in Vaultwarden. In the container:
   ```
   DEBIAN_FRONTEND=noninteractive apt install -y ddclient
   install -m 600 /dev/null /etc/ddclient.conf
   nano /etc/ddclient.conf
   ```
   Config (replace the placeholders):
   ```
   daemon=60
   syslog=yes
   pid=/var/run/ddclient.pid
   use=web, web=checkip.dynu.com/, web-skip='IP Address'
   server=api.dynu.com
   protocol=dyndns2
   login=YOUR-DYNU-USERNAME
   password='YOUR-IP-UPDATE-PASSWORD'
   YOUR-HOSTNAME
   ```
   Test the credentials before trusting them: `read -r -p "Dynu username: " U`, `read -rs -p "IP update password: " P; echo`, then `curl -s -u "$U:$P" "https://api.dynu.com/nic/update?hostname=YOUR-HOSTNAME"; echo; unset U P`. A reply of `good` or `nochg` is a pass; `badauth` means the password or username is wrong.
   Then `systemctl restart ddclient` and `systemctl status ddclient --no-pager`.
3. **MX records.** At the domain's DNS host, add two MX records for the subdomain only: host `cyber`, `store1.dynu.com` priority 10 and `store2.dynu.com` priority 20. Do not touch the main domain's records. Check with `nslookup -type=mx cyber.example.dev`.
4. **Listener on port 26.** Admin page, Settings > Network > Listeners, open the `smtp` listener and add `[::]:26` to its bind addresses (keep `[::]:25`). Save, run `systemctl restart stalwart`, and confirm with `ss -tlnp | grep -E ':(25|26) '`.
5. **Authentication rules.** Settings > MTA > Session > AUTH Stage:
   - Require Authentication: `local_port != 25 && local_port != 26`
   - Allowed Mechanisms: add `&& local_port != 26` to both IF conditions, leave THEN and ELSE alone.
   - Save and **restart the service**, or the old rule stays in force.
6. **Router forward.** In the Xfinity app: Add Port Forward, device `stalwart`, port `26`, protocol TCP only. This is open to the whole internet; the gateway cannot limit the source. A firewall in the container now limits it (see "Port 26 filter").
7. **Mailbox.** Management > Directory > Accounts: create `brie`, domain `cyber.example.dev`, a password from Vaultwarden, default groups and roles.

## Checks

```
systemctl status stalwart --no-pager
ls -la /opt/stalwart_data/data /opt/stalwart_data/logs
ss -tlnp | grep stalwart
grep -E 'message-ingest|delivery.completed|queue.message-queued' /opt/stalwart_data/logs/stalwart.$(date -u +%Y-%m-%d) | tail -n 10
```

A good delivery shows `Queued message for delivery`, `Message ingested (message-ingest.ham)` and `Delivery completed`. If Dynu connects but the log stops at `MAIL FROM without authentication` (visible at Debug level), the authentication rules in step 5 are not in force.

## Certificate

A Let's Encrypt certificate for `mail.cyber.example.dev`, issued by `acme.sh` in the container with the DNS challenge, because there are no inbound ports 80 or 443.

- The challenge is delegated: a CNAME at the main DNS host, `_acme-challenge.mail.cyber` pointing at `_acme-challenge.example.net`. `example.net` is a second domain I own, hosted at deSEC, used for nothing else. The deSEC token can only write TXT records named `_acme-challenge` on that domain. The token is in Vaultwarden and in a root-only acme.sh file in the container.
- Files: `/opt/stalwart_data/etc/tls/fullchain.pem` and `privkey.pem` (mode 600). Stalwart runs as root.
- In Stalwart: Settings > TLS > Certificates, one certificate that points at those two file paths.
- Renewal: acme.sh's cron job (four times a day), with reload command `systemctl restart stalwart`. Next renewal is picked from the CA's suggested window, about 60 days after issue. A new or changed certificate only takes effect after a restart. Check what is served:

```
echo | openssl s_client -connect 127.0.0.1:993 -servername mail.cyber.example.dev -verify_hostname mail.cyber.example.dev -brief 2>&1 | head -n 12
acme.sh --info -d mail.cyber.example.dev --ecc | grep -E 'Le_ReloadCmd|Le_NextRenewTimeStr'
```

## Outbound mail

Everything not for a local domain goes through SMTP2GO (free plan), because the ISP blocks outbound port 25.

- Stalwart: Settings > MTA > Outbound > Routes has a route named `smtp2go` (Relay Host, `mail.smtp2go.com`, port 465, implicit TLS on, the SMTP user's login). In the Outbound Delivery Strategy, the Routing rule is `is_local_domain(rcpt_domain)` to `'local'`, ELSE `'smtp2go'`. The route name and the strategy's value must match exactly. Changing the strategy needs a restart of the service.
- DNS: the sender domain `cyber.example.dev` is verified at SMTP2GO with three CNAMEs under the `cyber` subdomain (return path, DKIM, tracking link). No SPF or DMARC record was added; the main domain's DMARC policy is inherited and tests passed.
- Verified: a message to another provider and a message to Gmail both showed `spf=pass`, `dkim=pass` and `dmarc=pass`. The first Gmail messages were delayed about fifteen minutes.
- Stalwart also signs with its own DKIM keys, which are not published in DNS (manual DNS mode). Receivers ignore those signatures; SMTP2GO's signature is the one that passes.

## Port 26 filter

A firewall inside the container (nftables, in `/etc/nftables.conf`, loaded at boot) accepts TCP port 26 only from loopback and from Dynu's published ranges (`162.216.242.0/24`, `72.51.58.0/24`, `142.202.188.16/29`, `107.161.27.201`, `107.161.24.148`) and drops the rest. The ranges come from Dynu's FAQ pages, which differ slightly between copies. If Dynu delivers from an address outside the list, its mail is refused and retried. Undo live with `nft flush chain inet filter input`. The original file is backed up as `/etc/nftables.conf.bak-2026-11-02`.

Port 26 offers STARTTLS with the real certificate and no login. Dynu's Connection Security is set to STARTTLS (Optional). To check that a delivery was encrypted, look in the message source for `RCVD_TLS_LAST` in the spam result (Stalwart adds no `Received:` line of its own).

## Reading mail

Use the real hostname `mail.cyber.example.dev`, or the certificate will not match.

- **Thunderbird on the PC:** settings are found automatically (IMAP 993 and SMTP 465, both SSL/TLS, full address as the username). The PC does not use CoreDNS, so it has `10.0.0.250 mail.cyber.example.dev` in its hosts file.
- **Thunderbird for Android:** IMAP 993 and SMTP 465, both SSL/TLS, full address as the username. The Tailscale app must be connected: a Split DNS entry in the Tailscale admin console (nameserver `10.0.0.30`, restricted to `mail.cyber.example.dev`) is what makes the name resolve on the phone. The check interval is Never (manual refresh), so there are no new-mail alerts.
- CoreDNS has a server block for that one name (see `coredns.md`).
- The web page at `https://mail.cyber.example.dev` is Stalwart's account management page, not a mailbox reader.


## Around the container

- **DNS:** see "Address" above, and [coredns.md](coredns.md).
- **Uptime Kuma:** a ping monitor on `stalwart.lab` and an HTTP monitor on `http://stalwart.lab:8080`.
- **Backups:** CT 1004 is in the local, cloud and USB jobs. See [proxmox-backup-server.md](proxmox-backup-server.md).

## Open items

- Stalwart's own DKIM keys are not published in DNS, so the extra signatures on outgoing mail cannot be verified (harmless so far).
- Dynu's hold time for mail when the server is unreachable is unknown. Other Stalwart settings may treat port 26 differently from port 25 (not checked).
- No new-mail alerts on the phone, and no webmail in a browser (Bulwark was set aside; it would need a host).
- The admin page is plain HTTP on the LAN, and SSH is root with a key.
