# 19a - Wazuh addendum: firewall rule events (4946 to 4948) never alerted

Status: **Resolved** with a custom rule. Windows logged the events and the manager received them, but no built-in rule produced an alert, so nothing showed under `data.win.system.eventID:4946` in the dashboard.

## Symptom

I added a firewall rule on VM 203 and deleted it. Searching the dashboard for event IDs 4946 and 4948 returned nothing. Account and group events (4720, 4732, 4625) did show up.

## What I checked

1. **Windows logged them.** `auditpol /get /subcategory:"MPSSVC Rule-Level Policy Change"` showed Success and Failure enabled, and `Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4946,4948}` returned the events.
2. **The agent was reading the Security log.** `ossec.log` on the VM showed `Analyzing event log: 'Security'`.
3. **The manager received them.** I turned on `<logall_json>yes</logall_json>` in `/var/ossec/etc/ossec.conf`, restarted the manager, and grepped `/var/ossec/logs/archives/archives.json` for `"eventID":"4946"`. The events were there. The archive file holds events before rule matching, so an entry there with no `rule` block does not mean anything about rule matching.
4. **No alert existed.** The same grep on `/var/ossec/logs/alerts/alerts.json` found nothing.

## Cause

Wazuh's Windows rules form a chain. Rule 60000 (any Windows event channel event) leads to 60001 (Security channel), then 60103 (audit success), and event-specific rules hang below that. No shipped rule below 60103 matches the firewall rule change events, so they were decoded and archived but never alerted.

## Fix

A custom rule chained under 60103, at the bottom of `/var/ossec/etc/rules/local_rules.xml`:

```
<group name="windows,firewall,">
  <rule id="100100" level="6">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4946$|^4947$|^4948$</field>
    <description>Windows Firewall rule change (event $(win.system.eventID)): $(win.eventdata.ruleName)</description>
  </rule>
</group>
```

Then test the syntax and restart:

```
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

After adding and deleting a test rule on VM 203, `rule.id:100100` in the dashboard showed the alerts.

## Gotchas

- My first attempts chained the rule under 60000 and 60001. They loaded without errors and never fired. The rule has to sit under the sibling that the event actually matches, which is 60103 for success audit events.
- `wazuh-logtest` decodes pasted Windows JSON as plain `json`, so it cannot confirm a rule chain for Windows event channel events. Test with a real event.
- This only covers the audit success path (60103). Failed attempts land under 60104.
- When grepping for the rule ID, use `grep '"id":"100100"'`. A looser `grep 100100` also matches your own shell history line in the log.
- Turn `logall_json` back to `no` when done. The archive file grows quickly.
- Remove test rules afterward. A leftover `LAB TEST` rule showed up later in the firewall review.
