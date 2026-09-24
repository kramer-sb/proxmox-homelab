# 14 - Jelu Book Tracker

Not part of the course. This is the first app added to the lab on my own, using the patterns the course taught: a Docker LXC from the Proxmox helper scripts, a `.lab` name in CoreDNS, and HTTPS through Caddy.

## Goal

Track the physical books I own and the books I want to buy, and use it from my Android phone while shopping.

What I want per book:

| Need | How Jelu covers it |
|---|---|
| Title, author | Book metadata (typed in, or auto-filled from an ISBN or title search) |
| Genre | Tags (tags can also become custom shelves) |
| Notes | "Personal notes" field on the book (only visible to my user) |
| Own it or want it | "Owned" checkbox; unticked plus "To read" works as a wishlist |

Bonus: Jelu can scan an ISBN barcode with the phone camera. That needs HTTPS, which is why Caddy is part of this install and not optional.

## Summary

| Item | Value |
|---|---|
| App | Jelu (https://github.com/bayang/jelu), MIT license |
| Version installed | 0.87.3 |
| Install method | Community script "Docker" LXC, then Docker Compose (Jelu + Caddy) |
| CTID | `1003` |
| Hostname | `jelu` |
| IP | `10.0.0.45` (static) |
| DNS server | `10.0.0.30` (CoreDNS) |
| URL | `https://books.lab` |
| App port | `11111` (internal to Docker only; not published to the LAN) |
| Compose folder | `/root/jelu` |
| Login | Stored in Vaultwarden |

## Step 1: Create the Docker LXC

1. Proxmox web UI (`https://10.0.0.5:8006`) > select the node > **Shell**.
2. Run the community Docker script:

   ```bash
   bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/docker.sh)"
   ```

3. Choose **Advanced** and set:

   | Setting | Value |
   |---|---|
   | OS | Debian (default) |
   | Container type | Unprivileged |
   | Hostname | `jelu` |
   | Container ID | `1003` (next free after 1002) |
   | Disk | 8 GB (default is 4; extra room for book covers) |
   | CPU / RAM | 2 cores / 2048 MB (default) |
   | IPv4 | Static, `10.0.0.45/24` |
   | Gateway | `10.0.0.1` |
   | DNS server | `10.0.0.30` |
   | Install Portainer | No |

   > The DNS server field wants the **existing** CoreDNS address. It is not asking for a new IP in the +5 pattern. Getting this wrong leaves the container with no DNS at all (see `lab-gotchas`, PBS install).

4. Two prompts come up near the end of the script:
   - **APP DEFAULTS: Differences detected.** Choose **Keep Current**. This only controls what gets saved as defaults for the *next* Docker LXC; it does not change this one. "Update Defaults" would pre-fill the next Docker LXC with this hostname and IP, which invites an IP conflict.
   - **Expose Docker TCP socket (insecure)?** Answer `n`. Nothing outside the LXC needs to control Docker, and `a` would let anyone on the network run containers with no password.
5. The script's summary does not show the DNS server, so confirm it (and Start at boot) from the Proxmox host shell:

   ```bash
   pct config 1003 | grep -E "nameserver|onboot"
   ```

   Expected:

   ```
   nameserver: 10.0.0.30
   onboot: 1
   ```

   If either is wrong: `pct set 1003 --nameserver 10.0.0.30` / `pct set 1003 --onboot 1`.

## Step 2: Check the timezone file

Jelu's compose file mounts `/etc/timezone`. Newer Debian images may not have it, and Docker would create an empty folder in its place.

```bash
pct enter 1003
cat /etc/timezone
```

The helper script set the timezone to `America/Indiana/Indianapolis` (Eastern time, same as New York), and the file already existed with that value. No change needed.

If it ever says "No such file", create it:

```bash
echo "America/Indiana/Indianapolis" > /etc/timezone
```

## Step 3: Create the compose folder and files

Still inside the LXC:

```bash
mkdir -p /root/jelu
cd /root/jelu
nano docker-compose.yml
```

Full `docker-compose.yml`:

```yaml
services:
  jelu:
    image: wabayang/jelu
    container_name: jelu
    volumes:
      - ./config:/config
      - ./database:/database
      - ./files/images:/files/images
      - ./files/imports:/files/imports
      - /etc/timezone:/etc/timezone:ro
    expose:
      - "11111"
    restart: unless-stopped

  caddy:
    image: caddy:2
    container_name: jelu-caddy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy-data:/data
      - ./caddy-config:/config
    depends_on:
      - jelu
    restart: unless-stopped
```

Notes:

- `expose` instead of `ports` means Jelu is only reachable by Caddy inside Docker. Same idea as Gitea and Kuma listening on `127.0.0.1` only.
- Caddy talks to Jelu by container name (`jelu`), the same pattern as Vaultwarden.

Save (`Ctrl+O`, `Enter`, `Ctrl+X`), then:

```bash
nano Caddyfile
```

Full `Caddyfile`:

```
books.lab {
    tls internal
    reverse_proxy jelu:11111
}
```

## Step 4: Start it

```bash
docker compose up -d
docker compose ps
```

Both `jelu` and `jelu-caddy` should show as `Up`. In the PORTS column, Jelu shows only `11111/tcp` (no `0.0.0.0`), confirming it is not published to the LAN; Caddy shows `0.0.0.0:80` and `0.0.0.0:443`.

Confirm Jelu finished starting:

```bash
docker compose logs jelu | grep -i "started"
```

Expected (took about 13 seconds):

```
Started JeluApplicationKt in 13.329 seconds (process running for 14.249)
```

> The last lines of the log are two `WARN ... SpringDoc` messages saying the API docs endpoints are enabled by default. They print at the end of startup and are harmless on a private lab network. They are not errors.

## Step 5: Add the DNS record in CoreDNS

1. Leave the Jelu LXC (`exit`), then `pct enter 2000` (CoreDNS LXC).
2. Edit the hosts file:

   ```bash
   nano /etc/coredns/lab.hosts
   ```

3. Add this line at the bottom. The gap must be a real **Tab**, not spaces:

   ```
   10.0.0.45	books.lab
   ```

4. Restart CoreDNS (it does not hot-reload):

   ```bash
   systemctl restart coredns.service
   ```

5. Test from Windows PowerShell (not `nslookup`, because of the NRPT quirk):

   ```powershell
   Resolve-DnsName books.lab
   ```

   Result:

   ```
   Name        Type   TTL   Section    IPAddress
   books.lab   A      3600  Answer     10.0.0.45
   ```

## Step 6: First login from Windows

1. Browse to `https://books.lab`. A certificate warning is expected until Step 7 (Advanced > Continue).
2. Jelu asks for the first user on first launch; that user is the admin.
3. Save the login to Vaultwarden right away.
4. Landing page after login shows "Not currently reading anything". Working.

## Step 7: Trust this Caddy's CA

Every Caddy instance generates its own root CA, so this one needs importing just like Gitea's, Kuma's, and Vaultwarden's.

1. In the Jelu LXC, print the cert:

   ```bash
   pct enter 1003
   cat /root/jelu/caddy-data/caddy/pki/authorities/local/root.crt
   ```

   (Caddy's `/data` folder is mounted from `./caddy-data`, so the CA lives there on the LXC.)

2. Copy everything from `-----BEGIN CERTIFICATE-----` through `-----END CERTIFICATE-----` (including those lines), paste into Notepad, then **File > Save As**, set **Save as type: All files (\*.\*)**, and name it `books-lab-root.crt`. Without "All files", Notepad saves it as `.crt.txt`.
3. **Windows:** double-click the file > **Install Certificate** > **Local Machine** > **Place all certificates in the following store** > **Trusted Root Certification Authorities** (same steps as `functional-docs/trusting-caddy-local-ca-windows.md`). Fully close and reopen the browser. `https://books.lab` then loaded with no warning.
4. **Android:** email the `.crt` to myself (or put it in Google Drive) and download it on the phone. Then install it as a **CA certificate**. Quickest route: Settings > search **CA certificate**. Typical paths:
   - Pixel: Settings > Security & privacy > More security & privacy > Encryption & credentials > Install a certificate > CA certificate
   - Samsung: Settings > Security and privacy > More security settings > Install from device storage > CA certificate

   Tap **Install anyway** on the warning (it shows for any user-installed CA; this one is my own Caddy). Android may ask for the screen lock PIN.
   - Phone model / exact path used: TODO

## Step 8: Test from the phone

1. Turn off Wi-Fi so the test goes over cellular.
2. Turn on Tailscale.
3. Open Chrome and go to `https://books.lab`. It should load with no certificate warning.
4. Log in and add a book with the scanner (see "Adding a book" below). Allow camera access when Chrome asks.
5. Chrome menu (⋮) > **Add to Home screen** so it opens like an app.

Result: page loaded over cellular with no cert warning, scanner read the barcode, and the book was fetched and added.

## Step 8b: Adding an iPhone (kids' devices)

Same idea as Android: the phone needs Tailscale (to reach the lab and resolve `.lab`) and needs to trust this Caddy's CA (for HTTPS and the camera scanner). iOS has one extra trust toggle that is easy to miss.

> Tailscale is needed **even at home on Wi-Fi.** Phones get DNS from the Xfinity gateway, which has never heard of `books.lab`. Only Tailscale's Split DNS sends `.lab` lookups to CoreDNS.

### 1. Install and sign in to Tailscale

1. App Store > install **Tailscale**.
2. Sign in. Two options:
   - **Sign in with my account** (simplest). The phone shows up as one of my devices.
   - **Invite the kid as their own user** from the Tailscale admin console (Users > Invite). This keeps their devices separate, and lets me restrict them with access controls later. Check the free plan's current user limit first.
3. Allow the VPN configuration when iOS asks.
4. Leave **Use Tailscale DNS** on (it is on by default). iOS accepts the subnet route automatically; nothing else to set.

> Screen Time note: if the kid's device has Content & Privacy Restrictions on, "VPN" or profile installs may be blocked. Allow them temporarily from the parent's Screen Time settings.

> Access note: a device on my Tailscale account can reach the **whole** lab subnet (Proxmox UI, Vaultwarden, etc.), not just Jelu. The services are still password protected, but if I want the kids limited to Jelu only, invite them as separate users and add an ACL in the admin console that only allows `10.0.0.45:443`. TODO: decide.

### 2. Get the certificate onto the iPhone

The `.crt` has to be opened by an Apple app so iOS recognizes it as a profile:

- **Email it** as an attachment and open it in the built-in **Mail** app (the Gmail app will not hand it to iOS correctly), or
- **AirDrop** it from another Apple device, or
- Put it in iCloud Drive and tap it in the **Files** app.

Tapping it shows **"Profile Downloaded"**. Choose **Close**.

### 3. Install the profile

1. Settings > **Profile Downloaded** (near the top). If it's not there: Settings > General > **VPN & Device Management** > under Downloaded Profile, tap the Caddy profile.
2. **Install** (top right) > enter the passcode > **Install** > **Install** again > **Done**.

### 4. Turn on full trust (the step people miss)

Installing the profile is not enough. Until this is on, Safari still shows a certificate warning and the scanner will not work.

1. Settings > General > About > scroll to the bottom > **Certificate Trust Settings**.
2. Under "Enable full trust for root certificates", switch on the **Caddy Local Authority** entry.
3. Tap **Continue** on the warning.

### 5. Test and add to the Home Screen

1. Tailscale on. Open **Safari** and go to `https://books.lab`. It should load with no warning.
2. Log in, then add a book with the scanner (see "Adding a book" below). Tap **Allow** when Safari asks for the camera.
3. Share button > **Add to Home Screen** so it opens like an app.

### 6. Give each kid their own Jelu login (optional)

Jelu supports multiple users, and the Owned checkbox, personal notes, and to-read status are per user. Separate logins keep each kid's library and wishlist apart.

- As admin, use the **Users** icon in the left sidebar to create a user for each kid.
- Save each login to Vaultwarden.
- Kids can see each other's libraries in Jelu but not each other's personal notes.

Tested on: TODO (device / iOS version)

## Adding a book (the workflow)

1. Left sidebar > **Add book** (➕ icon).
2. Click **Auto Fill**. This opens the ISBN field with the scan button.
3. Scan the barcode (or type the ISBN).
4. Click **Fetch Book**. Jelu looks up title, author, cover, etc.
5. Review the result. Tick **Owned** for books I have; leave it unticked for the wishlist. Add tags for genre and anything in personal notes.
6. Save.

> **Gotcha:** the yellow barcode button in the **top search bar** also scans, and it will read the digits fine, but it only searches books **already in my library**. It does not look anything up online. On an empty library it finds nothing, which looks like a broken metadata fetch but isn't. Always use **Add book > Auto Fill** to add new books.

> Reminder for shopping trips: Tailscale must be on, or `books.lab` will not resolve or load.

## Step 9: Housekeeping

- **Backups:** check Datacenter > Backup. If the daily job is set to back up specific CTIDs rather than "All", add the new CTID.
- **Uptime Kuma:** add an HTTPS monitor for `https://books.lab`.
- **README:** add a row to the App Table.

## Troubleshooting

| Symptom | Check |
|---|---|
| `books.lab` does not resolve | Tab (not spaces) in `lab.hosts`; `systemctl restart coredns.service`; CoreDNS LXC running |
| Page loads but "Search online" finds nothing | LXC DNS server set to `10.0.0.30`? Fix: `pct set <CTID> --nameserver 10.0.0.30` then `pct reboot <CTID>` |
| 502 from Caddy right after start | Jelu still starting; wait a minute and check `docker compose logs jelu` |
| Camera scan button missing or blocked | Page must be on `https://` with a trusted cert; recheck Step 7 on the phone |
| Scan reads the digits but no book appears | Scanned from the top search bar (searches my library only). Use Add book > Auto Fill > Fetch Book |
| Fetch Book in Auto Fill returns nothing | Built-in Calibre lookup can come back empty. Add Inventaire as a fallback in `/root/jelu/config/application.yml` (see below), then `docker compose restart jelu` |
| Works at home, not away | Tailscale off on the phone |
| `books.lab` won't load on a phone at home on Wi-Fi | Tailscale must be on at home too (Split DNS is what resolves `.lab`) |
| iPhone still shows a cert warning after installing the profile | Turn on full trust: Settings > General > About > Certificate Trust Settings |
| iPhone: tapping the `.crt` does nothing | Open it from the Mail app, Files, or AirDrop, not the Gmail app |

### Optional: fallback metadata source

Not needed so far (Calibre lookup worked), but if fetches start failing, this adds Inventaire (free, no API key) as a backup. Create `/root/jelu/config/application.yml`:

```yaml
jelu:
  metadataProviders:
    - name: "inventaireio"
      is-enabled: true
      order: 200000
      config: "en"
```

Then `docker compose restart jelu`. Google Books can also be added, but it needs a free API key and only searches by ISBN. See https://bayang.github.io/jelu-web/configuration/.
