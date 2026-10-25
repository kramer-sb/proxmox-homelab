# 20 - Tailscale access controls (ACL lockdown)

Status: **Done** on 2026-10-25. The tailnet now allows only my own account to reach anything. Any other user gets no access until a rule names them.

## Why

A published security report (Tailworm, 2026-10-25) described attacks that start from one compromised tailnet device and spread to others the access rules allow. The findings are unconfirmed by Tailscale and are about Tailscale SSH, which I do not use. Reading it still pointed at a real gap in my setup: the default policy let every device on the tailnet reach the whole `10.0.0.0/24` subnet, including the Proxmox UI and Vaultwarden. That mattered more once I planned to put family phones on the tailnet.

## What I checked first

1. Tailscale SSH is off on the router. On `ts-router` (`pct enter 2001`): `tailscale debug prefs | grep -i ssh` returned `"RunSSH": false`.
2. The tailnet has three devices, all under my account: `ts-router`, my Pixel, and my Windows host.
3. The default policy file also had an `ssh` block (all members, check mode) and a `funnel` attribute. I use neither.

## What I changed

In the admin console, Access controls, JSON editor, I replaced the default policy with one grant for my own account and removed the `ssh` and `nodeAttrs` sections:

```
{
	"grants": [
		{"src": ["<my-github-login>@github"], "dst": ["*"], "ip": ["*"]},
	],
	"tests": [
		{
			"src": "<my-github-login>@github",
			"accept": ["10.0.0.5:8006", "10.0.0.15:443", "10.0.0.45:443", "10.0.0.30:53"],
		},
	],
}
```

With no `ssh` section, Tailscale SSH is not permitted anywhere. With no `nodeAttrs` section, Funnel (public exposure of a device) is off. The `tests` block runs every time the policy is saved, so a mistake that locks me out of the lab shows up as an error instead.

## Adding a limited user later

Invite them as their own user (Users, Invite). Never have them sign in with my account, since my account has full access. Then add two grants, one for the app and one for DNS so the `.lab` name resolves:

```
{"src": ["their-login@github"], "dst": ["10.0.0.45"], "ip": ["tcp:443"]},
{"src": ["their-login@github"], "dst": ["10.0.0.30"], "ip": ["tcp:53", "udp:53"]},
```

## Verification

`https://gitea.lab` still loads from the Windows host, but that host is on the home LAN, so it does not exercise the Tailscale path. The real check passed on 2026-10-25: with Wi-Fi off and the phone on cellular data, `https://books.lab` loaded. That confirms the phone matches the full-access rule and Split DNS still resolves `.lab` names under the new policy.

## Related

- Enabled two-factor authentication on the GitHub account that signs in to Tailscale. Before that, anyone who took over the GitHub account would also control the tailnet.
