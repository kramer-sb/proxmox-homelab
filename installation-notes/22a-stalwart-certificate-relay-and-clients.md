# Installation Notes 22a: Stalwart certificate, outbound relay, port 26 filter and mail clients

*Addendum to [22-installing-stalwart.md](22-installing-stalwart.md). Built 2026-11-02.*

**Result:** the mailbox can be read in Thunderbird on the PC and in Thunderbird for Android on the phone, over a real Let's Encrypt certificate that renews itself. Mail sent from it goes out through SMTP2GO and passes SPF, DKIM and DMARC at Gmail and at a second provider. Inbound mail from Dynu now arrives encrypted, and port 26 only accepts Dynu's published address ranges. This closes most of the "Not done yet" list in note 22.

This repo is public, so real values are replaced. `cyber.example.dev` is my real subdomain, `mail.cyber.example.dev` the server hostname, `example.net` the second domain I bought only to answer certificate checks, `<ddns-hostname>` the free dynamic DNS name, and `emNNNNNNN` and `sNNNNNNN` stand in for the account-specific record names SMTP2GO gave me. No tokens or passwords are recorded here. They are in Vaultwarden.

## Step 12: Start the container and check state

The container was stopped at the start of the session. `pct start 1004` in the Proxmox shell, then in `ssh stalwart`: `systemctl is-active stalwart ddclient` showed both `active`. Dynu's Delivery Queue and Rejected Emails pages were empty, but they never showed anything even during real deliveries, so that said nothing about how long Dynu holds mail.

## Step 13: Prove the mailbox is readable without a client

A direct IMAP login from inside the container, using a hidden prompt so the password never lands in shell history:

```
read -rs -p "mailbox password: " P; echo; (printf 'a1 LOGIN "brie@cyber.example.dev" "%s"\r\n' "$P"; sleep 2; printf 'a2 SELECT INBOX\r\n'; sleep 2; printf 'a3 LOGOUT\r\n'; sleep 1) | openssl s_client -quiet -connect 127.0.0.1:993 2>&1 | grep -E '^a[0-9] |EXISTS'; unset P
```

`a1 OK ... Authentication successful`, `* 1 EXISTS`, `a2 OK`, `a3 OK`.

**Gotcha:** my first version sent all three commands at once. The select ran before the login finished and returned `NO Not authenticated`. Sleeping between commands fixed it.

## Step 14: A real certificate with no inbound ports

Let's Encrypt's HTTP and TLS-ALPN checks need inbound ports 80 or 443, which this server does not have. That leaves the DNS challenge: prove control by publishing a TXT record named `_acme-challenge`.

**Why not give a tool API access to the main DNS zone.** The DNS host's API call for changing records replaces the whole record set at once (from third-party tool documentation, not the host's own page), the zone carries the main domain's mail records and website, and the API needs a whitelisted IPv4 address while my home address can change. A mistake there could break my main mail. So the challenge is delegated to a second domain.

**The second domain.** deSEC's free `dedyn.io` names were suspended (a notice on their site), so I bought `example.net` (about $15 a year at the regular price, no SSL add-on). Its nameservers were set to deSEC's `ns1.desec.io` and `ns2.desec.org` at the registrar, and `nslookup -type=ns example.net 1.1.1.1` returned both. DNSSEC was left off. I considered a second Dynu account and rejected it: Dynu's terms prohibit multiple accounts to get around hostname limits and say all accounts may be terminated, and my mail path runs through Dynu.

**A restricted deSEC token.** deSEC tokens can be limited by policy. In PowerShell, the login and token creation prompted for the account email and password with `Read-Host` (hidden), created a token named `stalwart-acme`, then set two policies on it through the API:

```
{"domain": null, "subname": null, "type": null, "perm_write": false}
{"domain": "example.net", "subname": "_acme-challenge", "type": "TXT", "perm_write": true}
```

The default policy blocks all writes and the second allows only TXT records named `_acme-challenge` on that one domain. The token went into Vaultwarden. **Gotcha:** paste the two `Read-Host` lines on their own. Pasting everything at once lets PowerShell use the next pasted line as the answer.

**One CNAME at the main DNS host** (the only change to that zone for this step), so Let's Encrypt follows the challenge to the second domain:

| Type | Host | Value |
|---|---|---|
| CNAME | `_acme-challenge.mail.cyber` | `_acme-challenge.example.net` |

Checked with `nslookup -type=cname _acme-challenge.mail.cyber.example.dev 1.1.1.1`.

