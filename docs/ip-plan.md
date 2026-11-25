# IP Plan

Status: Phase 0 draft. Addresses marked **(plan)** are my choices; everything else comes straight from the phase guide.

## Address blocks

| Block | Use |
|---|---|
| 10.10.0.0/23 | HQ LANs (10.10.0.0 to 10.10.1.255) |
| 10.10.2.0/24 | Branch LANs |
| 10.10.255.0/24 | Infrastructure: loopbacks and the HQ transit link |
| 192.0.2.0/24 | Our public block (TEST-NET-1), advertised in BGP as AS 64500 |
| 198.51.100.0/24 | Provider point-to-point links (TEST-NET-2), six /30s |
| 203.0.113.16/30 | Public web server segment behind INET (TEST-NET-3) |

## HQ VLANs (VLSM)

HSRP group number = VLAN ID. VIP is the first usable address. R1 = priority 110 + preempt, R2 = 100.

| VLAN | Name | Subnet | Mask | VIP | HQ-R1 | HQ-R2 | DHCP excluded |
|---|---|---|---|---|---|---|---|
| 10 | USERS | 10.10.0.0 | /25 | .1 | .2 | .3 | .1-.10 |
| 20 | ADMIN | 10.10.0.128 | /26 | .129 | .130 | .131 | .129-.138 |
| 30 | IT | 10.10.0.192 | /27 | .193 | .194 | .195 | .193-.198 |
| 40 | GUEST | 10.10.1.0 | /27 | .1 | .2 | .3 | .1-.6 |
| 50 | SERVERS | 10.10.1.32 | /28 | .33 | .34 | .35 | static, no DHCP pool |
| 99 | MGMT | 10.10.1.48 | /28 | .49 | **.52 (plan)** | **.53 (plan)** | static, no DHCP pool |
| 998 | NATIVE | none | | | | | unused, no IP |
| 999 | PARKING | none | | | | | unused ports, shut |

VLAN names and the VLAN-to-subnet mapping are my assumption. Free for growth: 10.10.1.64 to 10.10.1.255.

## Branch VLANs

Single router, so no HSRP. The gateway is BR-R1.

| VLAN | Name | Subnet | Mask | Gateway (BR-R1) |
|---|---|---|---|---|
| 110 | BR-STAFF | 10.10.2.0 | /25 | .1 |
| 140 | BR-GUEST | 10.10.2.128 | /26 | .129 |
| 199 | BR-MGMT | 10.10.2.192 | /28 | .193 |
| 998 / 999 | native / parking | none | | |

## Infrastructure

| Item | Address |
|---|---|
| HQ-R1 Loopback0 | 10.10.255.1/32 |
| HQ-R2 Loopback0 | 10.10.255.2/32 |
| BR-R1 Loopback0 **(plan)** | 10.10.255.3/32 |
| HQ transit HQ-R1 Gi0/2 / HQ-R2 Gi0/2 | 10.10.255.65 / 10.10.255.66 (10.10.255.64/30) |

## Static hosts and device management

| Host | Address | Gateway |
|---|---|---|
| DNS and DHCP server | 10.10.1.36/28 | 10.10.1.33 |
| Intranet web server (intranet.local) | 10.10.1.37/28 | 10.10.1.33 |
| PC-MGMT | 10.10.1.54/28 **(plan)** | 10.10.1.49 |
| HQ-SW1 SVI VLAN 99 | 10.10.1.50 | 10.10.1.49 |
| HQ-SW2 SVI VLAN 99 | 10.10.1.51 | 10.10.1.49 |
| BR-SW SVI VLAN 199 **(plan)** | 10.10.2.194 | 10.10.2.193 |

## Provider side

