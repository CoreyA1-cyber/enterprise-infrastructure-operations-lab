# Validation Evidence: Network Recovery (2026-09-29)

## 1. Proxmox host had no network link (outage)
`ip -br a` (as recorded): `lo UNKNOWN`, `nic0 DOWN`, `wlp1s0 DOWN`, `vmbr0 DOWN`

```
$ ip route
default via 192.168.10.1 dev vmbr0 proto kernel onlink linkdown
192.168.10.0/24 dev vmbr0 proto kernel scope link src 192.168.10.50 linkdown
```
**Shows:** Physical link down; host address was 192.168.10.50, not the 192.168.1.50 being browsed to.

## 2. Root cause confirmed and resolved
```
$ ethtool nic0 | grep "Link detected"
Link detected: no      # before: switch unpowered
Link detected: yes     # after: switch power restored
```

## 3. Stale static IP on admin workstation
Before (manual address from previous lab, outside DHCP range, DHCP disabled):
```
IPv4 Address. . . . . . . . . . . : 192.168.10.150
Subnet Mask . . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . . : 192.168.10.1
```
After switching the adapter to DHCP:
```
Connection-specific DNS Suffix  . : home.arpa
IPv4 Address. . . . . . . . . . . : 192.168.10.100
Subnet Mask . . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . . : 192.168.10.1
```
**Shows:** Lease issued by fw01 (domain suffix `home.arpa` is pfSense's default).

## 4. VPN split tunnel (direct cable unplugged during test)
```
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0      192.168.1.1     192.168.1.57     35
     192.168.10.0    255.255.255.0     192.168.99.1     192.168.99.2    281
```
**Shows:** Only the lab subnet routes through the tunnel; internet traffic uses the normal path. The direct cable was unplugged so the test could only pass through the VPN.