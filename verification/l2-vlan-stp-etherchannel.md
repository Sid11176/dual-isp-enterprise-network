# Phase 1 Verification: Layer 2 (VLANs, trunks, LACP, RSTP)

| Field | Value |
|---|---|
| Date | 2026-11-26 |
| Packet Tracer version | 9.0.1.0858 |
| Topology file | `pkt/v1-layer2.pkt` |
| Configs | `configs/HQ-SW1.txt`, `configs/HQ-SW2.txt`, `configs/BR-SW.txt` |
| Devices | HQ-SW1, HQ-SW2, BR-SW (+ PC-USER1, PC-USER2, PC-ADMIN, PC-IT, PC-GUEST) |
| Commit | `feat: add layer 2 foundation` (hash: ) |

## 1. Result summary

| # | Gate check | Device(s) | Expected | Result | Evidence |
|---|---|---|---|---|---|
| 1 | VLANs present and named per `ip-plan.md` | all 3 | HQ: 10,20,30,40,50,99,998,999. BR: 110,140,199,998,999 | PASS | §3.1 |
| 2 | Trunk native 998, explicit allowed list | HQ-SW1/SW2 (Po1) | Native 998, allowed 10,20,30,40,50,99,998 | PASS | §3.2 |
| 3 | BR-SW uplink trunk configured | BR-SW | Gi0/1 trunk, native 998, allowed 110,140,199,998 (router not cabled yet) | PASS | §3.2 |
| 4 | EtherChannel up | HQ-SW1/SW2 | Po1 `(SU)`, Fa0/23 and Fa0/24 `(P)`, LACP | PASS | §3.3 |
| 5 | STP root placement | HQ-SW1/SW2 | HQ-SW1 root for VLANs 10,20,30,40,50,99; HQ-SW2 Root/FWD on Po1 | PASS | §3.4 |
| 6 | No blocked ports | HQ-SW1/SW2 | All FWD | PASS (no `BLK` role in any VLAN) | §3.4 |
| 7 | PortFast + BPDU guard on access ports | all 3 | Enabled on every access port in use | PASS (config) | §3.5 |
| 8 | Unused ports parked | all 3 | VLAN 999, `disabled` | PASS | §3.6 |
| 9 | Same-VLAN ping across Po1 | PC-USER1 to PC-USER2 | 0% loss | PASS (0% loss) | §3.7 |
| 10 | Port map matches `ip-plan.md` | all 3 | Cabling = map | PASS | §3.8 |

**Overall: PASS. The Phase 1 gate is closed.** All ten checks pass on the saved topology after a power cycle of the devices.

## 2. Items that were open and how they closed

| Item | Finding | Resolution |
|---|---|---|
| O1 | Before the power cycle, HQ-SW2 reported itself as root for VLANs 20, 30, 40, 50 and 99 (only VLAN 10 converged) | After saving and power-cycling the devices, HQ-SW2 shows HQ-SW1 (00E0.8F30.4A22) as root for all six VLANs, with Po1 `Root FWD`. The temporary VLAN 20 PC test was not run, so the cause is attributed to stale Packet Tracer STP state (a known quirk) and is not confirmed |
| O2 | VLAN 998: HQ-SW1 Po1 showed `Altn BLK` | After the power cycle HQ-SW1 Po1 shows `Root FWD` for VLAN 998 and no port is blocked in any VLAN. VLAN 998 is not in the priority list (the plan covers VLANs 10 to 99; 998 carries no hosts), so HQ-SW2 is its root on lowest MAC |
| O3 | Test addresses 10.10.0.1 and .2 would clash with the Phase 2 HSRP addresses | PC-USER1 and PC-USER2 now use 10.10.0.100 and 10.10.0.101; ping re-run (§3.7). Switch both to DHCP in Phase 3 |

## 3. Evidence

