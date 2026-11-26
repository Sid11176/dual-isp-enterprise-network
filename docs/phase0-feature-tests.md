# Phase 0 Feature Tests

Packet Tracer version: 9.0.1.0858   Date: 25 November 2026

Each feature the design depends on was tested in a throwaway Packet Tracer file (`scratch-feature-test.pkt`, not kept in the repo) before any lab work started. Results are below; the one failure (Test 1) changed the design, recorded in `design-decisions.md` (D3, D4, D5, D14, R7). The steps are kept so the tests can be repeated.

| # | Feature | Device | Pass? | Notes |
|---|---|---|---|---|
| 1 | BGP + route-map + AS-path prepend (not used) | 2911 x2 | FAIL | Packet Tracer has no route-map, `neighbor weight`, `bgp default local-preference`, `update-source`, `default-originate` or iBGP. Replaced by Test 1R: extra upstream router (AS 64499) for AS-path length, eBGP only, floating static default |
| 2 | IPsec site-to-site, `securityk9` | 2911 x2 | PASS | Extended ping with a source works only when typed interactively, one prompt at a time |
| 3 | LACP EtherChannel | 2960 x2 | PASS | |
| 4 | HSRP v2 on a subinterface | 2911 x2 + 2960 | PASS | |
| 5 | AP with WPA2-PSK | Access Point-PT + laptop | PASS | |
| 1R | Replacement BGP test: AS-path length steering and floating-default failover (6 routers) | 2911 x6 | PASS | INET prefers ISP-A; failover and recovery verified; `router-id` works; iBGP is not supported |

## 1. BGP, route-map, prepend (FAILED, not used)

Packet Tracer has no route-map, `neighbor weight`, `bgp default local-preference`, `update-source` or `default-originate`. None of these appear anywhere in the lab design. Test 1R below replaces this test.

## 2. IPsec on a 2911

On both routers: `show version`. If the technology package shows `securityk9` as disabled:
```
license boot module c2900 technology-package securityk9
```
Accept the agreement, then `copy run start` and `reload`.

Scratch design: R1 Gi0/0 10.0.0.1/30, LAN Lo1 10.1.1.1/24; R2 Gi0/0 10.0.0.2/30, LAN Lo1 10.2.2.1/24; static routes to each other's LAN.
```
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key PLACEHOLDER-KEY address 10.0.0.2      ! use 10.0.0.1 on R2
crypto ipsec transform-set TS esp-aes esp-sha-hmac
ip access-list extended VPN
 permit ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255          ! mirror on R2
crypto map CM 10 ipsec-isakmp
 set peer 10.0.0.2                                         ! 10.0.0.1 on R2
 set transform-set TS
 match address VPN
interface Gi0/0
 crypto map CM
```
Generate traffic: `ping 10.2.2.1 source Loopback1` from R1.
Pass: `show crypto isakmp sa` shows QM_IDLE and `show crypto ipsec sa` shows encaps and decaps rising.

## 3. LACP

Two 2960s, two cables Fa0/23-24 on each.
```
interface range fa0/23 - 24
 channel-protocol lacp
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
```
Pass: `show etherchannel summary` shows `Po1(SU)` with `Fa0/23(P) Fa0/24(P)`. If ports show `(I)` or `(D)`, shut and no-shut the ports after config.

## 4. HSRP on a subinterface

Two 2911s on one 2960 trunk. Both routers:
```
interface Gi0/0
 no shutdown
interface Gi0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.2 255.255.255.0      ! .3 on R2
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110                     ! 100 on R2
 standby 10 preempt
```
Add a PC in VLAN 10 (gateway 192.168.10.1).
Pass: `show standby brief` shows R1 Active, R2 Standby; the PC pings the VIP.

## 5. Access Point with WPA2-PSK

Access Point-PT on a switch access port. Config tab, Port 1 > Wireless: set an SSID, Authentication WPA2-PSK, passphrase `PLACEHOLDER-PASS`. On a laptop, power off, swap the wired module for the wireless one (WPC300N), power on, and in the PC Wireless app connect to the SSID with the same passphrase.
Pass: laptop shows connected and receives an address (or pings a wired PC).

