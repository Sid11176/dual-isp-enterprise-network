# Design Decisions

## Decisions

| # | Decision | Why |
|---|---|---|
| D1 | Dual-homed HQ (ISP-A primary, ISP-B backup), single-homed branch on ISP-B | Shows dual-homed eBGP at HQ and a realistic single-ISP branch |
| D2 | OSPF only inside HQ; BGP is not redistributed into OSPF | Keeps internet routes out of the IGP; the HQ routers reach each other's exit through a floating static default (D4) |
| D3 | No iBGP. HQ-R1 and HQ-R2 each run eBGP only to their own ISP | Packet Tracer 9 supports external BGP neighbors only. Loopbacks stay as OSPF and BGP router IDs |
| D4 | Outbound exit via ISP-A because HQ-R1 is HSRP active. Floating static default (AD 250) on HQ-R1 via HQ-R2 (and the reverse on HQ-R2) covers an ISP failure | Without iBGP or BGP policy, a floating static route is how the other edge router becomes the exit when the BGP routes disappear |
| D5 | Inbound preference for ISP-A: add ISP-B-UP (AS 64499) between ISP-B and INET so the path via ISP-B is longer | Packet Tracer has no route-map, so no prepend. A longer real path gives the same result and keeps NAT return traffic on HQ-R1 |
| D6 | HSRP v2, group number = VLAN ID, R1 priority 110 with preempt | Predictable gateway; HQ-SW1 is STP root so root and HSRP active are on the same side |
| D7 | LACP EtherChannel between HQ switches | Redundant link that is one logical trunk to STP |
| D8 | Rapid PVST+ with explicit priorities for VLANs 10, 20, 30, 40, 50, 99: HQ-SW1 24576 (root), HQ-SW2 28672 (secondary) | Fast convergence; same result as `root primary` / `root secondary` but deterministic |
| D9 | Native VLAN 998 on trunks; unused ports in VLAN 999 and shut | Standard Layer 2 hardening |
| D10 | PAT on each edge router, using 192.0.2.1 (R1) and 192.0.2.2 (R2) | Public pool is inside the block we advertise |
| D11 | Named extended ACLs inbound on HQ subinterfaces | Policy sits next to the source; hit counts are easy to read |
| D12 | SSH only, local user, VTY access-class limited to MGMT and IT | Basic management-plane hardening |
| D13 | Branch uses a local DHCP pool and PAT in Phases 1 to 5; IPsec VPN added in Phase 6 | Keeps Tier 1 shippable without crypto |
| D14 | ISPs advertise INET's prefixes with `network` statements; no default route from the ISPs | Packet Tracer has no `default-originate`; HQ only needs 8.8.8.8/32 and 203.0.113.16/30 |
| D15 | NOC-SRV (10.10.1.55, VLAN 99) is the syslog, NTP and TFTP backup target; added in Phase 4 | Shows operations tooling; if Packet Tracer cannot do it, fall back to `show logging` buffers |
| D16 | Port security on active access ports: sticky, maximum 1, violation shutdown (Phase 4) | Pairs with BPDU guard and parked unused ports |
| D17 | The branch VPN peers on HQ-R2 (the ISP-B router) with mirrored crypto ACLs (HQ 10.10.0.0/23, branch 10.10.2.0/24) and a NAT exemption at the top of each NAT ACL (Phase 6) | The branch only has ISP-B; HQ-R2 already faces ISP-B |

## Plan review findings (resolved in Phase 0)

Findings from checking the original phase guide against the IP plan. All were accepted on 25 Nov 2026 and applied in `ip-plan.md`.

| # | Finding | Resolution |
|---|---|---|
| R1 | The guide uses 10.10.0.0/16 for HQ (NAT ACL, Phase 6 crypto ACL), but the branch 10.10.2.0/24 sits inside it | HQ is 10.10.0.0/23. Use `10.10.0.0 0.0.1.255` in the NAT ACLs and 10.10.0.0/23 in the Phase 6 crypto ACLs |
| R2 | VLAN 99: the guide gives the switch SVIs .50/.51, but the "VIP .1, R1 .2, R2 .3" pattern would also put R1 and R2 on .50/.51 | VLAN 99 routers use .52 and .53; PC-MGMT is .54 |
| R3 | Phase 6 names a new failover test "05-branch-vpn", but Phase 5 already has "05 Policy check" | Name the VPN test `06-branch-vpn` |
| R4 | VLAN names are not given for 10 to 99 | Assumed 10 USERS, 20 ADMIN, 30 IT, 40 GUEST, 50 SERVERS, 99 MGMT |
| R5 | A strict "Guest to any 10.x fails" rule would also block DNS and DHCP | Guest ACL permits DHCP and `udp any host 10.10.1.36 eq 53` (destination port 53) only, then denies 10.0.0.0/8, then permits the rest |
| R6 | INET cannot return traffic to the branch PAT address 198.51.100.10 unless ISP-B advertises 198.51.100.8/30 | ISP-B adds `network 198.51.100.8 mask 255.255.255.252` in Phase 3 |
| R7 | Packet Tracer 9.0.1.0858 has no route-map, weight, local-pref, update-source, default-originate or iBGP (confirmed by the Phase 0 feature tests) | D3, D4, D5 and D14 replace the planned policy; README notes the limitation |