### 3.1 VLANs (`show vlan brief`)
HQ-SW1:
```
HQ-SW1#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig0/1
10   USERS                            active    Fa0/1
20   ADMIN                            active    Fa0/2
30   IT                               active
40   GUEST                            active
50   SERVERS                          active
99   MGMT                             active
998  NATIVE                           active
999  PARKING                          active    Fa0/3, Fa0/4, Fa0/5, Fa0/6
                                                Fa0/7, Fa0/8, Fa0/9, Fa0/10
                                                Fa0/11, Fa0/12, Fa0/13, Fa0/14
                                                Fa0/15, Fa0/16, Fa0/17, Fa0/18
                                                Fa0/19, Fa0/20, Fa0/21, Fa0/22
                                                Gig0/2
1002 fddi-default                     active
1003 token-ring-default               active
1004 fddinet-default                  active
1005 trnet-default                    active
```

HQ-SW2:
```
HQ-SW2#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig0/1
10   USERS                            active    Fa0/1
20   ADMIN                            active
30   IT                               active    Fa0/2
40   GUEST                            active    Fa0/3
50   SERVERS                          active
99   MGMT                             active
998  NATIVE                           active
999  PARKING                          active    Fa0/4, Fa0/5, Fa0/6, Fa0/7
                                                Fa0/8, Fa0/9, Fa0/10, Fa0/11
                                                Fa0/12, Fa0/13, Fa0/14, Fa0/15
                                                Fa0/16, Fa0/17, Fa0/18, Fa0/19
                                                Fa0/20, Fa0/21, Fa0/22, Gig0/2
1002 fddi-default                     active
1003 token-ring-default               active
1004 fddinet-default                  active
1005 trnet-default                    active
```

BR-SW:
```
BR-SW#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig0/1
110  BR-STAFF                         active    Fa0/1
140  BR-GUEST                         active    Fa0/2
199  BR-MGMT                          active
998  NATIVE                           active
999  PARKING                          active    Fa0/3, Fa0/4, Fa0/5, Fa0/6
                                                Fa0/7, Fa0/8, Fa0/9, Fa0/10
                                                Fa0/11, Fa0/12, Fa0/13, Fa0/14
                                                Fa0/15, Fa0/16, Fa0/17, Fa0/18
                                                Fa0/19, Fa0/20, Fa0/21, Fa0/22
                                                Fa0/23, Fa0/24, Gig0/2
1002 fddi-default                     active
1003 token-ring-default               active
1004 fddinet-default                  active
1005 trnet-default                    active
```

VLAN 1 shows only Gig0/1 (uplink to the router in Phase 2). The HQ switches show VLANs 110, 140 and 199 absent and BR-SW shows VLANs 10 to 99 absent, as designed.

### 3.2 Trunks
HQ-SW1 and HQ-SW2: `show interfaces trunk`. BR-SW: `show running-config | section GigabitEthernet0/1` (no router cabled, so the port is not trunking yet).

HQ-SW1:
```
HQ-SW1#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Po1         on           802.1q         trunking      998

Port        Vlans allowed on trunk
Po1         10,20,30,40,50,99,998

Port        Vlans allowed and active in management domain
Po1         10,20,30,40,50,99,998

Port        Vlans in spanning tree forwarding state and not pruned
Po1         10,20,30,40,50,99,998
```

HQ-SW2:
```
HQ-SW2#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Po1         on           802.1q         trunking      998

Port        Vlans allowed on trunk
Po1         10,20,30,40,50,99,998

Port        Vlans allowed and active in management domain
Po1         10,20,30,40,50,99,998

Port        Vlans in spanning tree forwarding state and not pruned
Po1         10,20,30,40,50,99,998
```

BR-SW:
```
BR-SW#show running-config | section GigabitEthernet0/1
interface GigabitEthernet0/1
 switchport trunk native vlan 998
 switchport trunk allowed vlan 110,140,199,998
 switchport mode trunk
 switchport nonegotiate
```

### 3.3 EtherChannel (`show etherchannel summary`)
HQ-SW1:
```
HQ-SW1#show etherchannel summary
Flags:  D - down        P - in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------

1      Po1(SU)           LACP   Fa0/23(P) Fa0/24(P)
```

