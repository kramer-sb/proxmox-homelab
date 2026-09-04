# 08b - Verifying Remote Access from a Phone on Cellular Data (Addendum)

> Addendum to `08-tailscale-remote-access.md`. Covers testing true "away
> from home" access using a phone on cellular data — not connected to home
> Wi-Fi — following the instructor's demo in the course video.

## Why this test matters

Testing remote access only from a device still sitting on the home LAN
doesn't actually prove anything about *remote* access — some problems (DNS
settings not actually being pushed over the internet, routing issues that
only appear once a device is truly off-network, etc.) only surface once a
device has no direct path to the lab subnet except through Tailscale. A
phone on cellular data, with Wi-Fi off, is a simple and realistic way to
confirm the whole chain — subnet router, IP forwarding, Split DNS — works
end to end for a device that's actually away from home.

## Steps

1. **Install the Tailscale app** on the phone (Google Play or Apple App
   Store — official Tailscale app).
2. **Sign in** with the same Tailscale account used for the rest of the
   tailnet (Proxmox, ts-router, etc.). The phone becomes a new device on
   the tailnet automatically — no extra approval needed for the phone
   itself to join, though it will start with default permissions.
3. **Turn off Wi-Fi** on the phone (or physically leave the home network's
   range) so it's on cellular data only. This is the actual "away from
   home" simulation.
4. **Confirm Tailscale is connected** in the app. Since `ts-router` is
   already approved and advertising the `10.0.0.0/24` subnet route, the
   phone should automatically gain access to the lab subnet through it —
   no additional per-device route configuration required.
5. **Browse to a `.lab` URL** (e.g. `https://gitea.lab`) in the phone's
   browser. If Split DNS is correctly configured in the Tailscale admin
   console (see `08a-tailscale-icmp-troubleshooting-addendum.md` for the
   Split DNS setup that had to be diagnosed on the Windows side), this
   should resolve and load the same way it does on the Windows management
   host.

### Note: the "Set up VPN connection" prompt is expected

The first time Tailscale requests to establish its tunnel, both Android
and iOS show a system-level **"VPN connection"** permission prompt. This is
normal and required — Tailscale is built on WireGuard, and both mobile OSes
require any app creating a network tunnel to go through this same OS-level
permission flow, regardless of what the app actually does with it.

This is **not** a traditional full-tunnel VPN. In this configuration
(subnet router advertising a specific `10.0.0.0/24` range, not acting as an
exit node), only traffic destined for the lab subnet is routed through
Tailscale. Regular browsing and other apps continue over cellular data
normally and are unaffected.

### Note: self-signed certificate warnings apply here too

Since Caddy's local CA (see
`../functional-docs/trusting-caddy-local-ca-windows.md`) was only imported
into the Windows Trusted Root store, the phone's browser will still show
the same self-signed certificate warning that Windows originally showed
before that import. This is expected — trusting a CA on one device does not
extend to others. Click through the warning for testing purposes; importing
a trusted CA on mobile is a separate, more OS-specific process not covered
here.

## Result

Confirmed: `gitea.lab` loaded successfully on a phone with Wi-Fi disabled,
using cellular data only — verifying the full remote-access chain (subnet
router → IP forwarding → Split DNS) works for a genuinely off-network
device, not just devices already on the home LAN.
