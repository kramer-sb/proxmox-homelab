# 05 - Static IP Networking (In Progress)

Status: **In progress** — covers Course Section 3.3 (Static IP Addresses). Uptime Kuma and Kali still need static IPs before this section is complete.

## Goal

Move core home lab services off DHCP and onto static IPs, so their addresses never drift and DNS entries (Section 3.4) can point to them reliably.

## Network Baseline

- ISP/equipment: Xfinity-provided gateway (router mode)
- LAN subnet: `10.0.0.0/24`
- Default gateway: `10.0.0.1`

## Step 1: Reserved Static IP Range on the Xfinity Gateway

To avoid static/DHCP conflicts, the DHCP pool was narrowed to leave room for static assignments.

- Logged into the Xfinity Admin Tool at `http://10.0.0.1` (required enabling **Admin Tool Online Access** first via the Xfinity app: WiFi tab → View WiFi Equipment → Advanced Settings)
- Navigated to **Gateway → Connection → Local IP Network**
- Changed **DHCP Beginning Address** from `10.0.0.2` to `10.0.0.101`
- Left **DHCP Ending Address** at `10.0.0.253`
- Result: `10.0.0.2` – `10.0.0.100` is now reserved and free for static IP use

## Step 2: Static IP Assignment Plan

| Machine | Type | Static IP | Status |
|---|---|---|---|
| Gitea | LXC | `10.0.0.10` | ✅ Done |
| Vaultwarden | LXC | `10.0.0.15` | ✅ Done |
| Uptime Kuma | LXC | `10.0.0.20` | ⬜ Not started |
| Kali Linux | VM | `10.0.0.25` | ⬜ Not started |

Assignment pattern: increments of 5, starting at `10.0.0.10`, leaving room to insert more machines later without renumbering.

## Step 3: Setting Static IPs on Proxmox LXCs

For each LXC (process used for Gitea and Vaultwarden):

1. Shut down the LXC in Proxmox
2. Select the LXC → **Network** tab → select `net0` → **Edit**
3. Click **Static** (deselecting DHCP)
4. Set **IPv4/CIDR** to the assigned IP with `/24` (e.g. `10.0.0.10/24`)
5. Set **Gateway** to `10.0.0.1`
6. Click **OK**, then power the LXC back on
7. Verify inside the LXC console: `ip a`, `ping 8.8.8.8`, `ping google.com`

## Gitea — Completed

- Static IP set: `10.0.0.10`
- Verified network connectivity via `ip a` / ping tests
- Web UI confirmed reachable at `http://10.0.0.10:3000` (Gitea's default port — must be included, plain IP alone will not load it)

### Troubleshooting note: LXC console login rejected

Root password was rejected at the Proxmox Console login prompt despite being correct. Root cause was very likely the noVNC console's keyboard layout mismapping typed characters. **Workaround used** (bypasses console typing entirely) — run from the **Proxmox host shell**, not the LXC console:

```
pct enter <CTID>
passwd
```

This drops directly into the container as root (no password needed) and lets you set a fresh password. Useful as a general fallback any time an LXC console login seems to reject a password you're sure is correct.

## Vaultwarden — Completed

- Static IP set: `10.0.0.15`
- Discovered this instance runs as **Docker Compose**, not a native systemd service:
  - Compose file location: `/root/vaultwarden/docker-compose.yml`
  - Two containers: `vaultwarden` (app) and a `caddy` reverse proxy handling ports 80/443
  - Vault data persisted in `/root/vaultwarden/vw-data/` (bind mount, survives container recreation)
- Updated the Windows hosts file entry for `passwords.local` from the old DHCP address (`10.0.0.83`) to the new static IP (`10.0.0.15`)

### Troubleshooting note: SSL error on bare IP

Browsing directly to `https://10.0.0.15` threw `ERR_SSL_PROTOCOL_ERROR`. Cause: Caddy is configured with `DOMAIN: "https://passwords.local"` in the compose file, and only serves valid HTTPS for that hostname — not for a raw IP. **Fix:** always access via `https://passwords.local` (requires the hosts file entry above to point at the current IP).

### Troubleshooting note: Admin panel was disabled

`/admin` reported no `ADMIN_TOKEN` configured. Enabled it by editing `/root/vaultwarden/docker-compose.yml`, adding a new line under the `vaultwarden` service's `environment:` block, directly below `DOMAIN`:

```yaml
environment:
  DOMAIN: "https://passwords.local"
  ADMIN_TOKEN: "<REDACTED — stored in password manager>"
```

Applied with:

```
cd /root/vaultwarden && docker compose up -d
```

This recreates only the `vaultwarden` container with the new setting; the `vw-data` volume mount means existing vault data is untouched by the recreate.

### Troubleshooting note: Login rejected

Login at `https://passwords.local` kept rejecting credentials. Root cause: the Vaultwarden account was originally registered under `brie@homelab.local`, a dedicated address for this instance — not the personal email that was being tried. Confirmed via the newly-enabled admin panel's Users list. Once the correct email + saved master password were used, login succeeded.

**Takeaway:** document the exact registered account email for self-hosted services separately from personal credentials, since it's easy to assume the "obvious" email was used.

## What's Next

1. **Uptime Kuma** — repeat the LXC static IP process above, assign `10.0.0.20`
2. **Kali Linux (VM)** — static IP is set differently for VMs (inside the guest OS, not the Proxmox Network tab):
   - Console into Kali → Advanced Network Configuration → select connection → edit
   - IPv4 Settings tab → Method: Manual → Add address `10.0.0.25`, netmask `24` (i.e. `/24`), gateway `10.0.0.1`
   - Reboot the VM, then verify with `ip a` / ping tests
   - Assign `10.0.0.25`
3. **Validate Service Availability** — confirm Gitea, Vaultwarden, and Uptime Kuma are all reachable at their new static IPs/ports in a browser
4. Once all four machines are on static IPs, move to **Section 3.4: Home Lab DNS** — installing CoreDNS so services can be reached by name (e.g. `gitea.lab`) instead of IP address