HQ-SW2:
```
HQ-SW2#show etherchannel summary
Flags:  D - down        P - in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------

1      Po1(SU)           LACP   Fa0/23(P) Fa0/24(P)
```

### 3.4 Spanning tree (`show spanning-tree`)
HQ-SW1:
```
HQ-SW1#show spanning-tree
VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/23           Desg FWD 19        128.23   P2p
Fa0/24           Desg FWD 19        128.24   P2p

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24586  (priority 24576 sys-id-ext 10)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Po1              Desg FWD 12        128.27   P2p

VLAN0020
  Spanning tree enabled protocol rstp
  Root ID    Priority    24596
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24596  (priority 24576 sys-id-ext 20)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg FWD 19        128.2    P2p
Po1              Desg FWD 12        128.27   P2p

VLAN0030
  Spanning tree enabled protocol rstp
  Root ID    Priority    24606
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24606  (priority 24576 sys-id-ext 30)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Desg FWD 12        128.27   P2p

VLAN0040
  Spanning tree enabled protocol rstp
  Root ID    Priority    24616
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24616  (priority 24576 sys-id-ext 40)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Desg FWD 12        128.27   P2p

VLAN0050
  Spanning tree enabled protocol rstp
  Root ID    Priority    24626
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24626  (priority 24576 sys-id-ext 50)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Desg FWD 12        128.27   P2p

VLAN0099
  Spanning tree enabled protocol rstp
  Root ID    Priority    24675
             Address     00E0.8F30.4A22
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24675  (priority 24576 sys-id-ext 99)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Desg FWD 12        128.27   P2p

VLAN0998
  Spanning tree enabled protocol rstp
  Root ID    Priority    33766
             Address     0001.C902.B83A
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    33766  (priority 32768 sys-id-ext 998)
             Address     00E0.8F30.4A22
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Root FWD 12        128.27   P2p
```

HQ-SW2:
```
HQ-SW2#show spanning-tree
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     00E0.8F30.4A22
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28682  (priority 28672 sys-id-ext 10)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Po1              Root FWD 12        128.27   P2p

VLAN0020
  Spanning tree enabled protocol rstp
  Root ID    Priority    24596
             Address     00E0.8F30.4A22
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28692  (priority 28672 sys-id-ext 20)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Root FWD 12        128.27   P2p

VLAN0030
  Spanning tree enabled protocol rstp
  Root ID    Priority    24606
             Address     00E0.8F30.4A22
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28702  (priority 28672 sys-id-ext 30)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg FWD 19        128.2    P2p
Po1              Root FWD 12        128.27   P2p

VLAN0040
  Spanning tree enabled protocol rstp
  Root ID    Priority    24616
             Address     00E0.8F30.4A22
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28712  (priority 28672 sys-id-ext 40)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/3            Desg FWD 19        128.3    P2p
Po1              Root FWD 12        128.27   P2p

VLAN0050
  Spanning tree enabled protocol rstp
  Root ID    Priority    24626
             Address     00E0.8F30.4A22
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28722  (priority 28672 sys-id-ext 50)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Root FWD 12        128.27   P2p

VLAN0099
  Spanning tree enabled protocol rstp
  Root ID    Priority    24675
             Address     00E0.8F30.4A22
             Cost        12
             Port        27(Port-channel1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28771  (priority 28672 sys-id-ext 99)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Root FWD 12        128.27   P2p

VLAN0998
  Spanning tree enabled protocol rstp
  Root ID    Priority    33766
             Address     0001.C902.B83A
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    33766  (priority 32768 sys-id-ext 998)
             Address     0001.C902.B83A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Desg FWD 12        128.27   P2p
```

BR-SW:
```
BR-SW#show spanning-tree
VLAN0110
  Spanning tree enabled protocol rstp
  Root ID    Priority    32878
             Address     0060.4720.2A9A
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32878  (priority 32768 sys-id-ext 110)
             Address     0060.4720.2A9A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p

VLAN0140
  Spanning tree enabled protocol rstp
  Root ID    Priority    32908
             Address     0060.4720.2A9A
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32908  (priority 32768 sys-id-ext 140)
             Address     0060.4720.2A9A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg FWD 19        128.2    P2p
```

