# 15a - bond0 MTU Mismatch Addendum

## Symptom

During the Security Onion 3.3.0 `so-setup` wizard (Eval mode, see `16-security-onion-installation.md`), the install ran for over an hour, all 1,869 Salt states succeeded (`Succeeded: 1869 (changed=1195)`, `Failed: 0`), but setup still exited with:

```
Install had a problem. Please see /root/sosetup.log for details.
A summary of errors can be found in /root/errors.log.
```

Contents of `/root/errors.log`:

```
Error: Connection activation failed: Unknown error
```

This generic message gives no indication of the actual cause. It appeared during a `Verifying setup` step that runs after the real configuration work, not during the configuration itself.

## Root cause

Security Onion bonds the monitor interface(s) into a `bond0` device (even with only one monitor NIC selected), which then feeds `sobridge`, the internal bridge used for packet capture. `bond0` is created with a default MTU of **9000** (jumbo frames). The actual monitor NIC in this environment, `ens19` (a VirtIO NIC on Proxmox's `vmbr0`), runs at the standard **1500** MTU, since `vmbr0` also carries regular home LAN traffic and neither the Xfinity gateway nor the rest of the network supports jumbo frames.

This MTU mismatch between `bond0` (9000) and its slave `ens19` (1500) causes the kernel's bonding driver to refuse the enslavement, which NetworkManager surfaces as the unhelpful "Unknown error."

Confirmed via:
```
ip link show bond0   # showed mtu 9000, state DOWN, NO-CARRIER
ip link show ens19   # showed mtu 1500, state UP, PROMISC (correctly in capture mode)
```

The specific kernel-level failure, visible in `journalctl -u NetworkManager`, was:
```
platform-linux: do-change-link[3]: failure 22 (Invalid argument)
device (bond0): attaching bond port ens19: failed
device (ens19): Activation: connection 'bond0-slave-ens19' could not be attached as port
```

## Fix

**Do not raise the whole network to jumbo frames.** `vmbr0` carries regular LAN traffic and the home gateway doesn't support it; changing bond0 to jumbo frames is the correct direction, not raising ens19 to match.

1. Update the connection profiles for both `bond0` and its slave to MTU 1500:
   ```
   sudo nmcli connection modify bond0 mtu 1500
   sudo nmcli connection modify bond0-slave-ens19 mtu 1500
   ```
2. **Order matters here.** `nmcli connection modify` only updates the saved profile; it does not push the change to an already-active interface until that connection is reactivated. `bond0` must be cycled *before* retrying the slave, otherwise the live `bond0` device is still running the old MTU (9000) when enslavement is attempted, and the same failure recurs:
   ```
   sudo nmcli connection down bond0
   sudo nmcli connection up bond0
   sudo nmcli connection up bond0-slave-ens19
   ```
3. Verify:
   ```
   ip link show bond0
   ```
   Should now show `mtu 1500` and `state UP` (not `DOWN`/`NO-CARRIER`).

## Was a full reinstall needed?

No. Since the underlying Salt configuration had already succeeded (0 failures) and only the post-install network verification step had failed, fixing the bond and confirming service health directly was sufficient:

```
sudo so-status
```

All 18 Security Onion containers, including `so-suricata` and `so-zeek`, came up healthy without re-running `so-setup`.

## For future installs on this hardware

If a future VM (Security Onion or anything else) is given a bonded monitor/capture interface on this Proxmox host, expect this same MTU mismatch unless `vmbr0`'s MTU is ever raised to 9000 lab-wide (not recommended, given the gateway and regular LAN traffic on the same bridge). Set `bond0`'s MTU to 1500 as part of setup rather than waiting for the generic failure to reappear.
