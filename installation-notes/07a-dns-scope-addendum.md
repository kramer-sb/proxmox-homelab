# DNS Scope Addendum — CoreDNS Boundary Verification

*Addendum to [07-home-lab-dns.md](07-home-lab-dns.md)*

## Why this addendum exists

After completing Section 3.4 (CoreDNS setup), shutting down the Proxmox host
caused **all web browsing to fail** on the Windows host machine - not just
access to lab services. This looked like an internet/router outage but was
actually a DNS scope leak: the Windows host had CoreDNS (`10.0.0.30`) set as
its DNS server, rather than falling back to Xfinity's DNS. This addendum
documents the incident, the fix, and the verified device boundary so it
doesn't happen again (or is quick to diagnose if it does).

## What happened

1. Proxmox host was shut down.
2. CoreDNS (an LXC running on Proxmox) went offline with it - expected.
3. Unexpectedly, **all** web pages stopped loading on the Windows host used
   to manage the lab - not just `.lab` domains or lab services.
4. Root cause: the Windows host's Wi-Fi adapter had `10.0.0.30` manually
   set as its DNS server (`ipconfig /all` confirmed this), likely left over
   from testing during Section 3.4. With CoreDNS down, the host had no DNS
   resolver at all for *any* domain, public or internal.

## The fix

On the Windows host:

1. **Settings → Network & Internet → Wi-Fi → [network] → Edit DNS
   settings** (or `ncpa.cpl` → adapter Properties → IPv4 → Properties)
2. Changed DNS from manual (`10.0.0.30`) back to **Automatic (DHCP)**
3. Ran `ipconfig /release` then `ipconfig /renew` to force the change to
   take effect immediately (Windows does not always apply a DNS change
   retroactively without this)
4. Confirmed via `ipconfig /all` that the Wi-Fi adapter now shows:
   ```
   DNS Servers . . . . . . . . . . . : 75.75.75.75
                                        75.75.76.76
   ```
   (`75.75.75.75` / `75.75.76.76` are Comcast/Xfinity's public DNS
   resolvers, handed out automatically by the gateway via DHCP.)

## Verified device boundary

The intended design is: **only the lab VMs/LXCs use CoreDNS.** Everything
else (management host, personal devices) should use the Xfinity gateway's
default DNS. This was audited device-by-device and confirmed correct as of
this writing:

| Device | DNS Server(s) | Expected? |
|---|---|---|
| Gitea LXC (`10.0.0.10`) | `10.0.0.30` (CoreDNS) | ✅ Correct |
| Vaultwarden LXC (`10.0.0.15`) | `10.0.0.30` (CoreDNS) | ✅ Correct |
| Uptime Kuma LXC (`10.0.0.20`) | `10.0.0.30` (CoreDNS) | ✅ Correct |
| Kali VM (`10.0.0.25`) | `10.0.0.30` (CoreDNS) | ✅ Correct |
| Proxmox host (`10.0.0.5`) | `75.75.75.75` / `75.75.76.76` | ✅ Correct |
| Windows management host | `75.75.75.75` / `75.75.76.76` | ✅ Correct (fixed) |

The Xfinity gateway does **not** hand out CoreDNS via DHCP to the network at
large - it only ever served `10.0.0.30` to devices that were manually
configured to use it. So the leak was isolated to the one host and did not
affect phones, TVs, or other devices on the network.

## Expected behavior when Proxmox is shut down

This was live-tested after the fix and confirmed working as intended:

- **General internet browsing** (Windows host, phones, TVs, streaming
  services) - unaffected. These rely on gateway/Xfinity DNS, not CoreDNS.
- **Lab services by hostname** (`gitea.lab`, `https://passwords.local`,
  `kuma.lab`, etc.) - unreachable, since CoreDNS is what resolves those
  names. This is expected and correct; it is the tradeoff of running local
  DNS on infrastructure that isn't always-on.
- Lab services would still be reachable by raw IP (e.g. `10.0.0.10:3000`)
  if needed while Proxmox is up but CoreDNS specifically is down - not
  applicable here since CoreDNS lives on Proxmox itself.

## Troubleshooting reference

If a "sudden internet outage" happens again after a Proxmox shutdown, check
DNS scope first before assuming a router/ISP issue:

```
ipconfig /all
```

Look at the **DNS Servers** line for the active adapter (Wi-Fi or
Ethernet). If it shows `10.0.0.30` and Proxmox is down, that's the cause -
switch DNS back to automatic per the steps above.
