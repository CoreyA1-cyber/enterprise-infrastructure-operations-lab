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


