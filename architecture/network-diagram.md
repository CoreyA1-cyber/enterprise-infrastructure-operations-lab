# Network Architecture — Enterprise Infrastructure Operations Lab

> Paste this block into `architecture/network-diagram.md`. GitHub renders Mermaid natively.

```mermaid
flowchart TB
    INET([Internet]) --- ISP["ISP Router<br/>WAN"]
    ISP --- FW

    subgraph EDGE["Perimeter"]
        FW["fw01 — pfSense CE 2.8.1<br/>Router-on-a-stick · 802.1Q trunk<br/>default deny · stateful<br/>OpenVPN server"]
    end

    FW === SW["sw01 — TP-Link Omada SG2210MP<br/>Managed PoE+ switch · VLAN trunk/access"]

    AP["ap01 — EAP660 HD<br/>Wireless AP · VLAN 1"] --- SW
    PVE["pve — Proxmox VE 9<br/>Dell OptiPlex · vmbr0 VLAN-aware"] === SW

    subgraph V1["VLAN 1 — Management · 192.168.10.0/24"]
        MGMT["Admin PC · Proxmox UI<br/>pfSense UI · switch UI"]
    end

    subgraph V20["VLAN 20 — Servers · 192.168.20.0/24"]
        RHEL["rhel01 · .20<br/>RHEL 10 · Apache<br/>Splunk UF · healthcheck timer"]
        SPL["splunk01 · .30<br/>Ubuntu 26.04 · Splunk Enterprise<br/>indexes: linux / windows / network"]
        DC["dc01 · .10<br/>Win Server 2022 · AD DS + DNS<br/>corp.internal · Splunk UF"]
    end

    subgraph V30["VLAN 30 — Users · 192.168.30.0/24"]
        WS["ws01 · DHCP<br/>Win 11 · domain-joined<br/>mapped S: drive via GPO"]
    end

    subgraph VPN["VPN — 192.168.99.0/24"]
        RA["Remote admin<br/>OpenVPN client"]
    end

    PVE -.hosts.-> RHEL
    PVE -.hosts.-> SPL
    PVE -.hosts.-> DC
    PVE -.hosts.-> WS

    MGMT --- SW
    RHEL --- SW
    SPL --- SW
    DC --- SW
    WS --- SW
    RA -.tunnel.-> FW

    RHEL -- "9997 TCP" --> SPL
    DC   -- "9997 TCP" --> SPL
    WS   -- "AD: DNS/Kerberos/LDAP" --> DC

    classDef vlan1 fill:#e8eef7,stroke:#4a6fa5,color:#1a2a3a
    classDef vlan20 fill:#e7f3ea,stroke:#3f8c5a,color:#14321f
    classDef vlan30 fill:#fdf0e3,stroke:#c3843a,color:#3a2410
    classDef vpn fill:#f3e9f5,stroke:#8a4f9e,color:#2e1534
    classDef infra fill:#eceff1,stroke:#5a6570,color:#1c2228
    class MGMT vlan1
    class RHEL,SPL,DC vlan20
    class WS vlan30
    class RA vpn
    class FW,SW,AP,PVE infra
```

## Traffic policy summary (enforced on pfSense)

| Source | Allowed to | Purpose |
|---|---|---|
| VLAN 1 (Mgmt) | all VLANs | administration |
| VLAN 20 (Servers) | internet, each other | updates, log forwarding |
| VLAN 30 (Users) | dc01 (AD ports only), internet | domain services, browsing |
| VLAN 30 (Users) | rest of VLAN 20 | **blocked** (default deny) |
| VPN (99) | rhel01/splunk01 admin ports | remote administration |

Default posture is **deny**; every allow is an explicit, scoped rule.