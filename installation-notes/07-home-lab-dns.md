# 07 - Home Lab DNS (Complete)

Status: **Complete** - Course Section 3.4 (Home Lab DNS / CoreDNS). All lab
services now resolve by `.lab` hostname instead of raw IP, from every machine
that touches the lab.

## Goal

Stand up CoreDNS as the internal DNS server for the `10.0.0.0/24` lab
network, so hostnames like `gitea.lab` resolve to their static IPs - removing
the need for manual Windows hosts file edits (previously required for
`passwords.local`, see `06-static-ip-networking.md`).

## CoreDNS Server

- LXC static IP: `10.0.0.30`
- Provides authoritative answers for `.lab` domains and forwards everything
  else upstream (confirmed working - see verification below)

## Verification Method

Two tools, two different jobs:

- `nslookup <domain> <dns-server-ip>` - queries a **specific** DNS server
  directly, bypassing whatever's configured on the machine. Useful for
  testing a DNS server in isolation.
- `ping <hostname>` - uses whichever DNS server is configured in the OS's
  network settings (`/etc/resolv.conf` on Linux, adapter DNS settings on
  Windows).

**Confirmed working:**

```
nslookup gitea.lab 10.0.0.30
→ Name: gitea.lab, Address: 10.0.0.10   (authoritative, internal resolution)

nslookup facebook.com 10.0.0.30
→ Non-authoritative answer, real public IP returned   (forwarding works)
```

### Troubleshooting note: `nslookup <domain> <ip>` vs. plain `nslookup <domain>`

Early confusion running `nslookup gitea.lab 10.0.0.10` - this targeted
**Gitea's own IP** as if it were the DNS server, not CoreDNS's IP
(`10.0.0.30`). The second argument to `nslookup` must always be the DNS
server's address, not the target host's address.

### Troubleshooting note: `ping <host> <ip>` is NOT the same pattern as `nslookup`

Attempted `ping gitea.lab 10.0.0.30` expecting it to test resolution against
a specific server, the way `nslookup` does. This is invalid for `ping`.
`ping`'s syntax is `ping [hop...] destination` — the second argument is
interpreted as a legacy IP loose-source-routing hop, and the **last**
argument becomes the actual destination. So this command actually tried to
ping `10.0.0.30` (CoreDNS's own IP) via a source-routing hint through
`gitea.lab` - and modern routers/kernels silently drop loose-source-routed
packets for security reasons, producing 100% packet loss regardless of
whether DNS is working.

**Correct usage:**
- Test plain resolution: `ping <hostname>.lab` (uses configured resolver)
- Test resolution against a specific server: `nslookup <hostname>.lab <server-ip>`
  or `dig @<server-ip> <hostname>.lab`

## Xfinity Gateway: DHCP-wide DNS override is not available

Attempted to set `10.0.0.30` as the DNS server pushed out by DHCP, at
**Gateway → Connection → Local IP Network** in the Xfinity admin UI
(`http://10.0.0.1`). This page only exposes DHCP range, subnet mask, and
lease time - no DNS override field. Confirmed via research this is a known
Xfinity/Comcast restriction; DNS is locked to Comcast's own servers at the
gateway level unless the gateway is put into Bridge Mode with a separate
router (out of scope for this course).

**Fallback adopted:** manually set the DNS server on each device individually
instead of relying on DHCP. A reusable step-by-step doc for this exists at
`functional-docs/proxmox-lxc-dns-config.md`.

## Per-Machine DNS Configuration: Completed

| Machine | Static IP | DNS Server Set | Verified |
|---|---|---|---|
| Gitea LXC | `10.0.0.10` | `10.0.0.30` | ✅ `ping` + `nslookup` |
| Vaultwarden LXC | `10.0.0.15` | `10.0.0.30` | ✅ `ping` + `nslookup`| 
| Uptime Kuma LXC | `10.0.0.20` | `10.0.0.30` | ✅ `ping` + `nslookup`|
| Kali VM | `10.0.0.25` | `10.0.0.30` | ✅ `ping gitea.lab` + `nslookup kuma.lab 10.0.0.30` |
| Windows client (main machine) | DHCP | `10.0.0.30` (IPv4) | ✅ see below |

For the three LXCs: Proxmox → select LXC → **DNS** tab → Edit → set DNS
servers to `10.0.0.30` → Save → restart LXC → verify via
`cat /etc/resolv.conf` + `ping`/`nslookup` from inside.

For Kali (VM, no Proxmox-level DNS tab): configured inside the guest OS
under **Advanced Network Configuration → IPv4 Settings**, DNS servers field
set to `10.0.0.30`.

## Windows Client DNS Configuration: Completed

The Windows machine used for browsing to lab services (e.g.
`gitea.lab:3000`) needed the same manual fix, since it's on DHCP and Xfinity
won't push `10.0.0.30` automatically.

**Identifying the correct adapter:** the machine has several network
adapters (Bluetooth, unplugged Ethernet, VirtualBox/Hyper-V/VMware virtual
adapters, and Wi-Fi). Only **Wi-Fi** (`BigBlueHouse 2`) is the real
connection to the home network/internet - the VirtualBox Host-Only adapter
and other virtual adapters are for VM-to-host traffic only and don't route
to the Xfinity gateway.

**Steps applied (Wi-Fi adapter):**

1. Network Connections → right-click **Wi-Fi** → Properties
2. Select **Internet Protocol Version 4 (TCP/IPv4)** → Properties
3. "Use the following DNS server addresses" → Preferred DNS server:
   `10.0.0.30`
4. OK to save
5. `ipconfig /flushdns`, then retest with plain `nslookup gitea.lab`

### Troubleshooting note: IPv6 DNS override required too

After setting the IPv4 DNS server, `nslookup gitea.lab` (no server
specified) still failed, returning results from `cdns01.comcast.net` at an
IPv6 address (`2001:558:feed::1`). Windows was still using the IPv6 DNS
server auto-advertised by the Xfinity gateway, since IPv4 and IPv6 DNS are
configured independently. Because CoreDNS only has an IPv4 address, there's
no IPv6 address to point the IPv6 DNS field at.

**Fix:** on the same Wi-Fi adapter Properties screen, unchecked
**Internet Protocol Version 6 (TCP/IPv6)** entirely (not uninstalled, just
disabled for this adapter). This forces Windows to resolve DNS via IPv4
only, using the `10.0.0.30` setting. Confirmed working after
`ipconfig /flushdns` + retest. Reversible at any time by re-checking the box.

## What's Next

- **Section 3.5: SSL Termination** - see `08-ssl-termination.md`. Adds Caddy
  as a reverse proxy in front of each service so they're reachable over
  `https://` without a port number, instead of `http://gitea.lab:3000`.
