# Screenshots

Evidence supporting the build and validation of this lab. Each image is captioned with what it shows and why it matters.

**Sanitization:** Public IP addresses, the Netgate Device ID, serial numbers, and personal browser details have been redacted with solid boxes. Private RFC1918 addresses (192.168.x.x) are intentionally left visible because they are part of the documented design. No private keys, TLS keys, or certificate contents appear in any image.

---

## Phase 1: Network Foundation (pfSense)

| # | File | Caption |
|---|---|---|
| 01 | [01-pfsense-dashboard.png](01-pfsense-dashboard.png) | pfSense dashboard for `fw01`: all four interfaces (WAN, LAN, SERVERS, USERS) up at 1 Gbps; DNS limited to local resolver, 1.1.1.1, and 9.9.9.9 (ISP DNS override removed); AES-NI hardware crypto active. |
| 02 | [02-pfsense-interface-assignments.png](02-pfsense-interface-assignments.png) | Interface assignments: WAN on `igb0`, LAN (Management) on `igb1`, SERVERS and USERS as VLAN 20 and 30 subinterfaces on `igb1` (router-on-a-stick). |
| 03 | [03-pfsense-vlans.png](03-pfsense-vlans.png) | 802.1Q VLANs 20 (SERVERS) and 30 (USERS) defined on the LAN parent interface. |
| 04 | [04-pfsense-dhcp-lan.png](04-pfsense-dhcp-lan.png) | Management DHCP pool set to `.100–.200`, kept clear of statically assigned infrastructure addresses. |
| 05 | [05-pfsense-dhcp-static-mappings.png](05-pfsense-dhcp-static-mappings.png) | DHCP reservations tying the switch (`sw01`, .10.2) and access point (`ap01`, .10.3) to fixed addresses by MAC. |
| 06 | [06-pfsense-dhcp-users.png](06-pfsense-dhcp-users.png) | USERS VLAN DHCP pool `192.168.30.100–.200` for end-user devices. Servers VLAN intentionally has no DHCP (static only). |

## Phase 2: Firewall Policy

| # | File | Caption |
|---|---|---|
| 07 | [07-pfsense-rules-wan.png](07-pfsense-rules-wan.png) | WAN rules: default deny inbound, with a single pass rule for OpenVPN (UDP 1194) from the home network only. RFC1918 blanket block replaced by this explicit rule. |
| 08 | [08-pfsense-rules-servers.png](08-pfsense-rules-servers.png) | SERVERS rules (first match wins): DNS to firewall allowed; firewall admin access and Management network blocked; internet allowed for updates. |
| 09 | [09-pfsense-rules-users.png](09-pfsense-rules-users.png) | USERS rules: DNS allowed; firewall, Management, and Servers blocked; internet allowed. Specific domain controller exceptions to be added for Active Directory. |
| 10 | [10-pfsense-rules-openvpn.png](10-pfsense-rules-openvpn.png) | OpenVPN rules: VPN clients (`192.168.99.0/24`) limited to ping and admin ports only, to Management and Servers networks only. |
| 11 | [11-pfsense-alias-admin-nets.png](11-pfsense-alias-admin-nets.png) | `ADMIN_NETS` alias grouping the Management and Servers subnets reachable over VPN. |
| 12 | [12-pfsense-alias-mgmt-ports.png](12-pfsense-alias-mgmt-ports.png) | `MGMT_PORTS` alias: SSH (22), HTTPS/pfSense web UI (443), Proxmox web UI (8006). |

## Phase 3: Remote Access VPN

| # | File | Caption |
|---|---|---|
| 13 | [13-pfsense-openvpn-server.png](13-pfsense-openvpn-server.png) | OpenVPN server in Remote Access (SSL/TLS + User Auth) mode on UDP 1194, tunnel network `192.168.99.0/24`, AES-GCM ciphers, SHA256. |
| 14 | [14-pfsense-cert-authority.png](14-pfsense-cert-authority.png) | Internal certificate authority `Lab-CA` (10-year lifetime) used to issue VPN server and user certificates. |
| 15 | [15-pfsense-certificates.png](15-pfsense-certificates.png) | Certificates issued by Lab-CA: a server-type certificate for the VPN server and a user-type certificate whose CN matches the VPN username (strict CN matching). Note: the server certificate was issued with a 10-year lifetime; a shorter lifetime is planned on reissue. |
| 16 | [16-pfsense-aes-ni.png](16-pfsense-aes-ni.png) | AES-NI CPU crypto acceleration enabled and active to support VPN encryption on a low-power CPU. |
| 17 | [17-proxmox-over-vpn.png](17-proxmox-over-vpn.png) | Proxmox VE web UI (`192.168.10.50:8006`) reached from the admin workstation over the VPN, with no direct cable to the lab. |

## Phase 4: Switching

