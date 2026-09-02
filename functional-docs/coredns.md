# CoreDNS

Internal DNS server for the lab. Resolves `.lab` hostnames to their static IPs and forwards everything else upstream.

- **IP:** `10.0.0.30`
- **Port:** 53 (DNS)

## Adding a new `.lab` DNS entry

1. On the CoreDNS LXC, edit `/etc/coredns/lab.hosts`. Add a line:
   ```
   10.0.0.X    newapp.lab
   ```
   (tab-separated IP and hostname)
2. Restart CoreDNS:
   ```
   systemctl restart coredns.service
   ```
3. Verify:
   ```
   nslookup newapp.lab 10.0.0.30
   ```

## Setting DNS on a new machine

The Xfinity gateway does not support a DHCP-wide DNS override, so each
machine's DNS server must be set manually.

**LXCs (Proxmox DNS tab):**
1. Select the LXC in Proxmox → **DNS** tab → **Edit**
2. Set DNS servers to `10.0.0.30` → **Save**
3. Restart the LXC
4. Verify: `cat /etc/resolv.conf` shows `nameserver 10.0.0.30`

**VMs (guest OS network settings):**
Set `10.0.0.30` as the DNS server in the guest OS's network configuration
(e.g. Kali: Advanced Network Configuration → IPv4 Settings → DNS servers).

**Windows client machines:**
1. Network Connections → right-click active adapter → Properties
2. Internet Protocol Version 4 (TCP/IPv4) → Properties
3. "Use the following DNS server addresses" → Preferred DNS server: `10.0.0.30`
4. If DNS still doesn't resolve after this, also uncheck **Internet Protocol
   Version 6 (TCP/IPv6)** on the same adapter - Windows may otherwise prefer
   an IPv6 DNS server advertised by the gateway, which CoreDNS (IPv4-only)
   can't answer for.
5. `ipconfig /flushdns`, then test with `nslookup <hostname>.lab` (no server
   argument)

## Testing reference

- Test resolution using the system's configured DNS: `ping <hostname>.lab`
- Test resolution against a specific server: `nslookup <hostname>.lab 10.0.0.30`
  or `dig @10.0.0.30 <hostname>.lab`
- `ping <hostname>.lab <ip>` is **not** valid for targeting a specific DNS
  server the way `nslookup` is - `ping`'s second argument is a legacy
  source-routing hop, not a DNS server, and will fail even when DNS is fine.