Observations (captured after saving and power-cycling the devices):
- HQ-SW1 is root for VLANs 10, 20, 30, 40, 50 and 99 (priority 24576 plus VLAN ID).
- HQ-SW2 shows root ID 00E0.8F30.4A22 (HQ-SW1) for the same six VLANs, with Po1 `Root FWD`.
- No port is in `BLK` or `Altn` state in any VLAN; there is one logical inter-switch link, so no blocked port is expected.
- VLAN 998 (native, no hosts) is not in the priority list; HQ-SW2 is its root on lowest MAC and HQ-SW1 Po1 is `Root FWD`.
- VLANs 1 and 999 are not allowed on the trunk, so each HQ switch is root of its own copy. This is expected.
- Packet Tracer lists Fa0/23 and Fa0/24 as ports in a few VLANs next to Po1. Treat that as a display quirk; the bundle is `Po1(SU)` in §3.3.
- BR-SW has no neighbor yet, so it is root for VLANs 110 and 140.

### 3.5 PortFast and BPDU guard
PortFast and BPDU guard are enabled per interface (the global defaults are off), so the evidence is the exported configs rather than `show spanning-tree summary`. The earlier summary capture was taken before the power cycle and is not used: for HQ-SW2 it listed VLANs as rooted locally that §3.4 now shows rooted on HQ-SW1.

Per-interface evidence, exported configs:

HQ-SW1 (from exported config):
```
interface FastEthernet0/1
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

HQ-SW2 (from exported config):
```
interface FastEthernet0/1
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

