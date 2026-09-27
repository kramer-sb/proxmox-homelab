# 15 - Security Onion Installation (Eval Mode)

## Purpose

Added Security Onion for network security monitoring (NSM): Suricata (IDS), Zeek (network metadata), and the associated alerting/hunting/dashboard stack (Elasticsearch, Kibana, SOC console). Installed in **Eval mode**, which is intended for homelab/evaluation use, not production, and does not run Logstash or Redis.

Scope decision: monitors lab traffic on `vmbr0` only (inter-VM/LXC traffic), not the whole home network. Monitoring the whole home network would require a managed switch with a SPAN/mirror port or an inline network TAP between the Xfinity gateway and the rest of the network, neither of which is currently in place.

## Prerequisites: freeing up resources

Before installing, freed up both disk and RAM headroom:

- **Removed the original course Kali Linux VM** (VMID 200, no longer needed): `qm destroy 200 --purge`, followed by deleting its lingering backup files from `/var/lib/vz/dump/`.
- **Confirmed disk was not actually a constraint.** The Proxmox Summary widget's "HD space" only reflects the small `local` root partition (96 GB). The actual VM/LXC storage pool, `local-lvm` (an LVM-thin pool), was 815.4 GB, with existing VMs/LXCs only consuming ~58 GB of real physical space at the time (thin provisioning), leaving ample room for Security Onion's 200 GB virtual disk.
- **Reclaimed RAM via BIOS.** Proxmox reported only ~12.6 GB total RAM despite 16 GB being physically installed (two 8 GB DDR4 SODIMMs, confirmed via `dmidecode -t memory`). The gap was the BIOS's UMA (integrated graphics) frame buffer reservation, found under **Advanced → UMA Frame buffer Size**, which defaulted to **3G**. Since Proxmox runs headless, dropped this to **512M**. This reclaimed the missing RAM (host went from ~12.6 GB to ~15 GB total / ~12 GB available), which was enough to run Security Onion's Eval mode (8 GB minimum) without any RAM purchase.
- Motherboard's actual RAM ceiling is 32 GB (per its own DMI data), not the 64 GB some listings advertise. See `functional-docs/gmktec-nucbox-g10-pro-hardware.md` for full specs.

## Downloading and verifying the ISO

Current release at time of install: **Security Onion 3.3.0**, built 2026-09-11.

```
cd /var/lib/vz/template/iso
wget https://raw.githubusercontent.com/Security-Onion-Solutions/securityonion/3/main/KEYS -O - | gpg --import -
wget https://github.com/Security-Onion-Solutions/securityonion/raw/3/main/sigs/securityonion-3.3.0-20260911.iso.sig
wget https://download.securityonion.net/file/securityonion/securityonion-3.3.0-20260911.iso
gpg --verify securityonion-3.3.0-20260911.iso.sig securityonion-3.3.0-20260911.iso
```

Downloading directly into `/var/lib/vz/template/iso` puts the ISO where Proxmox's VM wizard looks for it automatically.

The `gpg --verify` output shows a "Good signature" line along with a "WARNING: This key is not certified with a trusted signature!" line. The warning is expected (no personal web-of-trust signature on their key) and is not a failure.

Also confirmed the published SHA256 checksum matched: `sha256sum securityonion-3.3.0-20260911.iso`.

## VM configuration (Proxmox)

VM numbering convention: 200-range is for VMs (LXCs use the ranges elsewhere in `overview.md`). VM 200 (removed Kali) and 201 (kali-python) already used the low end, so Security Onion is **VMID 202**.

| Setting | Value |
|---|---|
| VMID | 202 |
| Name | security-onion |
| OS type | Linux, 6.x - 2.6 Kernel |
| Machine | Default (i440fx) |
| BIOS | Default (SeaBIOS) |
| SCSI Controller | VirtIO SCSI single |
| Disk (scsi0) | local-lvm, 200 GiB, Discard on, iothread on |
| CPU | 4 cores, type `host` (for AES-NI passthrough) |
| Memory | 10240 MiB, ballooning **off** |
| net0 | VirtIO, bridge vmbr0 (management) |
| net1 | VirtIO, bridge vmbr0 (added after initial VM creation, via Hardware → Add → Network Device; this becomes the monitor interface) |

Ballooning is deliberately off, consistent with how other VMs in this lab are configured (a VM reserves its full assigned RAM whether or not it's using it).

## Base OS install

Booted the ISO, confirmed the disk-wipe warning (safe, only affects the new 200 GB virtual disk), created the `brie` user, set a root password (saved to Vaultwarden), and let the installer run (based on Oracle Linux, not Rocky/Ubuntu as older SO documentation might suggest).

After the base install completed:
1. Detached the ISO from the VM (Hardware → CD/DVD Drive → Edit → "Do not use any media") before rebooting, to avoid looping back into the installer.
2. VM required a hard **Stop**/**Start** rather than Reboot/Shutdown at this point, since ACPI signals had nothing left in the guest to respond to them (the installer environment had already torn itself down). This produced a `TASK ERROR: VM quit/powerdown failed - got timeout`, which is expected in that situation, not a real error.

## Security Onion setup wizard (`so-setup`)

Logged in as `root`, ran `sudo so-setup`, and selected:

| Prompt | Selection |
|---|---|
| Installation type | Install |
| Deployment type | **EVAL** |
| ELv2 license | AGREE |
| Internet access | Standard |
| Hostname | `security-onion` |
| Management NIC | `ens18` (matches `net0`'s MAC) |
| Network config | STATIC |
| Management IP | `10.0.0.55/24` |
| Gateway | `10.0.0.1` |
| DNS servers | `10.0.0.30` (CoreDNS, matching the rest of the lab's DNS boundary) |
| DNS search domain | left as default `searchdomain.local` (rejected a bare `lab` entry as invalid input; this field isn't actually used since all lab services are accessed by their full `.lab` name already) |
| Internet connection method | Direct |
| Docker IP range | Yes (keep default; separate from the `10.0.0.0/24` lab network, no conflict) |
| Monitor interface(s) | `ens19` (matches `net1`'s MAC) |
| Admin account email | `brie@homelab.local` |
| Web interface access method | IP (chosen over hostname/FQDN since no DNS entry exists yet for `security-onion`) |
| Allow web access | Yes |
| Allowed IP/subnet | `10.0.0.0/24` (LAN-only; not exposed over Tailscale, consistent with the fact that `kali-python` also isn't on Tailscale, only the Pixel phone and Windows host are) |
| SOC Telemetry | **Disabled** (privacy-first choice; adjustable later via SOC Configuration screen if desired) |

## Result

Confirmed via `sudo so-status`: all 18 containers running and healthy, including `so-suricata` and `so-zeek`. See `15a-bond0-mtu-mismatch-addendum.md` for a network issue hit and resolved during this install.

Access: `https://10.0.0.55` (self-signed cert warning is expected), login `brie@homelab.local`.

## Follow-ups / not yet done

- No CoreDNS entry yet for `security-onion` (accessed by IP only for now)
- Haven't yet verified the monitor interface is actually capturing lab traffic (planned: generate test traffic from Kali against another lab host and check for entries in the SOC Alerts/Hunt views)
- Whole-home-network monitoring (Option B, requiring a mirror port or TAP) was considered but not pursued; lab-only monitoring was chosen instead
