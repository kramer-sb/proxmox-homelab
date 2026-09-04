# 09a - Tailscale ICMP Connectivity Troubleshooting (Addendum)

> Addendum to `09-tailscale-remote-access.md`. Encountered during Section 4.3 setup while validating Tailscale connectivity between the Gitea LXC and the Windows management host.

## Symptom

After installing Tailscale on both the Gitea LXC (`10.0.0.10`) and the Windows host, connectivity was one directional.

- Windows host to Gitea (via Tailscale IP): worked, no packet loss
- Gitea to Windows host (via Tailscale IP `100.91.6.63`): 100% packet loss

## What Was Tried

Steps are listed in the order attempted. Each step ruled something out before the actual cause was found.

1. **Confirmed correct Tailscale IPs.** Verified `100.91.6.63` was in fact the Windows host's own Tailscale IP, not a copy/paste mix-up with Gitea's.
2. **Enabled Windows Firewall inbound ICMP rules.** The built-in "File and Printer Sharing (Echo Request - ICMPv4-In)" rules were disabled by default for the Private/Public profiles. Enabled them. Did not resolve the issue.
3. **Checked rule scope.** Found the Remote IP address scope was restricted to "Local subnet," which does not include Tailscale's `100.x.x.x` range. Changed scope to include `100.64.0.0/10` (Tailscale's CGNAT range). Did not resolve the issue.
4. **Checked Interface Types** on the rule's Advanced tab. Already set to "All interface types," so not the cause.
5. **Checked Local IP address** scope on the same rule. Already set to "Any IP address," so not the cause.
6. **Tested Edge Traversal setting.** Toggled from "Block edge traversal" to "Allow edge traversal" on the rule. Did not resolve the issue; reverted to Block (default, more secure).
7. **Checked Tailscale's own Windows client setting.** Confirmed "Allow incoming connections" was already enabled in the Tailscale tray menu under Preferences, ruling out Tailscale's built-in Shields Up feature.
8. **Ran `tailscale ping 100.91.6.63` from Gitea.** This is a Tailscale-native diagnostic, distinct from OS-level ping. It returned a successful `pong` via a direct LAN route. This confirmed the Tailscale tunnel itself was fully healthy, isolating the problem to somewhere in the Windows host's OS/security stack rather than Tailscale or basic network routing.
9. **Identified ESET Internet Security** was installed on the Windows host as a third-party security suite running its own independent firewall alongside Windows Defender Firewall.
10. **Temporarily disabled ESET's firewall** and re-tested. Ping succeeded, confirming ESET as the actual blocker.

## Root Cause

ESET Internet Security runs its own firewall engine that evaluates traffic independently of, and in addition to, Windows Defender Firewall. All Windows Firewall changes above were reasonable and generally worth keeping, but none of them could have fixed this. A pre-existing ESET rule named "Block ICMP communication" was explicitly denying all inbound ICMP traffic, found via ESET's Setup > Network Protection > "Resolve blocked communication" troubleshooting tool.

Additionally, ESET's firewall was running in Interactive mode, which prompts the user for every new connection any application makes, not just the traffic being tested. This needed to be switched to Automatic mode to stop unrelated prompt spam (Microsoft account sync, OneDrive, etc.) while working through the fix.

## The Fix

1. In ESET Setup > Network Protection > Firewall, switched Filtering mode from Interactive to Automatic.
2. Created a new ESET firewall rule to explicitly allow the traffic before the existing block rule catches it. ESET evaluates rules by ascending Priority number, so the first match wins:
   - Name: Allow ICMP - Home Lab & Tailscale
   - Action: Allow
   - Direction: Both
   - Protocol: ICMP
   - Local host: Any
   - Remote host: Trusted zone + `100.64.0.0/10` (Tailscale range)
   - Priority: set lower than "Block ICMP communication" so it's evaluated first
3. Verified the new rule sorted above "Block ICMP communication" in the Rules list.
4. Re-tested `ping 100.91.6.63 -c 4` from Gitea. Succeeded with no packet loss.

## Key Learning

When a device has a third-party antivirus/security suite installed (ESET, Norton, McAfee, Bitdefender, etc.), Windows Defender Firewall changes alone may be insufficient. These suites typically run their own separate firewall engine with its own rules, zones, and priority ordering that can silently override or duplicate OS-level settings.

`tailscale ping` (not regular `ping`) is a useful diagnostic to isolate whether a connectivity issue lives in the Tailscale/WireGuard layer versus the OS/security software layer. A successful `tailscale ping` alongside a failed regular `ping` points squarely at something in the OS or a security suite, not Tailscale itself.