| # | File | Caption |
|---|---|---|
| 18 | [18-switch-vlan-config.png](18-switch-vlan-config.png) | Omada SG2210MP VLAN configuration: VLAN 20 on ports 2–3 (Proxmox, pfSense) and VLAN 30 on ports 1–3 (AP, Proxmox, pfSense), tagged; VLAN 1 untagged Management on all ports. |
| 19 | [19-switch-port-pvid.png](19-switch-port-pvid.png) | All switch ports keep PVID 1, so untagged traffic stays in Management while VLAN 20/30 traffic is carried tagged. | 
| 20 | [20-vpn-client-split-tunnel.png](20-vpn-client-split-tunnel.png) | VPN client connected: TAP adapter at 192.168.99.2 with no default gateway; only lab subnets route through the tunnel. |
| 21 | [21-vpn-find-netroute.png](21-vpn-find-netroute.png) | Route check with privacy VPN active: lab traffic uses the lab VPN adapter, internet traffic uses the privacy VPN. |
| 22 | [22-proxmox-vmbr0-vlan-aware.png](22-proxmox-vmbr0-vlan-aware.png) | Proxmox bridge vmbr0 set to VLAN-aware, allowing VMs to be placed on VLAN 20/30 by tag. |
| 23 | [23-proxmox-rhel01-hardware.png](23-proxmox-rhel01-hardware.png) | rhel01 VM: CPU type host (required for RHEL 10), 40 GB OS disk + 10 GB data disk for LVM, NIC tagged VLAN 20, install media detached. |
| 24 | [24-proxmox-rhel01-summary.png](24-proxmox-rhel01-summary.png) | rhel01 summary: guest agent reporting 192.168.20.20 on the Servers VLAN. |
| 25 | [25-proxmox-splunk01-hardware.png](25-proxmox-splunk01-hardware.png) | splunk01 VM: 4 vCPU (host), 8 GB RAM, 80 GB disk, NIC tagged VLAN 20. |
| 26 | [26-splunk-indexes.png](26-splunk-indexes.png) | Custom indexes `linux`, `network`, and `windows`, each capped at 20 GB to prevent disk exhaustion on the 80 GB volume. |
| 27 | [27-splunk-rhel01-events.png](27-splunk-rhel01-events.png) | rhel01 logs (secure, messages, httpd access/error) arriving via the Universal Forwarder. |
| 28 | [28-splunk-svc-check.png](28-splunk-svc-check.png) | Synthetic HTTP check from splunk01 to rhel01 every minute: status 200, ~1 ms response, state UP. |
| 29 | [29-splunk-alert-config.png](29-splunk-alert-config.png) | Alert "rhel01 HTTP service down": runs every minute over the last 2 minutes, triggers on any non-UP result, throttled 10 minutes. |


## Phase 5: rhel01 (Proxmox)

| # | File | Caption |
|---|---|---|
| 22 | [22-proxmox-vmbr0-vlan-aware.png](22-proxmox-vmbr0-vlan-aware.png) | Proxmox bridge vmbr0 set to VLAN-aware (change applied, not pending), allowing VMs to be placed on VLAN 20/30 by tag. |
| 23 | [23-proxmox-rhel01-hardware.png](23-proxmox-rhel01-hardware.png) | rhel01 VM: CPU type host (required for RHEL 10), 40 GB OS disk + 10 GB data disk for LVM, NIC tagged VLAN 20, install media detached. |

## Phase 6: Monitoring (splunk01)

| # | File | Caption |
|---|---|---|
| 25 | [25-proxmox-splunk01-hardware.png](25-proxmox-splunk01-hardware.png) | splunk01 VM: 4 vCPU (host), 8 GB RAM, 80 GB disk, NIC tagged VLAN 20. |
| 26 | [26-splunk-indexes.png](26-splunk-indexes.png) | Custom indexes linux, network, and windows, each capped at 20 GB to prevent disk exhaustion on the 80 GB volume. |
| 27 | [27-splunk-rhel01-events.png](27-splunk-rhel01-events.png) | rhel01 logs (secure, messages, httpd access/error) arriving via the Universal Forwarder. |
| 28 | [28-splunk-svc-check.png](28-splunk-svc-check.png) | Synthetic HTTP check from splunk01 to rhel01 every minute: status 200, ~1 ms response, state UP. |
| 29 | [29-splunk-alert-config.png](29-splunk-alert-config.png) | Alert "rhel01 HTTP service down": runs every minute over the last 2 minutes, triggers on any non-UP result, throttled 10 minutes. |

## Phase 6b: Change Management (CHG-0001)

| # | File | Caption |
|---|---|---|
| 30 | [30-pfsense-pre-upgrade-version.png](30-pfsense-pre-upgrade-version.png) | Pre-change baseline: fw01 running pfSense CE 2.7.2 with 2.8.1 available. |
| 31 | [31-pfsense-bectl-snapshot.png](31-pfsense-bectl-snapshot.png) | Rollback point: ZFS boot environment `pre-2.8.1-upgrade` created before the upgrade (copy-on-write, 8K at creation). |
| 32 | [32-pfsense-post-upgrade-version.png](32-pfsense-post-upgrade-version.png) | Post-change validation: fw01 running pfSense CE 2.8.1. Upgrade completed in ~13 minutes, no rollback required. |

## Phase 7: Remote Hands (physical photos)

