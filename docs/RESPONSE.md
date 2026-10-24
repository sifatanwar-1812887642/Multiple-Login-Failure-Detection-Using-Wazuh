# Response notes

The original implementation report describes a Windows Firewall block following the multiple-failure alert. Its supplied code and configuration are incomplete as a reusable integration:

- The PowerShell script expects a SourceIP argument, but its wrapper does not supply it or parse Wazuh's input.
- A configured timeout does not by itself remove a firewall rule. The script must implement the response protocol and undo behavior.
- Matching Logon Type 3 does not establish the destination service. Blocking one service's port cannot be assumed appropriate for every network logon failure.

For these reasons, this repository delivers detection and verification instructions without enabling automatic blocking.

## Analyst workflow

1. Inspect source IP, target account, endpoint, timestamps and failure status.
2. Check whether stale credentials or expected automation explain the failures.
3. Investigate any subsequent successful authentication from the same source and account.
4. Select a proportionate response for the service actually involved.
5. Record the action and verify recovery.

## Requirements for a future automated response

Use the documented Wazuh custom active response protocol. Parse and validate the source address, account for management and trusted sources, handle duplicate requests, limit the firewall rule's scope, and implement both add and delete actions if using a timeout. Test failures, recovery and rollback on an isolated endpoint before enabling it.

Reference: [Wazuh custom active response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/custom-active-response-scripts.html).
