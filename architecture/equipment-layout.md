# Equipment Layout (positions assigned, no rack (yet))

| Position | Device | Power source | Network uplink |
|---|---|---|---|
| shelf 1, left | ISP router (Spectrum) | UPS | WAN → fw01 igb0 |
| Above access point, center | fw01 (pfSense) | UPS | igb0 → ISP; igb1 → sw01 P3 |
| Below access point, right | sw01 (SG2210MP) | UPS | P1 ap01, P2 pve, P3 fw01 |
| Small table | pve (OptiPlex) | UPS | nic0 → sw01 P2 |
| Above switch | ap01 (EAP660 HD) | PoE from sw01 P1 | P1 |