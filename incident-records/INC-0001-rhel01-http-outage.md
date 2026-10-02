# INC-0001: rhel01 web service unavailable

| Field | Value |
|---|---|
| Opened | 2026-10-02 10:34:00 EDT |
| Detected by | Splunk alert "rhel01 HTTP service down" |
| Severity | High (customer-facing service unavailable) |
| Status | Resolved |
| Assigned | NOC / Corey Armstrong |

## Pre-incident state
status 200 state=UP

## Symptom
splunk01 to http://192.168.20.20/ returning status 000 (no HTTP response received) with state=DOWN since 10:34. Previous checks returned 200/UP.

## Scope and impact
- Affected: HTTP (TCP 80) on rhel01 (192.168.20.20). Web content unavailable to all network clients.
- Not affected: host is up; SSH (TCP 22) reachable; Ping succeeded; DNS healthy; httpd service running and serving locally; other servers unaffected.
- User impact: all users of the rhel01 web service.

## Investigation log
| Time | Command / check | Result | Conclusion |
|---|---|---|---|
| 10:35 | `ping -c 3 192.168.20.20` (from splunk01) | Succeeded |Host reachable at the network layer; outage is not a host-down event. |
| 10:36 | `curl -v -m 5 http://192.168.20.20` (from splunk01) | *   Trying 192.168.20.20:80...
* connect to 192.168.20.20 port 80 from 192.168.20.30 port 42370 failed: No route to host
* Failed to connect to 192.168.20.20 port 80 after 0 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to 192.168.20.20 port 80 after 0 ms: Could not connect to server | It suggest that a firewall rule is blocking connection. |

## Resolution
Re-added the `http` service to firewalld's permanent configuration and reloaded:
- 11:04:22 `firewall-cmd --permanent --add-service=http`
- 11:04:42 `firewall-cmd --reload`
- Verified: `firewall-cmd --list-services` → `http ssh`

## Validation
- `curl` from splunk01 → 200
- `sudo traceroute -T -p 80 192.168.20.20` → completes without `!X`
- Splunk svc_check → 200/UP from <T3> onward
- No further alerts

## Timeline
| Time | Event |
|---|---|
| 10:32:20 | Firewall change takes effect (http removed) |
| 10:33:50 | Synthetic check records 000/DOWN |
| 10:34:00 | Alert triggered; ticket opened |
| 10:35–10:56 | Investigation (see log) |
| 10:56 | Root cause identified |
| 11:04:42 | Fix applied |
| 11:05:00 | Service confirmed UP |

**Status:** Resolved · **RCA:** [RCA-0001](RCA-0001-rhel01-http-outage.md)

## Evidence
```
$ curl -v -m 5 http://192.168.20.20
*   Trying 192.168.20.20:80...
* connect to 192.168.20.20 port 80 from 192.168.20.30 port 42370 failed: No route to host
* Failed to connect to 192.168.20.20 port 80 after 0 ms: Could not connect to server
curl: (7) Failed to connect to 192.168.20.20 port 80 after 0 ms: Could not connect to server
```