| Link | Subnet | Side A | Side B |
|---|---|---|---|
| ISP-A to HQ-R1 | 198.51.100.0/30 | ISP-A Gi0/0 .1 | HQ-R1 Gi0/1 .2 |
| ISP-B to HQ-R2 | 198.51.100.4/30 | ISP-B Gi0/0 .5 | HQ-R2 Gi0/1 .6 |
| ISP-B to BR-R1 | 198.51.100.8/30 | ISP-B Gi0/1 .9 | BR-R1 Gi0/1 .10 |
| ISP-A to INET | 198.51.100.12/30 | ISP-A Gi0/1 .13 | INET Gi0/0 .14 |
| ISP-B to ISP-B-UP | 198.51.100.16/30 | ISP-B Gi0/2 .17 | ISP-B-UP Gi0/0 .18 |
| ISP-B-UP to INET | 198.51.100.20/30 | ISP-B-UP Gi0/1 .21 | INET Gi0/1 .22 |
| INET to web server | 203.0.113.16/30 | INET Gi0/2 .17 | Web server .18 |

INET Loopback0 = 8.8.8.8/32. DNS: publicwebsite.com = 203.0.113.18, intranet.local = 10.10.1.37.

## NAT pool

| Router | Pool address |
|---|---|
| HQ-R1 | 192.0.2.1 |
| HQ-R2 | 192.0.2.2 |
| BR-R1 | PAT on Gi0/1 (198.51.100.10) |

## BGP

| AS | Device |
|---|---|
| 64500 | HQ-R1, HQ-R2 (our network; each router runs eBGP only to its own ISP, no iBGP) |
| 64496 | ISP-A |
| 64497 | ISP-B |
| 64499 | ISP-B-UP (upstream between ISP-B and INET; makes the ISP-B path one AS hop longer) |
| 64498 | INET |

Advertised by us: 192.0.2.0/24 (backed by a Null0 route on both edge routers).
Advertised by INET: 8.8.8.8/32 and 203.0.113.16/30.
Advertised by ISP-B: 198.51.100.8/30, so INET can return traffic to the branch PAT address.
No ISP sends a default route (Packet Tracer has no `default-originate`).

### Expected paths

| Direction | Via ISP-A | Via ISP-B | Winner |
|---|---|---|---|
| INET to 192.0.2.0/24 | 64496 64500 (2 hops) | 64499 64497 64500 (3 hops) | ISP-A |
| HQ-R1 to 8.8.8.8 | 64496 64498 (2 hops) | not learned by HQ-R1 | ISP-A (HQ-R1 is the HSRP active router) |
| HQ-R2 to 8.8.8.8 | not learned by HQ-R2 | 64497 64499 64498 (3 hops) | ISP-B, used only after ISP-A is lost (traffic arrives via the floating default on HQ-R1) |

## Switch port map (suggested)

| Device | Port | Connects to | Mode / VLAN |
|---|---|---|---|
| HQ-SW1 | Gi0/1 | HQ-R1 Gi0/0 | trunk |
| HQ-SW1 | Fa0/23-24 | HQ-SW2 Fa0/23-24 | LACP Po1, trunk |
| HQ-SW1 | Fa0/1 / Fa0/2 | PC-USER1 / PC-ADMIN | access 10 / 20 |
| HQ-SW1 | Fa0/3 / Fa0/4 | DNS-DHCP server / intranet web | access 50 |
| HQ-SW2 | Gi0/1 | HQ-R2 Gi0/0 | trunk |
| HQ-SW2 | Fa0/1 / Fa0/2 | PC-USER2 / PC-IT | access 10 / 30 |
| HQ-SW2 | Fa0/3 / Fa0/4 | PC-GUEST / PC-MGMT | access 40 / 99 |
| BR-SW | Gi0/1 | BR-R1 Gi0/0 | trunk |
| BR-SW | Fa0/1 / Fa0/2 | PC-BR-STAFF / PC-BR-GUEST | access 110 / 140 |
| BR-SW | Fa0/3 / Fa0/4 | AP-STAFF / AP-GUEST (Phase 7) | access 110 / 140 |

## Checks done on this plan

- No overlapping subnets. All HQ subnets sit inside 10.10.0.0/23; branch is outside it.
- Every static host falls inside its VLAN's usable range.
- Six provider /30s and the web segment do not overlap.