Photos of actual equipment. Device identifiers redacted; location metadata removed.

| # | File | Caption |
|---|---|---|
| 33 | [33-photo-fw01-cabling-labels.png](33-photo-fw01-cabling-labels.png) | fw01: WAN (igb0) and LAN (igb1) cabled with link LEDs active; OPT ports unused. Temporary label identifies the WAN cable as FW01-WAN to ISP router. |
| 34 | [34-photo-sw01-port-labels.png](34-photo-sw01-port-labels.png) | sw01: ports 1–3 in use with link LEDs active, matching the documented port map (P01 ap01, P02 pve, P03 fw01 LAN). Temporary identification labels placed per cable. |
| 35 | [35-switch-mac-table.png](35-switch-mac-table.png) | sw01 MAC address table: each device's MAC learned on its documented port and VLAN, confirming the port map from switch data. |

## Phase 8 — Active Directory (dc01 / ws01)

| # | File | What it shows |
|---|---|---|
| 36 | `36-dc01-ipconfig.png` | dc01 static IP 192.168.20.10/24, gateway 192.168.20.1, gateway ping and DNS resolution verified before promotion |
| 37 | `37-dc01-ad-validation.png` | Forest `corp.internal` created; `dcdiag` DNS, Connectivity, Advertising, NetLogons, and Services tests passed after resolving first-boot DNS registration errors |
| 38 | `38-ad-ou-structure.png` | OU design: Corp → Users (IT, HR, Finance, Sales), Groups, Workstations, Servers, Admins |
| 39 | `39-ad-users-by-department.png` | 12 users across 4 departments, enabled, each in its `GG_<Dept>` global group |
| 40 | `40-share-ntfs-permissions.png` | `Departments` share with access-based enumeration; NTFS on Finance limited to `DL_Share_Finance_RW` (Modify), Domain Admins, SYSTEM (AGDLP) |
| 41 | `41-gpo-drive-map.png` | GPO `U_DriveMap_Departments` maps S: to `\\dc01.corp.internal\Departments` |
| 42 | `42-pfsense-users-ad-rules.png` | USERS rules: AD_TCP / AD_UDP pass to dc01 only, placed above the block to SERVERS |
| 43 | `43-capture-users-vs-servers.png` | DNS troubleshooting: queries from VLAN 30 arrived on USERS but not on SERVERS (dropped by pfSense); after the AD rules were activated, full query/reply on SERVERS |
| 44 | `44-ws01-domain-joined.png` | ws01 joined to `corp.internal` and placed in OU=Workstations (via `redircmp`) |
| 45 | `45-ws01-finance-s-drive.png` | Finance user `treed`: S: shows only Finance, GPO applied (`gpresult`) |

## Phase 8b — TKT-0001: Transferred user access

| # | File | What it shows |
|---|---|---|
| 46 | `46-tkt0001-access-denied.png` | `jcole` (moved to Finance OU) gets Access denied on the Finance folder |
| 47 | `47-tkt0001-effective-access.png` | Effective Access shows no permissions; user still in GG_Sales, not GG_Finance |
| 48 | `48-tkt0001-resolved.png` | After group swap and re-logon: `whoami /groups` shows GG_Finance, S: shows Finance, write test succeeds |

## Phase 8c — Windows logging to Splunk

| # | File | What it shows |
|---|---|---|
| 49 | `49-splunk-dc01-sourcetypes.png` | dc01 Security, System, and Application logs arriving in `index=windows` |
| 50 | `50-splunk-tkt0001-group-changes.png` | Events 4728/4729 record jcole added to GG_Finance and removed from GG_Sales, giving an audit trail for TKT-0001 |
| 51 | `51-splunk-group-change-alert-triggered.png` | Alert "AD security group membership change" fired on a controlled test (jcole added to and removed from GG_HR, events 4728/4729) |
## Phase 8c — Windows logging & detection (Splunk)

| # | File | What it shows |
|---|---|---|
| 51 | `51-splunk-group-change-alert-triggered.png` | Alert "AD security group membership change" fires on a controlled test (jcole added to / removed from GG_HR, events 4728/4729) |

## Phase 9 — INC-0002: SSH brute-force detection & response

| # | File | What it shows |
|---|---|---|
| 52 | `52-inc0002-splunk-bruteforce.png` | Splunk detection flags a single source (192.168.99.2) with 64 failed SSH authentications in ~35 seconds on rhel01 |
| 53 | `53-inc0002-contained.png` | Containment verified: SSH from the blocked source changes from "Permission denied" to "Connection timed out" after a scoped firewalld drop rule |

## Phase 10 — Server health monitoring

| # | File | What it shows |
|---|---|---|
| 54 | `54-healthcheck-output.png` | lab-healthcheck.sh report — disk, memory, load, services (incl. SplunkForwarder), failed SSH, listening ports, connectivity; WARN correctly flags the residual failed-SSH count from INC-0002 |
| 55 | `55-healthcheck-timer.png` | systemd timer running the health check every 15 minutes, plus the rotated report logs in /var/log/healthcheck/ |