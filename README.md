# Dual-ISP Enterprise Network Lab

A dual-homed, two-site enterprise network built in Cisco Packet Tracer. An HQ and a branch connect to two simulated ISPs over eBGP, with HSRP gateway redundancy, OSPF internally, LACP EtherChannel, ACL-based segmentation, SSH-only management, and (planned) an IPsec site-to-site VPN and branch WLAN. Every feature is verified with saved command output, and failures are tested and written up as NOC-style incident reports and runbooks.

<!-- TODO (Phase 0): add topology image at topology/topology.png -->
![Topology](topology/topology.png)

> **Status:** Phase 0 in progress. Update the table below as each phase is finished.

## Project status

| Phase | Scope | Status |
|---|---|---|
| 0 | IP plan, topology, design decisions | In progress |
| 1 | VLANs, trunks, LACP EtherChannel, RSTP | Not started |
| 2 | HSRP gateways, OSPF | Not started |
| 3 | ISP edge, eBGP/iBGP, NAT, DHCP, DNS | Not started |
| 4 | SSH hardening, ACL policy, port security | Not started |
| 5 | Failover tests, incident reports, runbooks | Not started |
| 6 | IPsec site-to-site VPN (HQ to branch) | Not started |
| 7 | Branch wireless (WPA2, guest network) | Not started |

## Skills demonstrated

| Area | What was built |
|---|---|
| Addressing | VLSM plan from one /16, loopbacks and point-to-point links |
| Layer 2 | VLANs, 802.1Q trunks with native VLAN 998, RSTP root placement, PortFast and BPDU Guard, LACP |
| First-hop redundancy | HSRP with preempt on every HQ VLAN, aligned with the STP root |
| Interior routing | Single-area OSPF with passive interfaces |
| Exterior routing | eBGP to two ISPs, iBGP between edge routers, local preference and AS-path prepending |
| Services | DHCP with relay, DNS, PAT on both edges |
| Security | SSH-only management, named extended ACLs, port security, unused ports parked in VLAN 999 |
| VPN and wireless | IPsec site-to-site, WPA2 branch WLAN (planned) |
| Operations | Failover tests, incident reports, runbooks |

## Design summary

- **Company:** fictional logistics firm with an HQ in Cebu and one branch, dual-homed to ISP-A (primary) and ISP-B (backup).
- **Autonomous systems (documentation range):** company 64500, ISP-A 64496, ISP-B 64497, simulated Internet 64498.
- **Public block:** 192.0.2.0/24, advertised to both ISPs. Outbound prefers ISP-A (local preference 200). Inbound is steered with AS-path prepending toward ISP-B.
- **Internal addressing:** 10.10.0.0/16 allocated by VLSM. See [`docs/ip-plan.md`](docs/ip-plan.md).
- **OSPF scope:** internal reachability only (transit link, loopbacks, LAN prefixes). BGP carries everything external and is not redistributed into OSPF.
- **Branch:** single uplink to ISP-B. Private traffic to HQ will use the IPsec tunnel (Phase 6).

Details and trade-offs: [`docs/design-decisions.md`](docs/design-decisions.md) and [`docs/bgp-policy.md`](docs/bgp-policy.md).

## Key results

<!-- TODO (Phase 5): replace each "TBD" with a number you measured, using a continuous ping. -->

| Test | Fault | Result |
|---|---|---|
| ISP-A failure | Shut HQ-R1 link to ISP-A | TBD replies lost; traffic shifted to ISP-B; restored on recovery |
| HSRP active failure | Shut HQ-R1 LAN interface | TBD replies lost; HQ-R2 became Active |
| EtherChannel member failure | Shut one member port | TBD replies lost |
| STP root failure | Power off HQ-SW1 | TBD seconds to reconverge |
| Branch VPN failure | Take down the ISP-B link | TBD (Phase 6) |

Test procedures and evidence: [`failover-tests/`](failover-tests/).

## Repository map

```
.
├── docs/              IP plan, design decisions, BGP and ACL policy
├── topology/          draw.io source and exported PNG
├── packet-tracer/     staged .pkt files (v1 to v5)
├── configs/           one running-config per device
├── verification/      show-command evidence per feature
├── failover-tests/    fault injection with measured recovery
├── incidents/         NOC-style incident reports (index in incidents/README.md)
├── runbooks/          BGP, OSPF, and interface-flapping procedures
└── evidence/          screenshots (kept minimal)
```

## How to open the lab

1. Install Cisco Packet Tracer (built with version 9.0.1.0858).
2. Open the `.pkt` file for the stage you want to inspect:

| File | Adds |
|---|---|
| `v1-layer2.pkt` | VLANs, trunks, EtherChannel, RSTP |
| `v2-routing.pkt` | HSRP, OSPF |
| `v3-bgp-dual-isp.pkt` | ISPs, BGP, NAT, DHCP, DNS |
| `v4-security.pkt` | SSH, ACLs, port security |
| `v5-vpn-wireless.pkt` | IPsec VPN, branch WLAN |

3. Device configs are in [`configs/`](configs/) if you prefer reading to opening the lab.
4. Credentials in published configs are placeholders only.

## Known limitations

- **NAT with dual-homing:** replies to traffic that left via ISP-B can arrive via ISP-A, at a router with no translation. Normal traffic follows ISP-A, so this appears only in edge cases. Production designs handle this differently.
- **Packet Tracer BGP is basic:** it is enough to show sessions, policy, and failover, but does not model real internet scale or convergence timing. Measured times are indicative, not production figures.
- **ISP-B carries the branch VPN:** an ISP-B outage takes down branch connectivity to HQ.
- **Wireless uses two APs:** Packet Tracer access points support one SSID each. A production design would use one AP or a controller with VLAN-mapped SSIDs.
- **Simulator quirks:** occasional slow ping replies, settings not showing in `show` output, and pings failing despite correct configuration until devices are power-cycled. Save the running config before power-cycling.

## Troubleshooting and operations

- Incident reports: [`incidents/README.md`](incidents/README.md)
- Runbooks: [`runbooks/bgp-session-down.md`](runbooks/bgp-session-down.md), [`runbooks/ospf-neighbor-down.md`](runbooks/ospf-neighbor-down.md), [`runbooks/interface-flapping.md`](runbooks/interface-flapping.md)

## What I learned

<!-- TODO (Phase 8): 3 to 5 sentences in your own words: what was hardest, what you would change, what you would add next (for example, route filtering, BGP communities, or automation of config backups). -->

## Author

Pol, licensed electrical engineer (Philippines) working toward a network engineering and NOC career. [https://www.linkedin.com/in/john-paul-villaban-387429190/](https://www.linkedin.com/in/john-paul-villaban-387429190/)