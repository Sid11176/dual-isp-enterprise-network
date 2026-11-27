# Phase 2 Verification: OSPF

| Field | Value |
|---|---|
| Date | 2026-11-27 |
| Packet Tracer version | 9.0.1.0858 |
| Topology file | `pkt/v2-routing.pkt` |
| Configs | `configs/HQ-R1.txt`, `configs/HQ-R2.txt` |
| Devices | HQ-R1, HQ-R2 |
| Commit | `feat: add hsrp gateways and ospf` (hash: add after commit) |

## 1. Result summary

| # | Gate check | Device(s) | Expected | Result | Evidence |
|---|---|---|---|---|---|
| 1 | Adjacency | HQ-R1, HQ-R2 | Neighbor in state FULL over the transit link | PASS | §3.1 |
| 2 | OSPF interfaces | HQ-R1, HQ-R2 | Loopback, six HQ subinterfaces and Gi0/2 in area 0; only Gi0/2 non-passive | PASS | §3.2, configs |
| 3 | Routing table | HQ-R1, HQ-R2 | Peer loopback learned via Gi0/2 (gate reworded, see §2 O1) | PASS | §3.3 |
| 4 | OSPF carries traffic when a gateway interface fails | HQ-R1 | With `Gi0/0.10` shut, the VLAN 10 subnet is learned from HQ-R2 | PASS | §3.4 |

**Overall: PASS.**

## 2. Observations

| Item | Finding | Status |
|---|---|---|
| O1 | The roadmap gate said "HQ subnets learned". Only the peer's loopback appears in `show ip route ospf` | Expected. Every HQ subnet and the transit /30 are connected on both routers, so OSPF has nothing else to install. Gate reworded to "peer loopback learned" |
| O2 | `show ip ospf interface brief` shows `WAIT` on the loopbacks and several passive subinterfaces | Looks like a Packet Tracer display quirk (real IOS shows `LOOP` on a loopback). These interfaces have no neighbors. HQ-R1 `Gi0/0.10` shows `DR` after it was shut down and re-enabled, which fits that explanation |
| O3 | Neighbors show Pri 0 although `ip ospf priority 1` is set on Gi0/2 | Harmless: a point-to-point link holds no DR/BDR election, so priority has no effect |

## 3. Design and evidence

Single area 0. OSPF is enabled per interface with `ip ospf 1 area 0` on Loopback0, the six HQ subinterfaces and Gi0/2. `passive-interface default` is set, with only the transit link Gi0/2 non-passive and configured as point-to-point. Router IDs come from the loopbacks.

| Device | Router ID (Loopback0) | Transit Gi0/2 (10.10.255.64/30) |
|---|---|---|
| HQ-R1 | 10.10.255.1 | 10.10.255.65 |
| HQ-R2 | 10.10.255.2 | 10.10.255.66 |

Configuration excerpt (HQ-R1; HQ-R2 mirrors it with its own addresses):
```
interface Loopback0
 ip address 10.10.255.1 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0.10
 ip address 10.10.0.2 255.255.255.128
 ip ospf 1 area 0
!
interface GigabitEthernet0/2
 ip address 10.10.255.65 255.255.255.252
 ip ospf network point-to-point
 ip ospf priority 1
 ip ospf 1 area 0
!
router ospf 1
 log-adjacency-changes
 passive-interface default
 no passive-interface GigabitEthernet0/2
```

The other five subinterfaces (`.20`, `.30`, `.40`, `.50`, `.99`) carry `ip ospf 1 area 0` as well.

### 3.1 Neighbors (`show ip ospf neighbor`)
```
HQ-R1#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.10.255.2       0   FULL/  -        00:00:36    10.10.255.66    GigabitEthernet0/2

HQ-R2#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.10.255.1       0   FULL/  -        00:00:30    10.10.255.65    GigabitEthernet0/2
```

`FULL/ -` shows no DR/BDR role, which confirms the point-to-point network type. Gi0/2 reads `POINT` in §3.2.

