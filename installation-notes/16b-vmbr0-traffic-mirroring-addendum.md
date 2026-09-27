# 16b - vmbr0 Traffic Mirroring Addendum

## Symptom

After the `so-setup` install completed and the bond0/MTU issue (`16a-bond0-mtu-mismatch-addendum.md`) was fixed, `so-status` showed all containers healthy and the monitor interface (`ens19`, bonded into `bond0`) was up. However, generating test traffic from `kali-python` (a one-minute ping and an `nmap -sV` scan against another lab host) produced zero results on the Alerts page, and would have produced zero on Hunt as well.

## Root cause

`vmbr0` is a standard Linux bridge (not Open vSwitch). A standard Linux bridge only floods broadcast, multicast, and unknown-unicast frames to every port. Once it has learned which port a VM's MAC address lives behind (which happens almost immediately after any traffic from that VM), it switches unicast frames directly port-to-port instead of flooding them.

This means putting a second NIC (`ens19`/`net1` on VM 202) on the same `vmbr0` bridge in promiscuous mode does **not** make it see other VMs' unicast traffic. Promiscuous mode only affects what a NIC accepts once a frame arrives at it; it does nothing to make the bridge deliver frames there in the first place. Real port mirroring requires either converting the bridge to Open vSwitch with a configured mirror port, or using Linux `tc` (traffic control) filters with a `mirred` action to explicitly copy traffic to the monitor port.

Confirmed by:
- Zero Alerts and zero Hunt results from generated Kali-to-lab test traffic, despite `so-status` showing `so-suricata` and `so-zeek` healthy and `ens19` correctly in `PROMISC` mode.
- Proxmox community forum threads describing the identical symptom and root cause, e.g. ["Linux bridge port mirror using tc (only receiving broadcast traffic)"](https://forum.proxmox.com/threads/linux-bridge-port-mirror-using-tc-only-receiving-broadcast-traffic.47395/) and ["Deploying Security Onion / Proxmox Port mirroring"](https://forum.proxmox.com/threads/deploying-security-onion-proxmox-port-mirroring.37036/).

## Fix

Mirror `vmbr0` traffic to VM 202's monitor-NIC tap interface (`tap202i1`) using `tc` filters with a `mirred egress mirror` action (never `redirect`, which would steal the frame from its real path instead of copying it).

Found `tap202i1` (VM 202's `net1`, the monitor NIC) via:
```
ip link show | grep tap202
```
Note: `tap202i1` sits behind Proxmox's per-NIC firewall bridge (`fwbr202i1`) since `net1` has `firewall=1` set. This doesn't matter for the mirror: `tc mirred` transmits the mirrored copy directly onto the named device as a raw frame injection, which reaches the VM's `ens19` regardless of what bridge the tap is a member of.

Live test commands (run on the Proxmox host):
```
tc qdisc add dev vmbr0 ingress
tc filter add dev vmbr0 parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev tap202i1

tc qdisc add dev vmbr0 handle 1: root prio
tc filter add dev vmbr0 parent 1: protocol all u32 match u8 0 0 action mirred egress mirror dev tap202i1

ip link set vmbr0 promisc on
```
Confirmed both filters registered via `tc filter show dev vmbr0 parent ffff:` and `tc filter show dev vmbr0 parent 1:`, then confirmed real alerts appeared (`GPL ICMP PING *NIX` from the ping test, `ET SCAN Nmap Scripting Engine User-Agent Detected` and `ET SCAN Possible Nmap User-Agent Observed` from the `nmap -sV` test).

## Persistence

`tc` filters don't survive a reboot on their own, and adding them as `post-up` lines directly on the `vmbr0` stanza in `/etc/network/interfaces` doesn't work either: that file is applied during early boot, before any VMs (including Security Onion) have started, so `tap202i1` wouldn't exist yet and the commands would silently fail.

Instead, tied the mirror to VM 202's own lifecycle with a Proxmox hookscript, so it reapplies automatically whenever the VM (re)starts, when the tap interface is guaranteed to exist:

```
mkdir -p /var/lib/vz/snippets
```
`/var/lib/vz/snippets/so-mirror.sh`:
```bash
#!/bin/bash
VMID="$1"
PHASE="$2"

if [ "$PHASE" == "post-start" ]; then
    tc qdisc add dev vmbr0 ingress 2>/dev/null
    tc filter add dev vmbr0 parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev tap202i1 2>/dev/null

    tc qdisc add dev vmbr0 handle 1: root prio 2>/dev/null
    tc filter add dev vmbr0 parent 1: protocol all u32 match u8 0 0 action mirred egress mirror dev tap202i1 2>/dev/null

    ip link set vmbr0 promisc on
fi

if [ "$PHASE" == "pre-stop" ]; then
    tc filter del dev vmbr0 parent ffff: 2>/dev/null
    tc filter del dev vmbr0 parent 1: 2>/dev/null
fi
```
```
chmod +x /var/lib/vz/snippets/so-mirror.sh
qm set 202 --hookscript local:snippets/so-mirror.sh
```

Verified by cycling the VM (`qm shutdown 202` / `qm start 202`) and confirming the filters reappeared on `vmbr0` with no manual `tc` commands, then re-running the ping/nmap test and seeing alerts again.

## Was a full reinstall needed?

No. This was purely a Proxmox host-level networking gap, not a Security Onion configuration problem. No changes were made inside the Security Onion VM itself.

## For future installs on this hardware

Any future sensor/IDS VM given a "monitor" NIC on `vmbr0` (or any other standard Linux bridge) needs this same `tc mirred` + hookscript setup to actually see other VMs' traffic. Promiscuous mode on the monitor NIC alone is not sufficient on a stock Linux bridge; only Open vSwitch's built-in mirror ports would remove the need for this. Update the hookscript's `tap<VMID>i1` reference if the VMID changes.
