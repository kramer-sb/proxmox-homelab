# Installation Notes: Installing Stalwart (Mail Server)

My own project, outside the JHT course: run mail on my own domain, hosted in the lab. Built 2026-11-01.

**Result:** Stalwart v0.16.27 running in LXC 1004 at `10.0.0.250`. Mail from the internet reaches it and is stored in a mailbox (verified in the server log, see Step 10). Web admin at `http://stalwart.lab:8080/admin`, key-only SSH as `ssh stalwart`. **Update 2026-11-02:** reading mail in a client, a real TLS certificate, sending mail and limiting port 26 were all done afterwards. See [22a-stalwart-certificate-relay-and-clients.md](22a-stalwart-certificate-relay-and-clients.md). This note is kept as it happened on 2026-11-01, so some statements below (for example in Steps 9 and 11) describe the state before 22a.

This repo is public, so real values are replaced. `cyber.example.dev` stands in for my real subdomain, `mail.cyber.example.dev` for the server hostname, and `<ddns-hostname>` for the free dynamic DNS name. My public IP address is not recorded anywhere.

## Step 1: Install the LXC via community script

Command copied from the Stalwart page on community-scripts.org, run in the Proxmox shell, **Advanced Install**.

The page pins **v0.16.27** because Stalwart v1 changes the data store layout and needs an offline migration. **Do not run the container's update until I have read the migration guide.** From a summary of the script's update function, it appears to pull the latest upstream release, which could be v1.x on a 0.16 data store. That reading was not confirmed.

| Setting | Value |
|---|---|
| Container ID | `1004` |
| Hostname | `stalwart` |
| Disk | 40 GB (script default is 20) |
| IPv4 | static `10.0.0.85/24` at first, moved to `10.0.0.250/24` in Step 5. Gateway `10.0.0.1` |
| DNS server | `10.0.0.30` (CoreDNS) |
| DNS search domain | left blank (a bare `lab` was rejected as invalid in the Security Onion wizard, note 16, and I did not try it here) |
| Nesting | left at the app default (enabled) |
| GPU passthrough, APT cacher, HTTP proxy, mknod | no / none / none / off (default) |
| Host hook script | left empty |
| IPv6, container protection, root SSH | answers not recorded (confirm before committing) |

Everything else (2 cores, 4096 MB RAM, Debian 13, unprivileged) is from the script's defaults; I did not confirm those on screen. LXC root password generated and saved in Vaultwarden.

**Gotcha:** the installer's SSH keys screen would not accept the space bar, so I could not tick my key. Fixed after install in Step 2.

## Step 2: Key-only SSH

`ssh.service` was already active after the install, even though I did not turn on the installer's root SSH option.

1. On Windows, made a new ed25519 key named `stalwart` with a passphrase (one key per destination, like `kali-python`). Passphrase is in Vaultwarden.
2. In the container (`pct enter 1004`), wrote the public key to `/root/.ssh/authorized_keys`, `chmod 600`.
   - **Gotcha:** I ran the `echo` command with the placeholder text still in it. Fixed by overwriting with `>` and the real public key line, then `cat` to check.
3. Made SSH key-only with a drop-in, `/etc/ssh/sshd_config.d/10-keys-only.conf`:
   ```
   PasswordAuthentication no
   PermitRootLogin prohibit-password
   ```
   - **Gotcha:** pasting the multi-line heredoc into the Proxmox web console garbled the text (characters dropped, stray `>` prompts). Rewrote the file with a one-line `printf`, then `sshd -t` and a restart of `ssh`.
4. Verified with `sshd -T`: `permitrootlogin without-password` and `passwordauthentication no`.
5. Added a `stalwart` host entry to `~/.ssh/config` from PowerShell (`-Encoding ascii`). `ssh stalwart` asks for the key passphrase and logs in.

This is root with a key, not a non-root user with root login off as on Kali.

## Step 3: First login and setup wizard

1. `journalctl -u stalwart | grep -A8 'bootstrap mode'` shows the temporary login. Account name is `admin`. Password saved to Vaultwarden, not written here.
2. Opened `http://<container-ip>:8080/admin` and ran the 5-step wizard:

