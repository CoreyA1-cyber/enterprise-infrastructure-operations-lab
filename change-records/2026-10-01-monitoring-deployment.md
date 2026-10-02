# Change Record: Central Logging and Monitoring Deployment
**Date:** 2026-10-01
**Author:** Corey Armstrong
**Type:** Standard change (new service)

## Summary
Deployed splunk01 (Splunk Enterprise on Ubuntu 22.04.5) on the Servers VLAN, forwarded rhel01 logs via the Universal Forwarder, and added a synthetic HTTP check with an alert to detect web service outages.

## Changes
| # | Change | System |
|---|---|---|
| 1 | Created VM 130 splunk01 (4 vCPU, 8 GB, 80 GB, VLAN 20) | pve |
| 2 | Installed Splunk Enterprise; enabled receiving on 9997; created indexes linux/network/windows (20 GB cap) | splunk01 |
| 3 | Host firewall: ufw rules scoped by source network | splunk01 |
| 4 | Added TCP 8000 to MGMT_PORTS alias | fw01 |
| 5 | Installed Universal Forwarder; outputs/inputs configured | rhel01 |
| 6 | Read-only ACLs for `splunkfwd` on log files; logrotate postrotate hooks | rhel01 |
| 7 | Synthetic HTTP check script + 1-minute systemd timer; Splunk input | splunk01 |
| 8 | Alert "rhel01 HTTP service down" | splunk01 |

## Issues encountered
| Issue | Cause | Resolution |
|---|---|---|
| Ubuntu ISO download returned 76 KiB `text/html` | Copied the download page URL, not the file URL | Caught by checking size/type before download; used existing 22.04.5 server ISO |
| `dpkg` could not find the Splunk package | Windows `.msi` downloaded instead of Linux `.deb` | Removed `.msi`; downloaded the Linux `.deb` |
| Log paths not found | Command run on splunk01 instead of rhel01 | Verified hostname in prompt; named terminal tabs per host |
| Forwarder boot-start command left suspended (Ctrl+Z) | Job paused at a prompt, later terminated | Verified `systemctl is-enabled SplunkForwarder` = enabled |
| Index max size defaulted to 500 GB on an 80 GB disk | Default setting | Capped each custom index at 20 GB |

## Rollback
Stop and disable `Splunkd` / `SplunkForwarder`, `http-check.timer`; remove 8000 from MGMT_PORTS; revert logrotate hooks; delete VM 130.

## Validation
See `validation/splunk01-validation.md` and screenshots 25–29.