# Phase 2 Verification: HSRP gateways

| Field | Value |
|---|---|
| Date | 2026-11-27 |
| Packet Tracer version | 9.0.1.0858 |
| Topology file | `pkt/v2-routing.pkt` |
| Configs | `configs/HQ-R1.txt`, `configs/HQ-R2.txt`, `configs/HQ-SW1.txt`, `configs/HQ-SW2.txt` |
| Devices | HQ-R1, HQ-R2, HQ-SW1, HQ-SW2 (+ PC-USER1, PC-IT) |
| Commit | `feat: add hsrp gateways and ospf` (hash: add after commit) |

## 1. Result summary

| # | Gate check | Device(s) | Expected | Result | Evidence |
|---|---|---|---|---|---|
| 1 | HSRP state at baseline | HQ-R1, HQ-R2 | R1 Active (priority 110, preempt) and R2 Standby (100) on every VLAN | PASS | §3.1 |
| 2 | Ping to the VIP survives an HQ-R1 subinterface shutdown | PC-USER1, HQ-R1, HQ-R2 | VIP stays reachable; R2 becomes Active for the shut group | PASS (0 replies lost on shutdown) | §3.2, §3.4 |
| 3 | HQ-R1 reclaims Active via preempt | HQ-R1, HQ-R2 | After `no shutdown`, R1 Active and R2 Standby again | PASS (1 reply lost) | §3.3, §3.4 |
| 4 | Cross-VLAN ping | PC-USER1 to PC-IT | 0% loss, routed through the VIP gateway | PASS (0% loss, TTL 127) | §3.5 |

**Overall: PASS.** Fault used in every failover test: `shutdown`, then `no shutdown`, on HQ-R1 `Gi0/0.10` (VLAN 10, group 10).

## 2. Observations

| Item | Finding | Status |
|---|---|---|
| O1 | The VIP ping (§3.4) starts with one timeout | Most likely ARP resolution for the first packet. Not confirmed |
| O2 | During the fault, HQ-R2 lists its own address (10.10.0.3) in the Standby column for group 10 | Packet Tracer display quirk; real IOS would be expected to show `unknown` |
| O3 | The cross-VLAN ping (§3.2) lost 2 replies on failover and its replies carried TTL 126 instead of 127 | Explained by the OSPF reconvergence and transit-link path, see `ospf.md` §3.4 |
| O4 | `tracert` shows HQ-R1's real address (10.10.0.2), not the VIP | Expected: a router answers `tracert` from its own interface |

## 3. Design and evidence

HSRP version 2, group number = VLAN ID, VIP is the first usable address (see `docs/ip-plan.md`). HQ-R1 has priority 110 with preempt; HQ-R2 keeps the default 100.

| VLAN | Group | VIP | HQ-R1 | HQ-R2 |
|---|---|---|---|---|
| 10 | 10 | 10.10.0.1 | 10.10.0.2 | 10.10.0.3 |
| 20 | 20 | 10.10.0.129 | 10.10.0.130 | 10.10.0.131 |
| 30 | 30 | 10.10.0.193 | 10.10.0.194 | 10.10.0.195 |
| 40 | 40 | 10.10.1.1 | 10.10.1.2 | 10.10.1.3 |
| 50 | 50 | 10.10.1.33 | 10.10.1.34 | 10.10.1.35 |
| 99 | 99 | 10.10.1.49 | 10.10.1.52 | 10.10.1.53 |

Configuration excerpt (VLAN 10 subinterface; the other five follow the same pattern):

```
! HQ-R1
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.10.0.2 255.255.255.128
 standby version 2
 standby 10 ip 10.10.0.1
 standby 10 priority 110
 standby 10 preempt

! HQ-R2
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.10.0.3 255.255.255.128
 standby version 2
 standby 10 ip 10.10.0.1
```

### 3.1 Baseline (`show standby brief`)
```
HQ-R1#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         10   110 P Active   local           10.10.0.3       10.10.0.1
Gig         20   110 P Active   local           10.10.0.131     10.10.0.129
Gig         30   110 P Active   local           10.10.0.195     10.10.0.193
Gig         40   110 P Active   local           10.10.1.3       10.10.1.1
Gig         50   110 P Active   local           10.10.1.35      10.10.1.33
Gig         99   110 P Active   local           10.10.1.53      10.10.1.49

HQ-R2#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         10   100   Standby  10.10.0.2       local           10.10.0.1
Gig         20   100   Standby  10.10.0.130     local           10.10.0.129
Gig         30   100   Standby  10.10.0.194     local           10.10.0.193
Gig         40   100   Standby  10.10.1.2       local           10.10.1.1
Gig         50   100   Standby  10.10.1.34      local           10.10.1.33
Gig         99   100   Standby  10.10.1.52      local           10.10.1.49
```

### 3.2 Failover with cross-VLAN traffic
Continuous ping from PC-USER1 (gateway = VIP 10.10.0.1) to PC-IT, with the fault injected and removed on HQ-R1 `Gi0/0.10`.

| Phase | Observation |
|---|---|
| Baseline | Replies with TTL 127 (one router hop) |
| Shutdown | **2** replies lost, then replies with TTL 126 (return path crosses two routers, see O3) |
| `no shutdown` (preempt) | **1** reply lost, then TTL 127 again |

```
C:\>ping -t 10.10.0.199

Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time=10ms TTL=127
Request timed out.                                  <- shutdown
Request timed out.                                  <- shutdown
Reply from 10.10.0.199: bytes=32 time<1ms TTL=126
Reply from 10.10.0.199: bytes=32 time=1ms TTL=126
Reply from 10.10.0.199: bytes=32 time=10ms TTL=126
Reply from 10.10.0.199: bytes=32 time<1ms TTL=126
Reply from 10.10.0.199: bytes=32 time=10ms TTL=126
Request timed out.                                  <- no shutdown (preempt)
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time=1ms TTL=127
```

