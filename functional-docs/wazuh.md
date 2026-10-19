# Wazuh (SIEM)

Reference for VM 204 (`wazuh`). The raw build log is in [19-wazuh-siem.md](../installation-notes/19-wazuh-siem.md), and the firewall alert problem is in [19a](../installation-notes/19a-wazuh-firewall-events-addendum.md).

| Item | Value |
| --- | --- |
| IP | `10.0.0.65` (static) |
| Dashboard | `https://10.0.0.65` (self-signed certificate warning is expected), login `admin` |
| Ports | 443 dashboard, 1514 agent events, 1515 agent enrollment, 55000 API |
| DNS name | none (accessed by IP) |
| Version | Wazuh 4.14, all-in-one (manager, indexer, dashboard) |
| Agent | `LAB-WIN11-01` on VM 203 (`10.0.0.60`) |
| Monitoring | Uptime Kuma TCP Port monitors for `10.0.0.65` ports 443 and 1514 |

## Create the VM

1. Create a VM: ID 204, name `wazuh`, Linux 6.x kernel type, default machine and BIOS, VirtIO SCSI single, 60 GiB disk on `local-lvm` with Discard on, 4 cores (host type), 6144 MiB RAM with ballooning off, VirtIO NIC on `vmbr0`.
2. Install Ubuntu Server 24.04 LTS. Set IPv4 manually: `10.0.0.65/24`, gateway `10.0.0.1`, DNS `10.0.0.30`. Install OpenSSH server.
3. Detach the installer ISO (Hardware, CD/DVD Drive, Do not use any media) and reboot.
4. Install the guest agent and enable the Proxmox option (Options, QEMU Guest Agent):

```
sudo apt update && sudo apt full-upgrade -y
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```

## Install Wazuh

Check the current version at https://documentation.wazuh.com/current/quickstart.html and change the number in the URL if needed.

```
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

Save the `admin` password from the summary in your password manager. To recover it, run this in the folder where you ran the installer (never commit the file):

```
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Check the services and the manager version (the agent must not be newer than the manager):

```
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
sudo /var/ossec/bin/wazuh-control info
```

## Install the agent on the Windows VM

In administrator PowerShell on VM 203:

```
Test-NetConnection 10.0.0.65 -Port 1515
Test-NetConnection 10.0.0.65 -Port 1514
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi" -OutFile "$env:TEMP\wazuh-agent.msi"
msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /q WAZUH_MANAGER="10.0.0.65" WAZUH_AGENT_NAME="LAB-WIN11-01"
Start-Service wazuhsvc
Get-Service wazuhsvc
```

Both ports should show `TcpTestSucceeded : True`. Use an MSI version that is not newer than the manager. A few connection-refused errors in `C:\Program Files (x86)\ossec-agent\ossec.log` right after the first start are normal while the manager finishes starting up.

To collect Sysmon events, `ossec.conf` on the VM needs this block inside `<ossec_config>`. Check with `Select-String -Path "C:\Program Files (x86)\ossec-agent\ossec.conf" -Pattern "Sysmon"`, and restart the service with `Restart-Service wazuhsvc` after any edit:

```
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The agent should show Active in the dashboard within a couple of minutes. Search `data.win.system.providerName:Microsoft-Windows-Sysmon` under Threat Hunting to see only Sysmon events.

## Windows audit policy

For the dashboard to show account, group, firewall and logon changes, turn on these audit subcategories on VM 203 (administrator PowerShell), then verify with `auditpol /get /category:*`:

```
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Audit Policy Change" /success:enable /failure:enable
auditpol /set /subcategory:"MPSSVC Rule-Level Policy Change" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
```

Useful dashboard searches: `data.win.system.eventID:4720` (account created), `4732` (added to Administrators), `4625` (failed logon), `rule.id:100100` (firewall rule changes).

## Custom rule for firewall rule changes

Wazuh has no shipped rule that alerts on events 4946, 4947 and 4948. Add this at the bottom of `/var/ossec/etc/rules/local_rules.xml`, then test and restart:

```
<group name="windows,firewall,">
  <rule id="100100" level="6">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4946$|^4947$|^4948$</field>
    <description>Windows Firewall rule change (event $(win.system.eventID)): $(win.eventdata.ruleName)</description>
  </rule>
</group>
```

```
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

The rule must chain under 60103 (audit success). It does not cover failed attempts, which land under 60104.

## Debugging events that do not alert

To see whether the manager receives an event at all, set `<logall_json>yes</logall_json>` in `/var/ossec/etc/ossec.conf`, restart the manager, and grep `/var/ossec/logs/archives/archives.json`. Archived events have no rule information, so a missing `rule` block there means nothing. Alerts are in `/var/ossec/logs/alerts/alerts.json`. Set `logall_json` back to `no` afterward because the file grows quickly. `wazuh-logtest` decodes pasted Windows JSON as plain `json`, so it cannot test the live rule chain for Windows events.

## Backups and snapshots

Take a Proxmox snapshot before big changes (snapshot names must start with a letter). The `local` partition is small, so a 60 GiB disk will not fit the scheduled local job. Use an on-demand backup to the `usb-backup` datastore instead. See [proxmox-backup-server.md](proxmox-backup-server.md).
