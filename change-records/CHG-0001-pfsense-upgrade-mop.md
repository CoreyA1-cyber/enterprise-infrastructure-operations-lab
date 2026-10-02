# CHG-0001: Method of Procedure, pfSense CE Upgrade 2.7.2 → 2.8.1

| Field | Value |
|---|---|
| Change ID | CHG-0001 |
| Requested by / Implementer | Corey Armstrong |
| Date | 2026-10-02 |
| Planned window | 12:45–13:30 EDT (45 min) |
| Change type | Normal: planned, with tested rollback point |
| Status | **Implemented: Successful** |

---

## 1. Purpose
Upgrade fw01 from pfSense CE 2.7.2 to 2.8.1 to remain on a supported release with current security fixes.

## 2. Scope
- **In scope:** pfSense software upgrade on fw01; reinstallation of installed package (openvpn-client-export).
- **Out of scope:** ISC DHCP → Kea migration (separate change, CHG-0002); firewall rule, interface, or VPN configuration changes.

## 3. Affected assets
| Asset | Expected impact |
|---|---|
| fw01 | Reboots during upgrade |
| All VLANs (Management, Servers, Users) | No inter-VLAN routing, internet, DHCP, or DNS (pfSense resolver) during reboot |
| OpenVPN admin access | Unavailable during reboot |
| rhel01 ↔ splunk01 | **Unaffected**: same VLAN; traffic does not traverse fw01 |

## 4. Expected impact
Full outage of routing, DNS, DHCP, and VPN for approximately 5–15 minutes. Existing DHCP leases remain valid.

## 5. Risk assessment
| Risk | Likelihood | Mitigation |
|---|---|---|
| Upgrade fails / system will not boot | Low | ZFS boot environment snapshot; console access attached |
| Package incompatibility (openvpn-client-export) | Low | Packages reinstall automatically; verified post-change |
| Configuration conversion issue (rules, VPN) | Low | Configuration backup; post-checks compared against baseline |
| Loss of remote administrative access | Expected | Change performed from Management port (sw01 port 4) with console fallback; VPN not used during change |

## 6. Pre-change checks (baseline)
Performed 2026-10-02, ~12:25–12:45 EDT.

| # | Check | Command / location | Baseline result |
|---|---|---|---|
| 1 | Version | Dashboard | 2.7.2-RELEASE; 2.8.1 offered |
| 2 | Interfaces | Status → Interfaces | WAN (igb0), LAN (igb1), SERVERS (igb1.20), USERS (igb1.30): all up, 1000baseT full-duplex, 0 errors |
| 3 | Gateways | Status → Gateways | WAN_DHCP 192.168.1.1 Online, 0% loss; WAN_DHCP6 Online |
| 4 | Rule count | `pfctl -sr \| wc -l` | 127 |
| 5 | Routing table | `netstat -rn -f inet` | Default via 192.168.1.1 (igb0); connected routes 192.168.10.0/24 (igb1), 192.168.20.0/24 (igb1.20), 192.168.30.0/24 (igb1.30), 192.168.99.0/24 (ovpns1) |
| 6 | DHCP | Status → DHCP Leases | Static mappings present: sw01 (192.168.10.2), ap01 (192.168.10.3) |
| 7 | OpenVPN | Status → OpenVPN | ovpns1 running; 1 client connected; AES-256-GCM |
| 8 | Installed packages | System → Package Manager | openvpn-client-export 1.9.2 (current) |
| 9 | Disk | Dashboard → Disks | 843 MB of 50 GB used (2%), ZFS |
| 10 | Configuration backup | Diagnostics → Backup & Restore | Downloaded; stored offline (not in repository) |
| 11 | Boot environment | `bectl list` | `pre-2.8.1-upgrade` created 12:42 (8K, copy-on-write); `default` active (NR) |
| 12 | Service checks (from splunk01) | `curl`, `ping`, `getent` | rhel01 HTTP 200; 192.168.20.1 and 192.168.10.1 reply; DNS resolves redhat.com |

## 7. Implementation steps
| # | Step | Time |
|---|---|---|
| 1 | Admin PC connected to sw01 port 4 (Management, untagged); DHCP address 192.168.10.x confirmed | 12:50:00 |
| 2 | VPN disconnected; HDMI console attached to fw01 as fallback | 12:51:00 |
| 3 | Pre-checks 1–12 completed and recorded | ~12:45 |
| 4 | System → Update → Update Settings → Branch set to Current Stable Version (2.8.1) | ~12:48 |
| 5 | Upgrade confirmed (2.7.2 → 2.8.1) | **12:50** |
| 6 | Download, install, and reboot; web UI polled "not yet ready" during first-boot tasks | 12:50–13:03 |
| 7 | Web UI available; logged in | **13:03** |

## 8. Validation (post-change)
| # | Check | Expected | Result |
|---|---|---|---|
| 1 | Version | 2.8.1-RELEASE | Success |
| 2 | Interfaces | All four up, 0 errors | Success |
| 3 | Gateways | WAN_DHCP Online | Success |
| 4 | Rule count | 127 (or differences explained) | Success: 127 |
| 5 | Routing table | Matches baseline | Success |
| 6 | DHCP | Static mappings present; admin PC received a lease | Success |
| 7 | OpenVPN | Server running; client reconnects from home network | **PASS**: VPN reconnected from home network after change |
| 8 | Packages | openvpn-client-export reinstalled | Success |
| 11 | Boot environments | New BE active; `pre-2.8.1-upgrade` retained | Success |
| 12 | Services | rhel01 HTTP 200; DNS resolves; Splunk Web reachable over VPN | Success |
| 13 | Intra-VLAN traffic during outage | splunk01 → rhel01 check stayed UP | Success: `index=linux sourcetype=svc_check earliest=-90m \| stats count by state`> |

## 9. Rollback plan (not required)
**Trigger:** any validation failure not resolvable within 15 minutes, or fw01 fails to boot.
1. `bectl activate pre-2.8.1-upgrade` → reboot (or console boot menu → Boot Environments → select `pre-2.8.1-upgrade`). Restores pfSense 2.7.2 with pre-change configuration.
2. If boot environments are unavailable: reinstall 2.7.2 and restore the configuration backup.
3. Re-run validation checks against the baseline.

## 10. Final status
| Field | Value |
|---|---|
| Outcome | **Successful**: rollback not required |
| Upgrade start | 12:50 EDT |
| Upgrade complete (web UI available) | 13:03 EDT |
| Total service impact | **~13 minutes** (within the 5–15 minute estimate) |
| Deviations from plan | Web UI unavailable longer than the initial reboot while first-boot upgrade tasks and package reinstallation completed (the progress page repeatedly showed "not yet ready"). Normal for a major-version upgrade on low-power hardware; monitored via console; no action required. |
| Findings (out of scope) | Splunk Web was not reachable from the Management LAN during the change because splunk01's ufw restricts TCP 8000 to the VPN subnet. Working as configured, but it leaves no admin path to Splunk when the VPN is unavailable.  |

## 11. Follow-ups
| # | Item | Type |
|---|---|---|
| 1 | CHG-0002: migrate DHCP backend from ISC DHCP (end-of-life) to Kea | Planned change |
| 2 | Decide whether to permit Management (192.168.10.0/24) to splunk01 TCP 22/8000 as a break-glass path | Design decision |
| 3 | Remove `pre-2.8.1-upgrade` boot environment after a 7-day observation period | Housekeeping |
| 4 | Reissue the VPN server certificate with a shorter lifetime (current: 10 years) | Planned change |