State during the fault:
```
HQ-R1#show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         10   110 P Init     unknown         unknown         10.10.0.1
Gig         20   110 P Active   local           10.10.0.131     10.10.0.129
Gig         30   110 P Active   local           10.10.0.195     10.10.0.193
Gig         40   110 P Active   local           10.10.1.3       10.10.1.1
Gig         50   110 P Active   local           10.10.1.35      10.10.1.33
Gig         99   110 P Active   local           10.10.1.53      10.10.1.49

HQ-R2#show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         10   100   Active   local           10.10.0.3       10.10.0.1
Gig         20   100   Standby  10.10.0.130     local           10.10.0.129
Gig         30   100   Standby  10.10.0.194     local           10.10.0.193
Gig         40   100   Standby  10.10.1.2       local           10.10.1.1
Gig         50   100   Standby  10.10.1.34      local           10.10.1.33
Gig         99   100   Standby  10.10.1.52      local           10.10.1.49
```

Only group 10 changed state. Groups 20, 30, 40, 50 and 99 stayed Active on HQ-R1.

### 3.3 Preempt recovery
```
HQ-R1(config)#int g0/0.10
HQ-R1(config-subif)#no shutdown
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0.10, changed state to up
%HSRP-6-STATECHANGE: GigabitEthernet0/0.10 Grp 10 state Standby -> Active     (HQ-R1)
%HSRP-6-STATECHANGE: GigabitEthernet0/0.10 Grp 10 state Speak -> Standby      (HQ-R2)

HQ-R1#show standby brief
Gig         10   110 P Active   local           10.10.0.3       10.10.0.1
Gig         20   110 P Active   local           10.10.0.131     10.10.0.129
Gig         30   110 P Active   local           10.10.0.195     10.10.0.193
Gig         40   110 P Active   local           10.10.1.3       10.10.1.1
Gig         50   110 P Active   local           10.10.1.35      10.10.1.33
Gig         99   110 P Active   local           10.10.1.53      10.10.1.49

HQ-R2#show standby brief
Gig         10   100   Standby  10.10.0.2       local           10.10.0.1
Gig         20   100   Standby  10.10.0.130     local           10.10.0.129
Gig         30   100   Standby  10.10.0.194     local           10.10.0.193
Gig         40   100   Standby  10.10.1.2       local           10.10.1.1
Gig         50   100   Standby  10.10.1.34      local           10.10.1.33
Gig         99   100   Standby  10.10.1.52      local           10.10.1.49
```

### 3.4 Failover ping to the VIP
Same fault and recovery, pinging the VIP itself from PC-USER1.

| Phase | Observation |
|---|---|
| Baseline | Replies with TTL 255 (the router answers directly) |
| Shutdown | **0** replies lost |
| `no shutdown` (preempt) | **1** reply lost |

```
C:\>ping -t 10.10.0.1

Request timed out.                                  <- first packet (see O1)
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time=4ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time=4ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time=1ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time=1ms TTL=255
Request timed out.                                  <- no shutdown (preempt)
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
Reply from 10.10.0.1: bytes=32 time<1ms TTL=255
```

The VIP ping needs no routing, so it loses nothing on shutdown. The cross-VLAN ping in §3.2 lost 2 replies because its return path also depends on OSPF (`ospf.md` §3.4).

### 3.5 Cross-VLAN ping
| Host | Address | Gateway |
|---|---|---|
| PC-USER1 (VLAN 10) | 10.10.0.100 255.255.255.128 | 10.10.0.1 |
| PC-IT (VLAN 30) | 10.10.0.199 255.255.255.224 | 10.10.0.193 |

```
C:\>ping 10.10.0.199

Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127
Reply from 10.10.0.199: bytes=32 time=10ms TTL=127
Reply from 10.10.0.199: bytes=32 time<1ms TTL=127

Ping statistics for 10.10.0.199:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),

C:\>tracert 10.10.0.199

  1   0 ms      0 ms      0 ms      10.10.0.2
  2   0 ms      0 ms      0 ms      10.10.0.199

Trace complete.
```

## 4. Issues found and fixed

| Issue | Cause | Fix | Re-test |
|---|---|---|---|
| After shutting and re-enabling `Gi0/0`, HQ-R1's subinterfaces came back as Standby at priority 100 | `standby priority 110` and `standby preempt` had no group number, so they applied to group 0. Groups 10 to 99 ran at the default priority 100 with no preempt | Added the group number to each command (`standby 10 priority 110`, `standby 10 preempt`, and so on) on all six subinterfaces | §3.1, §3.3 |
| PC-IT had default gateway 10.10.0.1, the VLAN 10 VIP | Wrong gateway entered for a VLAN 30 host. Cross-VLAN pings still worked, most likely through router proxy ARP, so the test passed for the wrong reason | Changed the gateway to 10.10.0.193 | §3.5 |
| HQ-SW2 startup config had `interface Vlan99` at 10.10.1.50, a duplicate of HQ-SW1 | Wrong address saved in the startup config | Set 10.10.1.51 and ran `copy run start`; `configs/HQ-SW2.txt` shows 10.10.1.51 | `configs/HQ-SW2.txt` |

The first issue is the source for incident report INC-002 (Phase 5).

## 5. Known limitations carried forward

- Only group 10 was failed individually. Whole-router and STP-root failures are covered by failover tests 02 and 04 in Phase 5, which will reuse this procedure with a continuous ping.
- Packet Tracer simulates HSRP without real timers, so lost-reply counts are lab figures, not production figures.