| Step | Setting | Value |
|---|---|---|
| 1 Server identity | Server hostname | `mail.cyber.example.dev` |
| 1 | Default email domain | `cyber.example.dev` (no `mail.`) |
| 1 | Automatically obtain TLS certificate | **off** (Let's Encrypt cannot reach this server) |
| 1 | Generate email signing keys (DKIM) | on |
| 2 Storage | Main data storage | RocksDB, path `/opt/stalwart_data/data/` |
| 2 | Blob size, write buffer, block cache | defaults (16.44 KB, 128 MB, 128 MB) |
| 2 | Attachment, search index, cache | "Use data store" for all three |
| 3 Directory | Directory type | internal directory |
| 4 Logging | Destination | log file, path `/opt/stalwart_data/logs/`, prefix `stalwart`, daily rotation, level Info |
| 5 DNS | DNS server type | manual |

**Gotcha:** the wizard's default paths were `/var/lib/stalwart/` and `/var/log/stalwart/`. Neither exists on this container. The script keeps everything in `/opt/stalwart_data` (`etc`, `data`, `logs`), the folder to back up, so I changed both paths to sit inside it.

3. Finishing the wizard issued a new login. **Gotcha:** the account name is the full address, `admin@cyber.example.dev`, not `admin`. The temporary `admin` login stops working once the wizard ends, and I lost time trying it.

**Gotcha:** `/opt/stalwart_data/logs` stayed empty until `systemctl restart stalwart`. The log file only starts after a restart.

## Step 4: Verify

- `systemctl status stalwart` is `active (running)`.
- `/opt/stalwart_data/data` filled with RocksDB files.
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

Ports 587, 143 and 110 do not listen by default.

## Step 5: Move to a gateway-reserved address (`10.0.0.250`)

I needed a port forward from the Xfinity gateway to the container, and that turned out to be the hard part:

- The gateway's admin tool no longer does port forwarding; it says to use the Xfinity app.
- The app's Add Port Forward screen only lists devices the gateway knows about. All my containers have static addresses, so none were listed, only the Proxmox host.
- The admin tool's Connected Devices page can add a device with a reserved IP address, but only inside the DHCP pool (`10.0.0.101` to `10.0.0.253`), not at `10.0.0.85`.

So I made an exception to the numbering pattern: Stalwart gets `10.0.0.250`, reserved on the gateway for its MAC address and set as its static IP in Proxmox.

1. Checked the address was free with `ping 10.0.0.250` (no reply) and the Connected Devices list.
2. Added the reservation in the admin tool (name `stalwart`, MAC from `ip -br link show` inside the container, reserved IP `10.0.0.250`).
3. Changed CT 1004 net0 to `10.0.0.250/24`, gateway `10.0.0.1`, and rebooted it. `ip -br addr` shows `10.0.0.250/24`.
4. CoreDNS: removed the old `stalwart.lab` line from `/etc/coredns/lab.hosts` (`sed -i '/stalwart\.lab/d'`), added `10.0.0.250	stalwart.lab` with `printf`, restarted `coredns.service`, confirmed with `nslookup stalwart.lab 10.0.0.30`.
5. Updated the `HostName` in the Windows `~/.ssh/config`.
6. The gateway then listed `stalwart` as a device, and the app's device list included it.

## Step 6: Uptime Kuma monitors

- Ping, `stalwart.lab`.
- HTTP(S), `http://stalwart.lab:8080`. Plain HTTP, since the admin page is served that way.

Both went red briefly after the reboot and the address change, then green on their own. Note that at Debug log level each HTTP check writes three log lines per minute, which swamps the log.

## Step 7: Backups

Added CT 1004 to all three jobs under Datacenter > Backup: local, cloud (Backblaze via PBS) and USB. Retention unchanged.

## Step 8: Local storage cleanup (same session)

`local` storage was at 80.81% (81.51 of 100.86 GB). Checked each VM's Hardware tab for attached ISOs. Only VM 205 still had them. Set its Windows ISO drive to "Do not use any media", kept `virtio-win`, and removed every other ISO from `local`. Usage dropped to 57.65% (58.15 GB).

## Step 9: Inbound mail path

My Xfinity connection blocks inbound port 25 (reported by Xfinity's own forum; not tested on my line). So mail comes in through a forwarding service on another port.

1. **Forwarding service.** Bought Dynu Email Store/Forward ($9.99 a year) for the subdomain `cyber.example.dev`. The checkout accepted the subdomain, no membership was needed. In its Email Service page: ETRN host = my free dynamic DNS hostname, ETRN port `26`, retry 10 minutes, connection security None, anti-spam on.
2. **Free dynamic DNS name.** Created a free hostname on one of Dynu's domains (`<ddns-hostname>`) and turned off the wildcard aliases and the IPv6 toggle. Installed `ddclient` in the Stalwart container with a config readable only by root (`install -m 600`), using a separate IP update password so my account password is not stored there.
   - **Gotcha:** the first update returned `badauth`. I tested the credentials with `curl -u` and learned the IP update password had not saved. Setting it again fixed it (`nochg`).
   - **Gotcha:** my own test command still contained the placeholder username, which proved nothing.
3. **MX records.** Added two MX records for the subdomain only, at the DNS host where the domain's records already live (host `cyber`, `store1.dynu.com` priority 10 and `store2.dynu.com` priority 20). The records for the main domain were not touched. Verified with `nslookup -type=mx` against my ISP's resolver and `8.8.8.8`. Dynu's page kept showing a warning that the domain is not using its MX records; it did not affect delivery.
4. **Listener on port 26.** Added `[::]:26` as a second bind address on the existing `smtp` listener (Settings > Network > Listeners), then restarted and checked with `ss -tlnp | grep ':26 '`. A forum report says a listener can appear in the UI without binding, so `ss` is the real check.
5. **Authentication rules.** By default Stalwart requires authentication on every port except 25 (`local_port != 25`). Dynu connected to port 26 and was turned away at the sender step (`MAIL FROM without authentication` in the debug log, then QUIT). The same rule offers logins on port 26 to anyone on the internet. Fixed in Settings > MTA > Session > AUTH Stage:
   - **Require Authentication:** `local_port != 25 && local_port != 26`
   - **Allowed Mechanisms:** both IF conditions got `&& local_port != 26` added (`local_port != 25 && local_port != 26 && is_tls`, and `local_port != 25 && local_port != 26`). THEN and ELSE left as they were.
   - **A restart was needed** (`systemctl restart stalwart`) before the change took effect.
   - A Stalwart forum thread says `local_port != 25` is used in about ten places, so port 26 may still behave differently from port 25 in other settings (spam and sender checks). I have not checked which.
6. **Router forward.** In the Xfinity app: Add Port Forward, device `stalwart`, port `26`, protocol TCP only. The gateway cannot limit the source, so port 26 is reachable from the whole internet. See "Not done yet". **Update:** a firewall rule inside the container now limits port 26 to Dynu's published ranges (22a, Step 18).
7. **Log level.** Raised the log tracer to Debug in Settings > Telemetry > Tracers to see why the session stopped, then set it back to Info.

## Step 10: First delivery

- Dynu's "Test ETRN Host" warning ("cannot be reached on port 26") cleared once the forward existed.
- Sent a message from a Gmail account to `admin@cyber.example.dev`. The log showed the full conversation from Dynu's address (a `162.216.242.0/24` address, one of the ranges Dynu lists): EHLO, MAIL FROM, RCPT TO, `DKIM verification passed` for the sender's domain, `Queued message for delivery`, then `Message ingested (message-ingest.ham)`, a `250 OK` DSN and `Delivery completed`.
- Created a normal account, `brie@cyber.example.dev`, under Management > Directory > Accounts (password in Vaultwarden). A second Gmail test was delivered to it, account ID 2, same ingest and delivery lines.

## Step 11: Reading the mail (not working)

Stalwart's web page is an admin console; I found no evidence it shows messages. I tried to add the mailbox to a desktop mail client over IMAP (`stalwart.lab`, port 993, SSL/TLS, normal password). The new account screen said it could not find the server, and its certificate exception dialog could not read the certificate.

What I confirmed: port 993 is reachable from my PC, and from inside the container `openssl s_client` completes a TLS 1.3 handshake with and without a server name, and returns Stalwart's IMAP greeting. The server presents a self-signed certificate (`CN=rcgen self signed cert`). I believe the self-signed certificate is why the client refuses, but I did not confirm that. I did not get to reading the mailbox directly with an IMAP login over `openssl`. **Update:** resolved in 22a, Steps 13 to 16. A real certificate and the right server name made the client work.

## Not done yet (as of the end of 2026-11-01)

Most of this list was done on 2026-11-02, see 22a. Still open after 22a:

- Stalwart's own DKIM keys are not published in DNS, so the extra signatures on outgoing mail cannot be verified (harmless so far).
- Dynu's hold time for mail when the server is unreachable is unknown.
- HTTPS for the admin page (plain HTTP on the LAN now), and a non-root user (root with key-only SSH).
- Reading Stalwart's v1 migration guide before any update.
- A look at how much disk the logs use after a few weeks.
