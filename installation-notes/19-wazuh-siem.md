# 19 - Wazuh SIEM

Status: **Working**. Wazuh 4.14 (manager, indexer and dashboard on one VM) runs as VM 204 at `10.0.0.65`. The Windows 11 lab VM (VM 203) sends its Security, System, Application and Sysmon events to it, and the dashboard shows them as alerts.

## Goal

A SIEM I can run daily alert reviews on, using the Windows endpoint lab as the source of events. Security Onion already covers network traffic. Wazuh covers what happens on the endpoint itself: new accounts, admin group changes, firewall rule changes and failed logons.

## VM settings

| Setting | Value |
| --- | --- |
| VM ID / name | 204 / `wazuh` |
| OS | Ubuntu Server 24.04 LTS |
| Machine / BIOS | Default (i440fx) / Default (SeaBIOS) |
| SCSI controller | VirtIO SCSI single |
| Disk | 60 GiB on `local-lvm`, Discard on |
| CPU / memory | 4 cores (host CPU type), 6144 MiB, ballooning off |
| Network | VirtIO on `vmbr0` |
| Other | QEMU guest agent installed and enabled |

## What I did

1. **Made room.** I checked memory and VM IDs with `free -h` and `qm list` before adding another VM. `10.0.0.65` was the next open increment after the Windows VM.
2. **Created the VM** with the settings above and installed Ubuntu Server with a manual IPv4 setup: `10.0.0.65/24`, gateway `10.0.0.1`, DNS `10.0.0.30`. I detached the installer ISO afterward, because a leftover ISO caused trouble on VM 203 (see 17a).
3. **Installed the guest agent** with `apt install qemu-guest-agent`, enabled it, and turned on the QEMU Guest Agent option in Proxmox.
4. **Installed Wazuh** with the official all-in-one installer: `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a`. The install summary prints the `admin` password once, so I saved it in my password manager straight away. If it is missed, it can be recovered from `wazuh-install-files.tar` in the folder where the installer ran. That file never goes in this repo.
5. **Checked the three services** (`wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`) with `systemctl status`, and logged in to `https://10.0.0.65` as `admin`. The browser shows a certificate warning because the dashboard uses Wazuh's own self-signed certificate.
6. **Installed the agent on VM 203.** From the VM I confirmed ports 1514 and 1515 on the manager were reachable with `Test-NetConnection`, downloaded the Windows agent MSI (same version series as the manager), and installed it silently with the manager address and the agent name `LAB-WIN11-01`. Then I started `wazuhsvc`. The agent showed Active in the dashboard within a couple of minutes.
7. **Turned on the audit categories on VM 203** with `auditpol` (user account management, security group management, audit policy change, MPSSVC rule-level policy change, special logon, logon), so the events the dashboard is meant to show are actually logged. The firewall events did not alert at first, which is covered in [19a](19a-wazuh-firewall-events-addendum.md).
8. **Added Uptime Kuma monitors.** Two TCP Port monitors for `10.0.0.65`: port 443 (dashboard) and port 1514 (agent traffic). A ping monitor is not used, and the Wazuh VM has no hostname in CoreDNS yet.

## What I saw after install

- The agent log on VM 203 showed a few `Unable to connect to '[10.0.0.65]:1514/tcp'` errors right after the service started, then `Connected to the server` under a minute later, so this cleared on its own.
- The manager log showed `Failed to sync agent '001': No available server` warnings from the indexer connector while the indexer was not yet reachable. I did not chase them further.

## Open items

- No CoreDNS entry and no friendly hostname for the VM. It is reached by IP, the same as Security Onion and the Windows VM.
- The Wazuh VM keeps its dashboard certificate warning. A Caddy front end with a `.lab` name would remove it.

See also: [addendum on the missing firewall alerts](19a-wazuh-firewall-events-addendum.md) and the [functional doc](../functional-docs/wazuh.md).