## 1R. Replacement BGP test (needed because Test 1 failed)

Packet Tracer 9 has no route-map, weight, local-pref, update-source, default-originate or iBGP. The replacement design needs only eBGP, the shorter-AS-path rule, and a floating static route. This test checks that Packet Tracer does that.

Routers and addresses (from `ip-plan.md`):

| Router | AS | Interfaces |
|---|---|---|
| HQ-R1 | 64500 | Gi0/1 198.51.100.2/30, Gi0/2 10.10.255.65/30, Lo0 10.10.255.1/32 |
| HQ-R2 | 64500 | Gi0/1 198.51.100.6/30, Gi0/2 10.10.255.66/30, Lo0 10.10.255.2/32 |
| ISP-A | 64496 | Gi0/0 198.51.100.1/30 (to HQ-R1), Gi0/1 198.51.100.13/30 (to INET) |
| ISP-B | 64497 | Gi0/0 198.51.100.5/30 (to HQ-R2), Gi0/2 198.51.100.17/30 (to ISP-B-UP) |
| ISP-B-UP | 64499 | Gi0/0 198.51.100.18/30, Gi0/1 198.51.100.21/30 (to INET) |
| INET | 64498 | Gi0/0 198.51.100.14/30, Gi0/1 198.51.100.22/30, Lo0 8.8.8.8/32 |

Cables: HQ-R1 Gi0/1 to ISP-A Gi0/0; HQ-R2 Gi0/1 to ISP-B Gi0/0; HQ-R1 Gi0/2 to HQ-R2 Gi0/2; ISP-A Gi0/1 to INET Gi0/0; ISP-B Gi0/2 to ISP-B-UP Gi0/0; ISP-B-UP Gi0/1 to INET Gi0/1.

BGP config: copy the "Phase 3 BGP build (revised)" block from `design-decisions.md`. Leave out `network 198.51.100.8 ...` on ISP-B and `network 203.0.113.16 ...` on INET in this test.

Checks:
1. `show ip bgp summary` on the four ISP-side routers and on HQ-R1 and HQ-R2: every neighbor shows a number, not `Idle` or `Active`.
2. INET `show ip bgp`: 192.0.2.0/24 has two paths and the best path (`>`) is via 198.51.100.13, path `64496 64500`.
3. HQ-R1 `show ip bgp`: 8.8.8.8/32 via 198.51.100.1, path `64496 64498`. HQ-R2 `show ip bgp`: 8.8.8.8/32 via 198.51.100.5, path `64497 64499 64498`.
4. `shutdown` HQ-R1 Gi0/1 and wait. INET `show ip bgp`: only the path via 198.51.100.21 (`64499 64497 64500`). HQ-R1 `show ip route`: `S* 0.0.0.0/0 [250/0] via 10.10.255.66`.
5. `no shutdown` HQ-R1 Gi0/1 and wait. INET returns to the path via 198.51.100.13, and HQ-R1 learns 8.8.8.8/32 from BGP again.

Pings from the routers are not a valid check here: the transit and loopback addresses are not advertised, so replies are lost. Real end-to-end failover is tested in Phase 5, once NAT exists.

## Fallbacks prepared in case a test failed

Outcome: Test 1 failed and was replaced by Test 1R (passed). No other fallback was needed.

| Failure | Fallback |
|---|---|
| Test 1R fails (shorter AS path not preferred, or the floating default does not take over) | Rework the BGP design in `design-decisions.md` before Phase 3 |
| `securityk9` unavailable | Skip Phase 6 and note it as out of scope |
| LACP does not form | Re-enter with ports shut, then `no shutdown`; last resort `mode on` and note it |
| HSRP v2 rejected | Use HSRP v1 and group numbers 1 to 5 |
| AP does not accept WPA2-PSK | Skip Phase 7 |
