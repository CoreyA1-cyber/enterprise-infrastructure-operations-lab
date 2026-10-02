# RCA-0001: rhel01 Web Service Outage

## Summary
On 2026-10-02, the rhel01 web service (HTTP, TCP 80) was unreachable from the network for 33m. The cause was a host firewall change that removed the `http` service from firewalld. Service was restored by re-adding the rule.

## Impact
- Service: HTTP on rhel01 (192.168.20.20)
- Duration: 10:32:20 to 11:05:00 (33m)
- Users affected: all network clients of the web service
- Not affected: host availability, SSH, DNS, other servers

## Timeline
| Time | Event | Source |
|---|---|---|
| 10:32:20 | `firewall-cmd --permanent --remove-service=http` + reload executed by <user> | Splunk `linux_secure` (sudo log) |
| 10:33:50 | Synthetic check records status 000 / DOWN | Splunk `svc_check` |
| 10:34:00 | Alert "rhel01 HTTP service down" triggered | Splunk Triggered Alerts |
| 10:35 | Investigation began | INC-0001 |
| 10:35 | Root cause identified: http missing from firewalld | `firewall-cmd --list-all` |
| 11:04:03 | Fix applied: http re-added, firewalld reloaded | rhel01 |
| 11:05:00 | Synthetic check returns 200 / UP | Splunk `svc_check` |


## Root cause
The `http` service was removed from rhel01's firewalld permanent configuration and the firewall was reloaded, blocking inbound TCP 80. httpd itself remained healthy throughout.

## Why it happened (5 Whys)
1. Why was the site unreachable? → Inbound TCP 80 was rejected by the host firewall.
2. Why was it rejected? → The `http` service had been removed from firewalld.
3. Why was it removed? → A configuration change was made directly on the server.
4. Why wasn't it caught before impact? → The change was made outside any change-control process, with no review or validation step.
5. Why was it hard to trace? → firewalld does not log configuration changes itself; the only record was the sudo command log.

## What went well
- Synthetic monitoring from a separate host detected the outage automatically within  minutes.
- Layered troubleshooting isolated the fault quickly (service healthy and working locally → host firewall).
- Centralized logging preserved the sudo record of the change, establishing who/what/when.

## What could be improved
- No alert existed for firewall configuration changes.
- The rhel01 backup did not include firewalld configuration.
- Misleading signals ("No route to host", UDP traceroute `!X`) required interpretation. Documented in the runbook.

## Corrective and preventive actions
| # | Action | Type | Status |
|---|---|---|---|
| 1 | Require a change record / MOP for firewall changes on production hosts | Preventive | Planned (CHG-0001 establishes the process) |
| 2 | Splunk alert on `firewall-cmd` execution via sudo | Detective | <Done/Planned> |
| 3 | Add `/etc/firewalld` to the rhel01 backup script | Recovery | <Done/Planned> |
| 4 | Runbook: troubleshooting an unreachable web service | Response | <Done/Planned> |