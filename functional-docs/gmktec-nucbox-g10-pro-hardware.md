# Functional Documentation: GMKtec NucBox G10 Pro (Host Hardware)

> **Functional documentation** is the short, orderly, repeatable version. This one is a reference sheet for the Proxmox host's real hardware limits, taken from the machine itself (`dmidecode`, `lscpu`, `lsblk`, `lspci`) and not from the retail listing. Check it before deciding whether to add RAM, storage, or another VM. See [`proxmox.md`](proxmox.md) for the install and hardening steps.

## At a glance

| | |
|---|---|
| **Host** | GMKtec NucBox G10 Pro, running Proxmox VE on bare metal |
| **CPU** | AMD Ryzen 5 3500U (Zen+), 4 cores / 8 threads, 1 socket |
| **Virtualization** | AMD-V (`svm` flag present, enabled in BIOS) |
| **RAM installed** | 16 GB: 2 x 8 GB DDR4 SO-DIMM, dual channel |
| **RAM maximum** | **32 GB** (2 slots, per the board's own SMBIOS data) |
| **Primary storage** | 1 TB NVMe SSD (TWSC TSC3AN1T0-F6Q10S) |
| **Wired network** | 1 x Realtek RTL8125 2.5GbE |
| **Wireless** | Realtek RTL8822CE 802.11ac (not used by the lab) |

## RAM

- `dmidecode -t memory` reports a maximum capacity of **32 GB** across **2 slots**, both populated.
- Retail listings for this model advertise support for 64 GB. The board's SMBIOS data says 32 GB, so plan around 32 GB.
- Both modules are 8 GB DDR4 SO-DIMM, rated 2667 MT/s, and run at 2400 MT/s. That fits the 3500U's memory controller, so 2400 MT/s is the expected speed for this CPU.
- Upgrade path: replace both sticks with 2 x 16 GB DDR4 SO-DIMM for 32 GB total. There is no free slot, so adding RAM means swapping, not adding.
- ECC is not supported.

## CPU

- 8 logical CPUs available to Proxmox. Plan VM and LXC core counts against that, and remember that VMs can be given more vCPUs than the host has threads but will compete for them.
- Virtualization is confirmed: `lscpu` shows `Virtualization: AMD-V`, and the flags include `svm`, `npt`, and `avic`. No BIOS change is needed for VMs.
- A few kernel-reported CPU vulnerabilities are mitigated with SMT left on (`Retbleed` is flagged "SMT vulnerable"). This is a personal lab, so it is accepted as is.

## Storage

- One internal NVMe drive, about 954 GB raw. Proxmox default layout (LVM):
  - `pve-root`: 96 GB
  - `pve-swap`: 8 GB
  - `pve-data` (LVM-thin pool for VM and LXC disks): about 815 GB
- Disks in the thin pool are thin provisioned, so the total of all disk sizes can exceed what is physically free. Watch actual pool usage in the Proxmox UI (Datacenter > Storage), not just the allocated sizes.
- The `STORE N GO` USB drive (about 115 GB) is the external backup target. See [`backup-strategy.md`](backup-strategy.md).
- **Not verified:** whether the second M.2 2280 slot mentioned in the retail listing is physically present and usable. `lsblk` shows only one NVMe device. Open the case and look before buying a second drive.

## Networking

- One physical Ethernet port, a Realtek RTL8125 2.5GbE controller. There is no second NIC, so there is no dedicated WAN/LAN split on this box.
- The listing for this model also mentions a USB-C port and dual HDMI. None of that matters for a headless Proxmox host.
- Wi-Fi is present but not used. The host should stay wired.

## Quick "can we add X" checks

| Question | Answer |
|---|---|
| More RAM? | Up to 32 GB total, but only by replacing both 8 GB sticks with 2 x 16 GB. |
| More storage? | Second NVMe slot unverified. USB storage works for backups. |
| Second NIC? | Only via a USB 2.5GbE or gigabit adapter. Test driver support before relying on it. |
| More vCPUs? | 8 threads total. Heavy VMs (Security Onion, Wazuh, Windows) will compete. |
| PCIe or GPU passthrough? | Not tested. IOMMU status has not been checked on this host. |

## Verify these yourself

Run on the Proxmox host shell to refresh this sheet:

```bash
dmidecode -t memory
lscpu
lsblk -o NAME,SIZE,MODEL,TRAN
lspci | grep -i -E "ethernet|network"
```