**acme.sh in the container.**

```
DEBIAN_FRONTEND=noninteractive apt install -y curl git cron
git clone --depth 1 https://github.com/acmesh-official/acme.sh.git /root/acme.sh-src
cd /root/acme.sh-src && ./acme.sh --install
/root/.acme.sh/acme.sh --set-default-ca --server letsencrypt
read -rs -p "deSEC token: " DEDYN_TOKEN; echo; export DEDYN_TOKEN
/root/.acme.sh/acme.sh --issue --server letsencrypt_test --dns dns_desec -d mail.cyber.example.dev --challenge-alias example.net
```

`--set-default-ca` matters because acme.sh defaults to ZeroSSL. The test run uses Let's Encrypt's staging server. acme.sh's own public DNS pre-check printed "Not valid yet" for a while and then the run completed. The acme.sh deSEC plugin's docs only show a `dedyn.io` example, but it worked with an own domain and the restricted token. The production certificate came from the same command with `--server letsencrypt` and `--force` (needed because acme.sh thought it already had this certificate from the staging run).

**Install the files where Stalwart can read them.** Stalwart runs as root in this container (`systemctl show stalwart -p User` is `root`).

```
mkdir -p /opt/stalwart_data/etc/tls && chmod 700 /opt/stalwart_data/etc/tls
/root/.acme.sh/acme.sh --install-cert -d mail.cyber.example.dev --fullchain-file /opt/stalwart_data/etc/tls/fullchain.pem --key-file /opt/stalwart_data/etc/tls/privkey.pem --reloadcmd "systemctl restart stalwart"
chmod 600 /opt/stalwart_data/etc/tls/privkey.pem
openssl x509 -in /opt/stalwart_data/etc/tls/fullchain.pem -noout -issuer -subject -dates
```

The issuer named Let's Encrypt with no `STAGING`. acme.sh issued an ECC certificate by default.

**Tell Stalwart about it.** Settings > TLS > Certificates > Create certificate. Each of the two fields has a dropdown (default "Text value" and "Secret value"). I chose the file option for both and entered `/opt/stalwart_data/etc/tls/fullchain.pem` and `/opt/stalwart_data/etc/tls/privkey.pem`.

**Gotcha:** the server kept presenting its old self-signed certificate until `systemctl restart stalwart`. Stalwart's docs say a new or changed certificate takes effect only on a reload or restart. The check:

```
echo | openssl s_client -connect 127.0.0.1:993 -servername mail.cyber.example.dev -verify_hostname mail.cyber.example.dev -brief 2>&1 | head -n 12
```

showed `Verification: OK` and `Verified peername: mail.cyber.example.dev`.

**Renewal.** acme.sh's cron job runs four times a day. I chose a service restart as the reload because it needs no API token or CLI, at the cost of a brief drop when a renewal happens. `acme.sh --info -d mail.cyber.example.dev --ecc` shows the recorded reload command and the next renewal (picked from the CA's suggested window, about 2027-01-01). I tested the whole path with `acme.sh --renew -d mail.cyber.example.dev --ecc --force`: it renewed, installed the files, restarted Stalwart, and the served certificate then had a later start date.

## Step 15: Making the new name resolve

Clients must connect as `mail.cyber.example.dev` for the certificate to match, but nothing resolved that name inside the lab.

- **CoreDNS.** My Corefile applies `lab.hosts` only to the `lab` zone and forwards everything else, so a line in `lab.hosts` for a non-`.lab` name is silently ignored. I appended a server block for just that name (after `cp Corefile Corefile.bak-2026-11-02`):

```
mail.cyber.example.dev {
    hosts {
        10.0.0.250 mail.cyber.example.dev
    }
    log
    errors
}
```

Restarted `coredns.service`. From the PC: `nslookup mail.cyber.example.dev 10.0.0.30` returned `10.0.0.250`, `nslookup example.dev 10.0.0.30` still returned the public record, and `nslookup gitea.lab 10.0.0.30` still worked.
- **Windows PC.** The PC deliberately does not use CoreDNS (note 07a), so I added `10.0.0.250 mail.cyber.example.dev` to its hosts file instead. Confirmed with `ping -n 1 mail.cyber.example.dev` (`nslookup` ignores the hosts file).
- **Phone.** A second Split DNS entry in the Tailscale admin console: nameserver `10.0.0.30`, "Restrict to domain" on, domain `mail.cyber.example.dev`. The existing `lab` entry was left alone.

