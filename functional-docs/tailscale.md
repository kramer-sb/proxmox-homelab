# Tailscale (Remote Access)

Clean, repeatable steps for the working Tailscale setup: subnet router + Split DNS, giving remote access to the entire `10.0.0.0/24` lab subnet and `.lab` domain resolution from any device on the tailnet, without installing Tailscale on every individual lab service.

## Overview

- **Subnet router:** a dedicated LXC (`ts-router`) running Tailscale, configured to advertise the whole `10.0.0.0/24` subnet. Once approved, any device on the tailnet can reach any lab IP through it, without needing Tailscale installed on that individual device.
- **Split DNS:** CoreDNS (`10.0.0.30`) is registered as a nameserver in the Tailscale admin console, restricted to the `lab` domain only. This lets any tailnet device resolve `.lab` hostnames while away from home, without needing CoreDNS set as their local DNS server.

## 1. Create the subnet router LXC

1. Build a plain Ubuntu LXC using the Proxmox VE Helper Scripts (Debian/Ubuntu base script), named `ts-router`, on a static IP.
2. Install Tailscale inside it following the official install script for the distro.
3. Authenticate it to the tailnet:
   ```
   sudo tailscale up
   ```

## 2. Enable IP forwarding on the router LXC

Required for the router to actually relay traffic, not just hold the route. Without this, the Tailscale admin console shows an "Unable to relay traffic" warning on the route.

```
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Verify:

```
sysctl net.ipv4.ip_forward
```

Should return `net.ipv4.ip_forward = 1`.

## 3. Advertise the subnet route

From the `ts-router` LXC:

```
sudo tailscale set --advertise-routes=10.0.0.0/24
```

## 4. Approve the route in the Tailscale admin console

New advertised routes are not automatically trusted. In the admin console, open the machine list, find `ts-router`, and approve the `10.0.0.0/24` route under its route settings. Confirm the "Unable to relay traffic" warning is gone (this confirms IP forwarding from step 2 took effect).

## 5. Configure Split DNS for `.lab`

In the Tailscale admin console, DNS tab:

1. Click "Add nameserver" > Custom.
2. Enter `10.0.0.30` (CoreDNS).
3. Enable "Restrict to a specific domain" (Split DNS) and enter `lab` (no leading dot).
4. Save.

This must be scoped to the `lab` domain specifically. Adding it as a global nameserver would send all DNS traffic (including normal internet browsing) through CoreDNS, which only knows how to resolve `.lab` names.

## 6. Client-side settings

On each device that should have remote access to the lab:

1. Install the Tailscale client and sign in to the same tailnet.
2. Enable "Use Tailscale subnets" (accept routes) so the device knows to route `10.0.0.0/24` traffic through `ts-router`.
3. Enable "Use Tailscale DNS settings" so the device picks up the Split DNS configuration from the admin console.

On Windows specifically, Split DNS is implemented via an NRPT (Name Resolution Policy Table) entry rather than changing the adapter's DNS server directly. This can be verified from an elevated PowerShell prompt:

```
Get-DnsClientNrptPolicy
```

Look for a `Namespace : .lab` entry pointing at `100.100.100.100` (Tailscale's MagicDNS resolver). Note that `nslookup` does not respect NRPT policy and will appear to fail even when resolution is actually working; use a browser or `Resolve-DnsName` in PowerShell to test instead.

## Verifying it works

From a device NOT on the home network (phone on cellular data, Wi-Fi off, with the Tailscale app installed and connected):

1. Browse to a `.lab` service, for example `https://gitea.lab`.
2. It should resolve and load exactly as it does on the home LAN, subject to accepting the same self-signed certificate warning as any other device that hasn't imported that service's Caddy CA (see `trusting-caddy-local-ca-windows.md`).

## Common pitfalls

- **Third-party firewalls/security suites** (ESET, Norton, etc.) run independently of the OS firewall and can silently block traffic even when OS-level firewall rules are correctly configured. See `../installation-notes/09a-tailscale-icmp-troubleshooting-addendum.md`.
- **Enabling "Use Tailscale subnets" before the router can actually relay traffic** (IP forwarding not yet enabled, or route not yet approved) breaks local access to the whole subnet, since the client starts routing that traffic through a router that cannot forward it. Disable "Use Tailscale subnets" to immediately restore local access if this happens.
- **`nslookup` is not a reliable test** for Windows Tailscale DNS. It bypasses the Windows DNS Client service (and therefore NRPT policy) entirely.