BR-SW (from exported config):
```
interface FastEthernet0/1
 switchport access vlan 110
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

Every in-use access port on all three switches carries `spanning-tree portfast` and `spanning-tree bpduguard enable` (HQ-SW1 Fa0/1 to 2, HQ-SW2 Fa0/1 to 3, BR-SW Fa0/1 to 2).

### 3.6 Unused ports (`show interfaces status`)
HQ-SW1:
```
HQ-SW1#show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Po1                          connected    trunk      a-full  a-100
Fa0/1                        connected    10         a-full  a-100 10/100BaseTX
Fa0/2                        connected    20         a-full  a-100 10/100BaseTX
Fa0/3                        disabled 999        auto    auto  10/100BaseTX
Fa0/4                        disabled 999        auto    auto  10/100BaseTX
Fa0/5                        disabled 999        auto    auto  10/100BaseTX
Fa0/6                        disabled 999        auto    auto  10/100BaseTX
Fa0/7                        disabled 999        auto    auto  10/100BaseTX
Fa0/8                        disabled 999        auto    auto  10/100BaseTX
Fa0/9                        disabled 999        auto    auto  10/100BaseTX
Fa0/10                       disabled 999        auto    auto  10/100BaseTX
Fa0/11                       disabled 999        auto    auto  10/100BaseTX
Fa0/12                       disabled 999        auto    auto  10/100BaseTX
Fa0/13                       disabled 999        auto    auto  10/100BaseTX
Fa0/14                       disabled 999        auto    auto  10/100BaseTX
Fa0/15                       disabled 999        auto    auto  10/100BaseTX
Fa0/16                       disabled 999        auto    auto  10/100BaseTX
Fa0/17                       disabled 999        auto    auto  10/100BaseTX
Fa0/18                       disabled 999        auto    auto  10/100BaseTX
Fa0/19                       disabled 999        auto    auto  10/100BaseTX
Fa0/20                       disabled 999        auto    auto  10/100BaseTX
Fa0/21                       disabled 999        auto    auto  10/100BaseTX
Fa0/22                       disabled 999        auto    auto  10/100BaseTX
Fa0/23                       connected    trunk      a-full  a-100 10/100BaseTX
Fa0/24                       connected    trunk      a-full  a-100 10/100BaseTX
Gig0/1                       notconnect   1          auto    auto  10/100/1000BaseTX
Gig0/2                       disabled 999        auto    auto  10/100/1000BaseTX
```

HQ-SW2:
```
HQ-SW2#show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Po1                          connected    trunk      a-full  a-100
Fa0/1                        connected    10         a-full  a-100 10/100BaseTX
Fa0/2                        connected    30         a-full  a-100 10/100BaseTX
Fa0/3                        connected    40         a-full  a-100 10/100BaseTX
Fa0/4                        disabled 999        auto    auto  10/100BaseTX
Fa0/5                        disabled 999        auto    auto  10/100BaseTX
Fa0/6                        disabled 999        auto    auto  10/100BaseTX
Fa0/7                        disabled 999        auto    auto  10/100BaseTX
Fa0/8                        disabled 999        auto    auto  10/100BaseTX
Fa0/9                        disabled 999        auto    auto  10/100BaseTX
Fa0/10                       disabled 999        auto    auto  10/100BaseTX
Fa0/11                       disabled 999        auto    auto  10/100BaseTX
Fa0/12                       disabled 999        auto    auto  10/100BaseTX
Fa0/13                       disabled 999        auto    auto  10/100BaseTX
Fa0/14                       disabled 999        auto    auto  10/100BaseTX
Fa0/15                       disabled 999        auto    auto  10/100BaseTX
Fa0/16                       disabled 999        auto    auto  10/100BaseTX
Fa0/17                       disabled 999        auto    auto  10/100BaseTX
Fa0/18                       disabled 999        auto    auto  10/100BaseTX
Fa0/19                       disabled 999        auto    auto  10/100BaseTX
Fa0/20                       disabled 999        auto    auto  10/100BaseTX
Fa0/21                       disabled 999        auto    auto  10/100BaseTX
Fa0/22                       disabled 999        auto    auto  10/100BaseTX
Fa0/23                       connected    trunk      a-full  a-100 10/100BaseTX
Fa0/24                       connected    trunk      a-full  a-100 10/100BaseTX
Gig0/1                       notconnect   1          auto    auto  10/100/1000BaseTX
Gig0/2                       disabled 999        auto    auto  10/100/1000BaseTX
```

BR-SW:
```
BR-SW#show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1                        connected    110        a-full  a-100 10/100BaseTX
Fa0/2                        connected    140        a-full  a-100 10/100BaseTX
Fa0/3                        disabled 999        auto    auto  10/100BaseTX
Fa0/4                        disabled 999        auto    auto  10/100BaseTX
Fa0/5                        disabled 999        auto    auto  10/100BaseTX
Fa0/6                        disabled 999        auto    auto  10/100BaseTX
Fa0/7                        disabled 999        auto    auto  10/100BaseTX
Fa0/8                        disabled 999        auto    auto  10/100BaseTX
Fa0/9                        disabled 999        auto    auto  10/100BaseTX
Fa0/10                       disabled 999        auto    auto  10/100BaseTX
Fa0/11                       disabled 999        auto    auto  10/100BaseTX
Fa0/12                       disabled 999        auto    auto  10/100BaseTX
Fa0/13                       disabled 999        auto    auto  10/100BaseTX
Fa0/14                       disabled 999        auto    auto  10/100BaseTX
Fa0/15                       disabled 999        auto    auto  10/100BaseTX
Fa0/16                       disabled 999        auto    auto  10/100BaseTX
Fa0/17                       disabled 999        auto    auto  10/100BaseTX
Fa0/18                       disabled 999        auto    auto  10/100BaseTX
Fa0/19                       disabled 999        auto    auto  10/100BaseTX
Fa0/20                       disabled 999        auto    auto  10/100BaseTX
Fa0/21                       disabled 999        auto    auto  10/100BaseTX
Fa0/22                       disabled 999        auto    auto  10/100BaseTX
Fa0/23                       disabled 999        auto    auto  10/100BaseTX
Fa0/24                       disabled 999        auto    auto  10/100BaseTX
Gig0/1                       notconnect   trunk      auto    auto  10/100/1000BaseTX
Gig0/2                       disabled 999        auto    auto  10/100/1000BaseTX
```

All unused Fa and Gi ports on all three switches are in VLAN 999 and `disabled`. HQ Gi0/1 stays in VLAN 1 (not connected) until it becomes the router trunk in Phase 2.

### 3.7 End-to-end L2 test
Temporary static addresses, chosen outside the Phase 2 HSRP addresses (.1 to .3) and the DHCP exclusion range (.1 to .10):
```
PC-USER1 ---> 10.10.0.100 (255.255.255.128)
PC-USER2 ---> 10.10.0.101 (255.255.255.128)

