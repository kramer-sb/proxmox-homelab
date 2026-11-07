# Blocking scanner traffic on the Stalwart server

Session date: 2026-11-07. Only what actually happened is recorded. Anything not confirmed is marked unconfirmed.

## Starting question

ISP app showed notices that connection attempts to Stalwart were blocked. Eight source addresses were listed over time: 172.212.203.91, 20.118.27.212, 185.180.141.14, 71.6.134.236, 157.245.245.57, 194.180.49.56, 40.124.82.201 and 185.242.226.17. The question was what could be done to block them from the server side.

## What was found

- The notice for 71.6.134.236 said the attempt was blocked on Stalwart. That address is in 71.6.128.0/17, registered to CariNet, a hosting provider. Neighboring addresses in the same block appear on public threat-intelligence pages as scanners. The exact address was not found on any public page. Whether it is a scanner is unconfirmed, though the pattern fits.
- The other seven addresses were not looked up. Their owners are unconfirmed.
- It is unconfirmed whether the provider notices describe inbound or outbound traffic. The app's threat details for the device were not read in this session.
- In the container (`ssh stalwart`), a search of today's Stalwart log for 71.6.134.236 returned nothing. The connection never reached Stalwart.
- A search of the log for authentication events showed only successful logins to the brie mailbox from the LAN address 10.0.0.35, over IMAP on 993 and on port 443. No failed logins. That 10.0.0.35 is the PC is unconfirmed.
- Whether any `security.` ban events appeared in the log is unconfirmed. That search result was not reported.

## Firewall check (read-only, nothing changed)

- `iptables` is not installed in the container. The firewall is nftables.
- The loaded ruleset has an input chain with default policy accept and three rules for TCP port 26: accept from loopback, accept from five source ranges, then drop everything else on port 26.
- The nftables service is enabled at boot, and `/etc/nftables.conf` contains the same three port 26 rules, so the rules survive a reboot.
- The five allowed ranges match the ranges Dynu publishes for Email Store/Forward in its email service FAQ. The fifth (107.161.24.148) appears there under the Email Backup section. It is a Dynu address, so it does no harm. If Dynu changes its ranges, mail from new ones would be dropped. Check the FAQ and the Dynu Delivery Queue if inbound mail stops.

## Stalwart auto-ban (read-only, nothing changed)

- Settings > Security > Settings (Automatic IP Banning) is already set: authentication failures 100 per 1 day, abuse 35 per 1 day, loitering 150 per 1 day, port scanning 30 per 1 day, plus the default list of exploit-probe HTTP paths.
- All four ban durations are "Not set". Per the Stalwart docs, automatic bans are then permanent until the entry is deleted. Which Stalwart version the docs page describes is not stated; that it matches v0.16.27 is unconfirmed, but the admin page showed the same settings.
- Settings > Security > Blocked IPs is empty.

## Conclusion

No change was needed. Provider filters first, the container drops anything on port 26 that is not from Dynu, and Stalwart's own auto-ban is on with the defaults. Individual scanner addresses are not worth blocking by hand.

## Open items

- Read the threat details in the Provider app for the `stalwart` device to see whether the blocks are inbound or outbound. Unconfirmed.
- Confirm that the port 26 forward is currently in place in the app. The earlier handoff said it was deleted, and later notes describe inbound mail working. Unconfirmed.
- Confirm 10.0.0.35 is the PC (`ipconfig` in PowerShell).
- Optionally look up the seven other blocked addresses.

## Sources

- [Stalwart auto-ban](https://stalw.art/docs/server/auto-ban/)
- [Stalwart auto-ban (0.15)](https://stalw.art/docs/0.15/server/auto-ban/)
- [Dynu email service FAQ](https://downloads.dynu.com/FAQ/Email-Service)
- [Threat-intelligence entry for 71.6.134.237 (same CariNet block)](https://jamesbrine.com.au/71.6.134.237/)