## Step 16: Thunderbird on the PC

New account with the server name `mail.cyber.example.dev`. Thunderbird found the settings itself (outgoing: SMTP port 465, SSL/TLS, normal password) and asked for the mailbox password. It also offered to sync the account's calendar and address book, which I accepted. The inbox showed the message received earlier. This supports the self-signed certificate having been the cause of yesterday's client failure, though I did not test that in isolation.

## Step 17: Outbound relay through SMTP2GO

My Xfinity connection blocks outbound port 25 too. The first send to Gmail connected to Google's mail server on port 25 and timed out after about two minutes (`delivery.connect-error` in the log).

**SMTP2GO account (free plan).** Sending > Verified Senders > Sender Domains, added `cyber.example.dev` (not the main domain). I chose manual DNS setup, because the automatic option would let the provider write to my DNS zone. It listed three CNAMEs, all under the `cyber` subdomain, which I added at the DNS host:

| Host | Value |
|---|---|
| `emNNNNNNN.cyber` | `return.smtp2go.net` (return path, makes SPF align) |
| `sNNNNNNN._domainkey.cyber` | `dkim.smtp2go.net` (DKIM) |
| `link.cyber` | `track.smtp2go.net` (tracking link domain) |

Each was checked with `nslookup -type=cname ... 1.1.1.1`, then Verify in the dashboard succeeded. An SMTP user was created under Sending > SMTP Users, with the login in Vaultwarden.

**In Stalwart.** Settings > MTA > Outbound > Routes > Create route:

| Field | Value |
|---|---|
| Route type | Relay Host |
| Address | `mail.smtp2go.com` |
| Port | `465` |
| Protocol | SMTP |
| Implicit TLS | on |
| Allow Invalid Certs | off |
| Authentication | the SMTP user's username and password (Secret set to the plain value option) |
| Name | `smtp2go` |

Then Settings > MTA > Outbound > Strategy (Outbound Delivery Strategy), Routing section: left the `is_local_domain(rcpt_domain)` to `'local'` rule alone and changed the ELSE value from `'mx'` to `'smtp2go'`. The Scheduling section's ELSE `'remote'` is a different setting and stays.

**Gotcha:** the strategy change did not take effect until `systemctl restart stalwart`. Until then the log still showed attempts to Gmail on port 25. After the restart both queued messages went through `mail.smtp2go.com` on port 465 with a `250` reply. A forum report on a nearby version describes a relay route being ignored when its name did not exactly match the strategy's value, so keep the two identical.

**Tests.**
- A message to myself (the same mailbox) used the local route and says nothing about the relay.
- A message to another provider arrived with `dmarc=pass`, `spf=pass` (return path under the subdomain) and `dkim=pass` for the subdomain.
- Two messages to Gmail were accepted by SMTP2GO but showed only "Processed" for about fifteen minutes, then arrived with SPF, DKIM and DMARC all `pass`. A later message to Gmail arrived quickly, so the delay looks like a one-off.
- Stalwart also signs with its own DKIM keys (selectors in the form `v1-ed25519-DATE` and `v1-rsa-DATE`). Those public keys are not published in DNS because DNS is in manual mode, so receivers can only verify SMTP2GO's signature. It passes, which is what DMARC needed. I have not decided whether to publish the keys or stop the extra signing.

## Step 18: Hardening port 26

**Open relay check.** From inside the container, an unauthenticated session to port 26 with `RCPT TO` an outside address was refused: `550 5.1.2 Relay not allowed`. That test ran from the server itself, not from the internet.

```
bash -c 'exec 3<>/dev/tcp/127.0.0.1/26; sleep 1; printf "EHLO test.local\r\nMAIL FROM:<a@example.com>\r\nRCPT TO:<x@gmail.com>\r\nQUIT\r\n" >&3; sleep 2; cat <&3'
```

**TLS on port 26.** `openssl s_client -starttls smtp -connect 127.0.0.1:26 -servername mail.cyber.example.dev -brief` showed TLS 1.3 and `Verification: OK`. Port 26 advertises STARTTLS and no login.

**Who actually connects.** Searching the logs for `remoteIp` found nothing at first. **Gotcha:** the log files contain terminal color codes between field names and values, which break the pattern. Strip them first:

```
sed 's/\x1b\[[0-9;]*m//g' /opt/stalwart_data/logs/stalwart.2026-11-0* | grep -o 'remoteIp = [0-9a-f:.]*' | grep -v '10\.0\.0\.' | sort | uniq -c | sort -rn
```