C:\>ping 10.10.0.101

Pinging 10.10.0.101 with 32 bytes of data:

Reply from 10.10.0.101: bytes=32 time<1ms TTL=128
Reply from 10.10.0.101: bytes=32 time<1ms TTL=128
Reply from 10.10.0.101: bytes=32 time<1ms TTL=128
Reply from 10.10.0.101: bytes=32 time<1ms TTL=128

Ping statistics for 10.10.0.101:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

### 3.8 Port map check
| Switch | Port | Device | VLAN | Matches `ip-plan.md`? |
|---|---|---|---|---|
| HQ-SW1 | Fa0/1 | PC-USER1 | 10 | Yes |
| HQ-SW1 | Fa0/2 | PC-ADMIN | 20 | Yes |
| HQ-SW1 | Fa0/3 to 4 | free for servers (Phase 3) | 999, shut | Yes |
| HQ-SW2 | Fa0/1 | PC-USER2 | 10 | Yes |
| HQ-SW2 | Fa0/2 | PC-IT | 30 | Yes |
| HQ-SW2 | Fa0/3 | PC-GUEST | 40 | Yes |
| HQ-SW2 | Fa0/4 | PC-MGMT (Phase 4) | 999, shut | Yes |
| BR-SW | Fa0/1 | branch staff client (Phase 3) | 110 | Yes |
| BR-SW | Fa0/2 | branch guest client (Phase 3) | 140 | Yes |
| BR-SW | Fa0/3 to 4 | APs (Phase 7) | 999, shut | Yes |

## 4. Issues found and fixed

| Issue | Cause | Fix | Re-test |
|---|---|---|---|
| BR-SW had no trunk, access ports or parking | Only VLANs were created in the first pass | Added Gi0/1 trunk, Fa0/1 to 2 access with PortFast and BPDU guard, parked all other ports | §3.1, 3.2, 3.5, 3.6 |
| PC-GUEST was on HQ-SW1 Fa0/3, which the port map reserves for the servers | Cabling differed from `ip-plan.md` | Moved PC-GUEST to HQ-SW2 Fa0/3 | §3.8 |
| HQ Gi0/2 not parked | Missed in the first pass | Parked in VLAN 999 and shut on both HQ switches | §3.6 |
| HQ-SW2 reported itself as root for VLANs 20, 30, 40, 50, 99 | Stale Packet Tracer STP state (not confirmed) | Saved and power-cycled the devices | §3.4 |
| VLAN 998 `Altn BLK` on HQ-SW1 | Same stale state | Cleared by the power cycle | §3.4 |
| Test addresses 10.10.0.1 and .2 clashed with the Phase 2 HSRP plan | Chosen without checking `ip-plan.md` | Changed to 10.10.0.100 and .101 | §3.7 |

## 5. Known limitations carried forward

- HQ Gi0/1 (router uplinks) and the BR-SW trunk are exercised in Phase 2 and Phase 3.
- Packet Tracer's STP state can go stale until the devices are saved and power-cycled (§2). Re-check §3.4 after Phase 2, when the router subinterfaces and SVIs make every VLAN active on both switches.
- Blocking behavior cannot be demonstrated until a second inter-switch path exists; failover test 04 covers STP and HSRP together in Phase 5.