## Draft ACL policy (finalised in Phase 4, `docs/acl-policy.md`)

| Source | Internet | Servers (VLAN 50) | Admin | IT | MGMT | Notes |
|---|---|---|---|---|---|---|
| USERS (10) | permit | permit | deny | deny | deny | DHCP permit first |
| ADMIN (20) | permit | permit | n/a | deny | deny | |
| IT (30) | permit | permit | permit | n/a | permit | |
| GUEST (40) | permit | deny (DNS only to 10.10.1.36) | deny | deny | deny | DHCP and DNS permits first |
| MGMT (99) | permit | permit | permit | permit | n/a | Management host |

ACLs are stateless, so SERVERS and MGMT subinterfaces get no inbound deny that would block replies.

## Known limitations (carry into the README)

- Packet Tracer lacks BGP route-maps, weight, local-pref, update-source, default-originate and iBGP. Inbound steering uses AS-path length (an extra upstream router), outbound uses HSRP plus a floating static default. A real network would use iBGP between the edge routers and local-pref.
- NAT with dual-homing: a reply that returns via the other ISP reaches a router with no translation. Normal traffic follows ISP-A, so this shows only in odd cases.
- Packet Tracer simulates BGP, HSRP and OSPF without real timers or hardware behaviour, so measured failover times are lab numbers only.
- If Phase 6 is built: an ISP-B outage takes down the branch.
- If Phase 7 is built: Packet Tracer APs support one SSID each, so two APs are used.

## Phase 3 BGP build (revised, replaces the guide's BGP bullets)

Do not use: route-map, `update-source`, `default-originate`, `weight`, `bgp default local-preference`, or any `neighbor ... remote-as 64500` between the two HQ routers (no iBGP).

```
! HQ-R1
ip route 192.0.2.0 255.255.255.0 Null0
ip route 0.0.0.0 0.0.0.0 10.10.255.66 250
router bgp 64500
 bgp router-id 10.10.255.1
 neighbor 198.51.100.1 remote-as 64496
 network 192.0.2.0 mask 255.255.255.0
! HQ-R2
ip route 192.0.2.0 255.255.255.0 Null0
ip route 0.0.0.0 0.0.0.0 10.10.255.65 250
router bgp 64500
 bgp router-id 10.10.255.2
 neighbor 198.51.100.5 remote-as 64497
 network 192.0.2.0 mask 255.255.255.0
! ISP-A
router bgp 64496
 neighbor 198.51.100.2 remote-as 64500
 neighbor 198.51.100.14 remote-as 64498
! ISP-B
router bgp 64497
 neighbor 198.51.100.6 remote-as 64500
 neighbor 198.51.100.18 remote-as 64499
 network 198.51.100.8 mask 255.255.255.252
! ISP-B-UP
router bgp 64499
 neighbor 198.51.100.17 remote-as 64497
 neighbor 198.51.100.22 remote-as 64498
! INET
router bgp 64498
 neighbor 198.51.100.13 remote-as 64496
 neighbor 198.51.100.21 remote-as 64499
 network 8.8.8.8 mask 255.255.255.255
 network 203.0.113.16 mask 255.255.255.252
```

Checks:
- INET: `show ip bgp` lists 192.0.2.0/24 twice; the best path (`>`) is via 198.51.100.13 (`64496 64500`).
- HQ-R1: `show ip bgp` has 8.8.8.8/32 via 198.51.100.1. HQ-R2 has it via 198.51.100.5.
- Shut HQ-R1 Gi0/1: INET's best path for 192.0.2.0/24 becomes 198.51.100.21 (`64499 64497 64500`), and HQ-R1 `show ip route` shows the default `S* 0.0.0.0/0 [250/0] via 10.10.255.66`.