All of Dynu's connections (about 37) came from `162.216.242.0/24`, one of the ranges Dynu publishes for whitelisting, so the gateway passes the real sender address through. The other outside addresses were SMTP2GO, Google's servers, and tests from the server. At Info log level, refused connections may not be logged, so this cannot show whether anything probed the port.

**Firewall in the container.** The container is unprivileged, but `nft` is installed and a harmless test (`nft add table inet testtbl`, then delete it) showed rules can be changed. The container had Debian's default `/etc/nftables.conf` (empty chains, loaded at boot by `nftables.service`, with `flush ruleset` at the top, so anything not in that file is lost on reload). I backed it up to `/etc/nftables.conf.bak-2026-11-02`, tried the rules live first, and then made them permanent by writing this file:

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
        chain input {
                type filter hook input priority filter;
                tcp dport 26 iifname "lo" accept
                tcp dport 26 ip saddr { 162.216.242.0/24, 72.51.58.0/24, 142.202.188.16/29, 107.161.27.201, 107.161.24.148 } accept
                tcp dport 26 drop
        }
        chain forward {
                type filter hook forward priority filter;
        }
        chain output {
                type filter hook output priority filter;
        }
}
```

Checked with `nft -c -f /etc/nftables.conf` before loading. The rules are still present after a container reboot. From the PC, before the rules `Test-NetConnection 10.0.0.250 -Port 26` succeeded, after them it failed while port 993 and ping still worked. The ranges come from Dynu's FAQ pages, which differ slightly between copies. If Dynu ever delivers from an address outside this list, its mail is refused and Dynu keeps retrying. I do not know how long it holds mail. Undo with `nft flush chain inet filter input` (live) or by restoring the backup file.

## Step 19: Gateway shows the container as offline

After the restart the gateway listed the reserved `stalwart` entry as offline and the Xfinity app did not offer it when adding the port forward again. The MAC address matched, pings from the container did not help, and neither did a gratuitous ARP (`arping -U -I eth0 -c 3 10.0.0.250`). After a container reboot, a new generic entry named for the network card's vendor appeared, and the `stalwart` device then showed as online in the app and could be selected. The cause is unknown. I considered switching the container to DHCP (the reservation would have given the same address) and did not need to.

## Step 20: Encrypted delivery from Dynu

Dynu's Connection Security dropdown has four choices: None, SSL/TLS, STARTTLS (Optional) and STARTTLS (Required). Port 26 speaks STARTTLS, not implicit TLS, so SSL/TLS would fail. Required would make Dynu hold mail if encryption ever failed (for example during the restart after a renewal). I chose **STARTTLS (Optional)**.

Stalwart does not add a `Received:` line of its own, so the evidence was Stalwart's spam result at the top of the message source (Thunderbird, Ctrl+U). Before the change it listed `RCVD_NO_TLS_LAST`. A new test from Gmail afterwards listed `RCVD_TLS_LAST`. I do not know whether Dynu checks the certificate name (it connects by `<ddns-hostname>`, not `mail.cyber.example.dev`). The delivery worked, so at least it did not refuse the connection.

## Step 21: Thunderbird for Android

**Gotcha:** the first test of `https://mail.cyber.example.dev` on the phone said "DNS address could not be found" because the Tailscale app was not connected. With it connected (and "Use Tailscale subnets" and "Use Tailscale DNS settings" on), the page loaded Stalwart's account login with no certificate warning. That page is for managing the mailbox, not reading mail.

Installed Thunderbird for Android and set the account up by hand: IMAP `mail.cyber.example.dev` port 993 SSL/TLS, SMTP the same host port 465 SSL/TLS, the full address as the username, the mailbox password. "Autodetect IMAP namespace" left on. The inbox messages appeared. There was no Archive folder, so I created one from the PC and picked it in the app. The check interval is Never (manual refresh), so there are no new-mail alerts. Background push was not tried. Phone mail only works while Tailscale is connected.

## Not done yet

- Stalwart's own DKIM keys are unpublished and still add unverifiable signatures to outgoing mail.
- Webmail in a browser (Bulwark) was set aside. It would need a host to run on.
- Dynu's hold time for mail when the server is unreachable is still unknown, and so is whether other Stalwart settings treat port 26 differently from port 25.
- The admin page is still plain HTTP on the LAN, root with key-only SSH, and Stalwart v1's migration guide is unread.
- Whether new-mail alerts on the phone are wanted.
