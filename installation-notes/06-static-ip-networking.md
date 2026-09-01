# 06 - Static IP Networking (Complete)

Status: **Complete** — Course Section 3.3 (Static IP Addresses). All home lab machines, including the Proxmox host itself, now run on fixed IPs.

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

## Step 2: Final Static IP Assignments

| Machine | Type | Static IP | Status |
|---|---|---|---|
| Proxmox host | Hypervisor | `10.0.0.5` | ✅ Done |
| Gitea | LXC | `10.0.0.10` | ✅ Done |
| Vaultwarden | LXC | `10.0.0.15` | ✅ Done |
| Uptime Kuma | LXC | `10.0.0.20` | ✅ Done |
| Kali Linux | VM | `10.0.0.25` | ✅ Done |

Assignment pattern: increments of 5, starting at `10.0.0.10` for services, with the Proxmox host itself set at `10.0.0.5`, leaving room to insert more machines later without renumbering.

## Step 3: Setting Static IPs on Proxmox LXCs

For each LXC (Gitea, Vaultwarden, Uptime Kuma):

1. Shut down the LXC in Proxmox
2. Select the LXC → **Network** tab → select `net0` → **Edit**
3. Click **Static** (deselecting DHCP)
4. Set **IPv4/CIDR** to the assigned IP with `/24` (e.g. `10.0.0.10/24`)
5. Set **Gateway** to `10.0.0.1`
6. Click **OK**, then power the LXC back on
7. Verify inside the LXC console (or via `pct enter <CTID>` from the host shell): `ip a`, `ping 8.8.8.8`, `ping google.com`

## Gitea — Completed

- Static IP set: `10.0.0.10`
- Verified network connectivity via `ip a` / ping tests
- Web UI confirmed reachable at `http://10.0.0.10:3000` (Gitea's default port — must be included, plain IP alone will not load it)

### Troubleshooting note: LXC console login rejected

Root password was rejected at the Proxmox Console login prompt despite being correct. Fastest workaround — run from the **Proxmox host shell**, not the LXC console:

```
pct enter <CTID>
passwd
```

This drops directly into the container as root (no password needed) and lets you set a fresh password without typing through the noVNC console at all. Useful as a general fallback any time an LXC console login seems to reject a password you're sure is correct.

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

## Uptime Kuma — Completed

- Static IP set: `10.0.0.20`
- Verified via `pct enter <CTID>` from the host shell: `ip a`, `ping 8.8.8.8`, `ping google.com` all passed
- Web dashboard confirmed reachable at `http://10.0.0.20:3001` (default Uptime Kuma port)

### Troubleshooting note: two separate login systems, easy to conflate

A reset OS/root password (`passwd` via `pct enter`) worked fine for the LXC console — confirmed in the systemd journal with a successful `ROOT LOGIN ON tty1` entry. The `pam_systemd... Failed to create session: Seat has no VTs` warning immediately above it is a harmless, common message in LXC containers and does **not** indicate a failed login.

Separately, trying that same root/OS password on the **Uptime Kuma web dashboard** (port 3001) failed with `[AUTH] WARN: Incorrect username or password for user root`. This is expected — Uptime Kuma's web dashboard has its own independent admin account (created during its first-run setup wizard), completely separate from the container's OS-level root password. The two were being conflated.

**Diagnostic path that resolved it:** checked `journalctl -n 50 --no-pager` inside the container (no `/var/log/auth.log` exists on this minimal image — no `rsyslog` installed) and compared timestamps of both login attempts to see they hit two different systems.

**Takeaway:** every self-hosted app with its own login page (Vaultwarden, Uptime Kuma, Gitea, etc.) has its own account system, separate from the LXC/VM's OS root password. Save each one distinctly in the password manager rather than assuming they share credentials.

## Kali Linux (VM) — Completed

- Static IP set: `10.0.0.25`
- Unlike the LXCs, static IP for a VM is **not** set through the Proxmox Network tab — it's configured inside the guest OS itself:
  1. Kali desktop → network icon (system tray) → Edit Connections
  2. Select the wired connection → Edit → **IPv4** tab
  3. Method: Manual → Add address `10.0.0.25`, netmask `24` (`255.255.255.0`), gateway `10.0.0.1`
  4. Save, then toggle the connection off/on (or reboot) to apply
- Verified via `ip a`, `ping 8.8.8.8`, `ping google.com` inside the VM

## Proxmox Host — Completed

The Proxmox host itself was still on DHCP throughout the LXC/VM work above, which caused a real disruption: after a mini PC restart, Proxmox's web UI became unreachable at its old address (`10.0.0.52`) because it had picked up a new DHCP lease. This is the exact scenario the Proxmox mini-course warns about.

- Found the new DHCP-assigned address (`10.0.0.124`) by checking directly at the physical console: `ip a`
- Logged into the web UI at the new address and set a permanent static IP:
  1. **Datacenter → node name → System → Network**
  2. Selected `vmbr0` (Linux Bridge, bound to `nic0`) → Edit
  3. Switched from DHCP to Static: IPv4/CIDR `10.0.0.5/24`, Gateway `10.0.0.1`
  4. **Apply Configuration** (session drops momentarily as the interface bounces)
  5. Reconnected at `https://10.0.0.5:8006/`
- Final, permanent Proxmox address: **`10.0.0.5`**

### Troubleshooting note: finding a changed IP with no display

If Proxmox's web UI ever becomes unreachable again after this, the fix is to connect a monitor/keyboard directly to the mini PC (or use an already-open console session) and run `ip a` to find its current real address — the same recovery step used here.

## What's Next

1. **Validate Service Availability** — confirm Gitea, Vaultwarden, and Uptime Kuma are all reachable at their new static IPs/ports in a browser (done for all three during this session)
2. Move to **Section 3.4: Home Lab DNS** — installing CoreDNS so services can be reached by name (e.g. `gitea.lab`) instead of IP address, removing the need for manual Windows hosts file edits going forward
