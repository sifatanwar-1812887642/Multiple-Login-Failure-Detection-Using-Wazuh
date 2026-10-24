# Setup and verification

## 1. Enable auditing

On the isolated Windows endpoint, run Administrator PowerShell:

```powershell
auditpol /set /subcategory:"Logon" /failure:enable
auditpol /get /subcategory:"Logon"
```

The subcategory name is localized on non-English Windows. Select the equivalent Logon audit subcategory there.

Use a dedicated test account. Review its lockout policy before testing.

## 2. Confirm Security event collection

Inspect the Windows agent's existing ossec.conf, commonly at:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

If Security collection is missing, merge the block from config/windows-security-collection.xml inside the existing ossec_config. Keep existing collection blocks. Restart the installed Wazuh agent service after changing configuration.

## 3. Confirm a real failed login event

Make one deliberately incorrect network authentication to your own lab endpoint. Inspect its actual event:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 1 |
    Format-List TimeCreated, Id, Message
```

Confirm Event ID 4625, Logon Type 3, the test account, and the test source IP. Other logon types will not match the provided rule.

Confirm the event also arrives in Wazuh. Dashboard alert fields commonly appear under data.win.*, while custom-rule field names use win.*.

## 4. Install manager rules

Back up existing custom rules first. Verify that IDs 100201 and 100202 are unused:

```bash
sudo rg 'id="10020[12]"' /var/ossec/etc/rules /var/ossec/ruleset/rules
```

If rg is unavailable, use grep -R. If IDs already exist, resolve the conflict; do not load duplicate IDs. Copy rules/multiple_login_rules.xml to:

```text
/var/ossec/etc/rules/multiple_login_rules.xml
```

Retain suitable ownership and permissions consistent with other manager rule files. Then run:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

Only after a successful check:

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

## 5. Verify individual and repeated failures

Start with one controlled failed network login. Look for rule 100201:

```bash
sudo tail -n 200 /var/ossec/logs/alerts/alerts.json |
    jq 'select(.rule.id == "100201" or .rule.id == "100202")'
```

Then make five controlled failed attempts from one lab source within 120 seconds. Record the actual event count and timestamps and look for rule 100202. Stop if account lockout intervenes.

The intended threshold is five; confirm the observed trigger count on your installed version, with a clean test session and without earlier failures contributing to the window. Do not increase test volume blindly when an alert is absent.

Dashboard queries:

```text
rule.id:100201
rule.id:100202
```

## 6. Check the boundaries

| Test | Expected result |
| --- | --- |
| One qualifying failure | 100201; no correlation alert in a clean window |
| Repeated same-source failures within 120 seconds | 100202 once the configured threshold is reached |
| Failures spread across distinct source IPs below threshold | No combined same-source correlation |
| Failures separated beyond the window | No correlation from those expired events |
| Logon Type other than 3 | No 100201 match |
| Missing or placeholder source address | No 100201 match |

Run each case independently and retain raw events. The JSON in examples/failed-login.json is only an illustrative decoded event, not a complete Windows eventchannel replay fixture.

## Troubleshooting

- No Windows event: inspect audit policy and the authentication path.
- Event exists only on Windows: inspect agent status and Security-channel collection.
- Event reaches Wazuh but no 100201: verify base rule 60122, actual logon type, event ID and source field.
- 100201 appears but no 100202: inspect timing, IP consistency, agent identity and ID conflicts.
- Alerts missing from dashboard: inspect alerts.json first, then indexing and time filters.
- Archives are not assumed enabled. Do not rely on archives.json unless archival logging is configured.

## Evidence to retain

Capture the real Event 4625, single-failure alert, correlation alert, timestamps and source address. Redact unrelated account names and host identifiers before publishing screenshots. Historical screenshots are included in the README walkthrough. They document the earlier lab, not a live test of the revised repository files.