### 3.2 Interfaces (`show ip ospf interface brief`)
```
HQ-R1#show ip ospf interface brief
Interface     PID   Area                     IP Address/Mask          Cost  State  Nbrs F/C
Lo0             1   0                    10.10.255.1/255.255.255.255   1     WAIT  0/0
Gig             1   0                    10.10.0.130/255.255.255.192   1     WAIT  0/0
Gig             1   0                    10.10.0.194/255.255.255.224   1     WAIT  0/0
Gig             1   0                      10.10.1.2/255.255.255.224   1     WAIT  0/0
Gig             1   0                     10.10.1.34/255.255.255.240   1     WAIT  0/0
Gig             1   0                     10.10.1.52/255.255.255.240   1     WAIT  0/0
Gig0/2          1   0                   10.10.255.65/255.255.255.252   1    POINT  0/0
Gig             1   0                      10.10.0.2/255.255.255.128   1       DR  0/0

HQ-R2#show ip ospf interface brief
Interface     PID   Area                     IP Address/Mask          Cost  State  Nbrs F/C
Lo0             1   0                    10.10.255.2/255.255.255.255   1     WAIT  0/0
Gig             1   0                      10.10.0.3/255.255.255.128   1       DR  0/0
Gig             1   0                    10.10.0.131/255.255.255.192   1       DR  0/0
Gig             1   0                    10.10.0.195/255.255.255.224   1       DR  0/0
Gig             1   0                      10.10.1.3/255.255.255.224   1       DR  0/0
Gig             1   0                     10.10.1.35/255.255.255.240   1       DR  0/0
Gig             1   0                     10.10.1.53/255.255.255.240   1       DR  0/0
Gig0/2          1   0                   10.10.255.66/255.255.255.252   1    POINT  0/0
```

Each router has eight interfaces in area 0. Gi0/2 is the only interface with a neighbor path; the others are passive (see O2).

### 3.3 Routing table (`show ip route ospf`)
```
HQ-R1#show ip route ospf
     10.0.0.0/8 is variably subnetted, 16 subnets, 6 masks
O       10.10.255.2 [110/2] via 10.10.255.66, 00:25:28, GigabitEthernet0/2

HQ-R2#show ip route ospf
     10.0.0.0/8 is variably subnetted, 16 subnets, 6 masks
O       10.10.255.1 [110/2] via 10.10.255.65, 00:25:50, GigabitEthernet0/2
```

Each router learns the other's loopback over the transit link (O1).

### 3.4 OSPF route during an HSRP fault
Captured on HQ-R1 with `Gi0/0.10` shut down (the route can only come from OSPF while the connected route is absent):
```
HQ-R1#show ip route 10.10.0.0
Routing entry for 10.10.0.0/25
Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 10.10.255.66 on GigabitEthernet0/2, 00:00:08 ago
  Routing Descriptor Blocks:
  * 10.10.255.66, from 10.10.255.2, 00:00:08 ago, via GigabitEthernet0/2
      Route metric is 2, traffic share count is 1
```

With its own VLAN 10 subinterface down, HQ-R1 learned 10.10.0.0/25 from HQ-R2 (router ID 10.10.255.2) as an intra-area route with metric 2. OSPF therefore carries VLAN 10 traffic across the transit link when a gateway interface fails. This explains the TTL 126 replies in `hsrp.md` §3.2: PC-IT's gateway for VLAN 30 is still HQ-R1, which forwards the reply to HQ-R2 over the transit link, a second router hop. It also likely explains the 2 replies lost on that test, which include the time HQ-R1 needs to learn the route. The VIP ping in `hsrp.md` §3.4 needs no routing and lost none.

## 4. Issues found and fixed

| Issue | Cause | Fix | Re-test |
|---|---|---|---|
| `show ip route ospf` gate expected "HQ subnets learned" | Wrong expectation: connected routes on both routers hide the HQ subnets from the OSPF table | Reworded the gate to "peer loopback learned" and added §3.4 as proof that the subnets are advertised | §3.3, §3.4 |

## 5. Known limitations carried forward

- Only HQ-R2's advertisement of the VLAN 10 subnet is proven (§3.4). Other routers join the OSPF domain in Phase 3 and will learn the remaining HQ subnets.
- OSPF is internal only; BGP carries external routes and is not redistributed into OSPF (`docs/design-decisions.md`).
- Router IDs come from the loopbacks. A pinned `router-id` is a possible later hardening step.
