# Multiple Login Failure Detection Using Wazuh

A defensive detection engineering lab that identifies repeated Windows authentication failures from the same source IP using Wazuh SIEM.

## Objective

Collect Windows Security Event ID **4625**, detect an individual network logon failure, and raise a higher-severity alert when failures repeat from the same source within a **120-second** window.

The lab's configured correlation threshold is **5**. This project uses **Multiple Login** as its title; the implemented rule specifically filters **Logon Type 3 (Network)**. It does not detect every Windows logon type or prove which service caused a failure.

## Detection flow

```mermaid
flowchart TD
    A["Windows Security: Event 4625"] --> B["Wazuh agent: Security channel"]
    B --> C["Rule 100201: single network logon failure"]
    C --> D["Same source IP within 120 seconds"]
    D --> E["Rule 100202: multiple login failure alert"]
```

## Lab environment

| Component | Recorded lab configuration |
| --- | --- |
| Wazuh manager | Ubuntu 24.04, Wazuh 4.14.5 |
| Windows endpoint | Windows 11, agent 002 |
| Test source | Ubuntu VM |
| Windows event | Security Event ID 4625 |
| Observed authentication | NTLM, Logon Type 3 |
| Base rule | 60122 |
| Single failure | 100201, level 7 |
| Repeated failures | 100202, level 12 |

These are historical lab details from the author's implementation report, not requirements for all installations. Confirm field names and base-rule behavior on your installed Wazuh version.

## Repository contents

- [Detection rules](rules/multiple_login_rules.xml)
- [Windows event collection snippet](config/windows-security-collection.xml)
- [Setup and verification](docs/SETUP.md)
- [Response notes and limitations](docs/RESPONSE.md)
- [Synthetic sample event](examples/failed-login.json)

## Quick start

1. Enable Windows Logon failure auditing and verify Event 4625 is generated.
2. Confirm the Wazuh agent collects the Security event channel.
3. Install the custom rules on the manager.
4. Validate the manager configuration, then restart it.
5. Perform controlled failed authentications using a dedicated lab account.
6. Verify rules 100201 and 100202 and inspect the source IP.

See [SETUP.md](docs/SETUP.md) for commands and the verification checklist.

## Rule logic

| Rule | Conditions | Result |
| --- | --- | --- |
| 100201 | Base rule 60122; Event 4625; Logon Type 3; nonempty, non-placeholder source address | Single network login failure |
| 100202 | Historical matches of 100201; same source IP; frequency 5; timeframe 120 | Possible brute-force activity |

Wazuh normally correlates frequency events within the same agent unless global frequency is enabled. This repository does not enable global correlation.

Repeated failures indicate suspicious activity, not a confirmed compromise. Stale credentials, saved passwords, scanners, and user mistakes can produce similar events. An IP shared by multiple users can aggregate their failures.

## Validation status

The author's original report records single-failure alerts, a multiple-failure alert, and a Windows firewall block. This repository is a cleaned-up reconstruction of that workflow, with neutral project naming and source-address filtering.

**The repository's revised files have not been executed against a live Windows/Wazuh environment in this publishing session.** Validate actual threshold behavior and event decoding before deployment. The sample JSON is synthetic and is not evidence of a successful live run.

Automatic firewall response is documented as follow-up work; this repository does not ship or enable an unverified blocking integration.

## MITRE ATT&CK

**T1110 — Brute Force** describes the behavior investigated by the repeated-failure alert. The alert alone does not establish password guessing, credential stuffing, or successful access.

## References

- [Wazuh rule syntax](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/rules.html)
- [Wazuh custom rules](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html)
- [Microsoft: Event 4625](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625)
- [MITRE ATT&CK: T1110](https://attack.mitre.org/techniques/T1110/)

Use controlled tests only on systems you own or are authorized to assess.
