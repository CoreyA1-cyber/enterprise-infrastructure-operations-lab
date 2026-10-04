# Enterprise Infrastructure Operations Lab

A physical, multi-VLAN enterprise environment built and operated end to end: segmented network, firewall, VPN, virtualization, Linux and Windows servers, Active Directory, and centralized logging — then deliberately broken, troubleshot, and restored under documented procedures.

This repo is run like a production environment, not a tutorial. Every change has a record, every incident has evidence, and every fix is validated.

---

## What this demonstrates

- **Network segmentation** — 3 VLANs with 802.1Q trunking, default-deny firewall, scoped inter-VLAN rules
- **Firewall & VPN** — pfSense router-on-a-stick, stateful rules, OpenVPN remote access
- **Virtualization** — Proxmox VE hosting all server and client VMs
- **Linux administration** — RHEL 10: LVM, SELinux, firewalld, key-only SSH, sudo delegation, systemd timers, backups
- **Windows / Active Directory** — Server 2022 domain, OUs, AGDLP group model, GPO, a department file share
- **Monitoring & detection** — Splunk Enterprise with forwarders, synthetic checks, and brute-force / group-change alerts
- **Operations discipline** — incident records, RCAs, change management (MOPs), remote-hands procedures, a ticket + KB article

---

## Environment

| Host | Role | OS | Network |
|---|---|---|---|
| fw01 | Firewall / router / VPN | pfSense CE 2.8.1 | trunk (VLANs 1/20/30), VPN 192.168.99.0/24 |
| sw01 | Managed switch | TP-Link Omada SG2210MP | VLAN trunk + access |
| ap01 | Wireless access point | TP-Link EAP660 HD | VLAN 1 |
| pve | Virtualization host | Proxmox VE 9 | VLAN-aware bridge |
| rhel01 | Web + Linux services | RHEL 10 | VLAN 20 · .20 |
| splunk01 | SIEM / log server | Ubuntu 26.04 | VLAN 20 · .30 |
| dc01 | Domain controller + DNS | Windows Server 2022 | VLAN 20 · .10 |
| ws01 | Domain workstation | Windows 11 | VLAN 30 · DHCP |

**Addressing:** Mgmt `192.168.10.0/24` · Servers `192.168.20.0/24` · Users `192.168.30.0/24` · VPN `192.168.99.0/24`
**AD domain:** `corp.internal`

See [`architecture/network-diagram.md`](architecture/network-diagram.md) for the topology.

---

## Incidents, changes & tickets

| Record | Summary |
|---|---|
| [INC-0001](incident-records/INC-0001-rhel01-http-outage.md) | Web outage from a removed firewalld service; detected by Splunk in <2 min, traced via sudo logs, restored |
| [INC-0002](incident-records/INC-0002-ssh-bruteforce.md) | SSH authentication flood detected in Splunk, contained with a scoped host-firewall block |
| [CHG-0001](change-records/CHG-0001-pfsense-upgrade-mop.md) | pfSense upgrade under a MOP with baseline, boot-environment rollback, and validation |
| [RH-0001](change-records/RH-0001-remote-hands-verification.md) | Remote-hands: MAC-table port mapping, cabling labels, AP port move with rollback |
| [TKT-0001](incident-records/TKT-0001-user-transfer-share-access.md) | Transferred user can't reach a share → AGDLP group fix → [KB-0001](knowledge-base/KB-0001-department-transfer-access.md) |

---

## Repo layout

```
architecture/     IP plan, asset inventory, network diagram, failure scenarios
build-guides/      step-by-step build docs per host
runbooks/          repeatable operational procedures
change-records/    MOPs, remote-hands, change timelines
incident-records/  incidents, RCAs, tickets
knowledge-base/    KB articles
validation/        test evidence per system
scripts/           Bash (Linux) and PowerShell (Windows) automation
lessons-learned/   what broke, what was learned
screenshots/       captioned evidence
```

---

## Skills index

**Networking:** VLANs · 802.1Q · pfSense · OpenVPN · DHCP · DNS · packet capture
**Linux:** RHEL 10 · systemd · LVM · SELinux · firewalld · SSH hardening · Bash
**Windows:** Server 2022 · Active Directory · GPO · AGDLP · PowerShell
**Monitoring:** Splunk · universal forwarders · SPL · alerting
**Operations:** incident response · RCA · change management · documentation

---

*Lab built and operated by Corey Armstrong. Simulated incidents and test data are labeled as such throughout.*