# Change Record: Lab Network Recovery and Rebuild
**Date:** 2026-09-29 to 2026-09-30
**Author:** Corey Armstrong
**Type:** Emergency recovery followed by planned rebuild

## Summary
The lab was unreachable after a long period offline. Troubleshooting from the physical layer up identified an unpowered core switch as the root cause. The firewall was then factory reset and rebuilt to a new design: three network segments, centralized DHCP, and certificate-based VPN for remote administration.

## Timeline
| # | Event | Finding / Action | Verification |
|---|---|---|---|
| 1 | Proxmox web UI timed out at 192.168.1.50 | Timeout (not refused) indicated no response from the host at all | — |
| 2 | Root password unknown on Proxmox console | Reset via boot loader (`init=/bin/bash`), remounted root rw, set new password. Initial login failure caused by entering `admin` instead of `root`. | Logged in as root |
| 3 | `ip -br a` / `ip route` on Proxmox | `nic0` and `vmbr0` DOWN, routes `linkdown`; host was actually at 192.168.10.50, not .1.50 | See validation evidence |
| 4 | `ethtool nic0` | `Link detected: no` | — |
| 5 | Switch inspected | No power/lights. **Root cause.** Power restored. | `Link detected: yes` |
| 6 | Cabling redesigned | ISP router → fw01 WAN directly; removed ISP-router-to-switch link so the firewall is the only path in/out | Physical inspection |
| 7 | pfSense factory reset | Clean known state; interfaces reassigned (WAN `igb0`, LAN `igb1`) | Console |
| 8 | LAN link intermittent | Restored after reseating cable. Suspected loose connector / bad cable; cause unconfirmed. | Link up |
| 9 | LAN moved to 192.168.10.1/24 | Default LAN (192.168.1.1) overlapped the WAN subnet | WAN .1.151 / LAN .10.1 |
| 10 | Workstation had address .150 outside DHCP range | Stale static IP from previous lab; switched adapter to DHCP | Lease .100 from fw01 |
| 11 | Setup wizard | Hostname `fw01`; ISP DNS override removed; time zone set; admin password changed | Dashboard |
| 12 | AES-NI enabled | Hardware crypto for VPN performance | Dashboard: active |
| 13 | Package manager failed to load | `pkg-static -d update` refreshed catalog. Probable cause: initial fetch failed during unstable WAN link (unconfirmed). | Packages listed |
| 14 | OpenVPN remote access deployed | Internal CA, server + user certs, Remote Access (cert + password), tls-crypt, strict CN, split tunnel. Initially misconfigured as Peer-to-Peer; corrected. | Connected; routes verified |
| 15 | WAN policy changed | Blanket RFC1918 block replaced by single pass rule: UDP 1194 from 192.168.1.0/24 only | Rule list |
| 16 | VLANs 20 / 30 created | pfSense subinterfaces, DHCP for Users, segmentation rules | Rule review |
| 17 | Rule review finding | SERVERS block rules scoped to "SERVERS address" (firewall IP only) instead of "SERVERS subnets"; would have bypassed segmentation. Corrected. | Rule list |
| 18 | Switch VLANs configured | VLAN 20 tagged ports 2–3; VLAN 30 tagged ports 1–3; VLAN 1 unchanged | Switch VLAN table |
| 19 | DHCP range reset by wizard | LAN pool reverted to .10–.245; corrected to .100–.200; reservations added for sw01/ap01 | Lease table |

## Known deviations / follow-ups
- VPN server certificate issued with 10-year lifetime (best practice ~1 year). Reissue planned.
- fw01 WAN address (192.168.1.151) depends on ISP router DHCP; reservation not available on app-managed router. If VPN fails, check WAN IP first.
- pfSense 2.8.1 available and ISC DHCP end-of-life: candidates for a formal change (MOP).
- Four legacy VMs on pve pending review/removal.

## Rollback
Configuration backups of fw01 taken at each known-good state (stored offline, not in this repository).