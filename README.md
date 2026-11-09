# CE Building Network Design

GNS3 model of the network for the new USP Civil Engineering Building, designed for STEMP as part of the CS400 Industry Experience Project (Semester 2, 2026). The configurations here were built and tested as a proof of concept before any hardware is purchased, and they are the starting templates for installation.

## Environment

| Tool | Use |
| --- | --- |
| GNS3 v2.2.16 | Main model of the building, using Cisco IOSv and IOSvL2 images |
| Cisco Packet Tracer | Checks of VLANs, trunks, DHCP relay and wireless settings that the GNS3 images do not support |
| Linux server VMs | DHCP-SRV (10.0.9.15) and CAMPUS-SRV (172.16.10.10) for RADIUS, syslog and NTP |

## Topology

Three-tier hierarchy (core, distribution, access) across four floors, with the MDF/server room on Level 4.

| Device | Role |
| --- | --- |
| EDGE-FW | Internet edge firewall (PERIMETER-IN ACL) |
| CAMPUS-R1 | Connects the building to the campus network over OSPF |
| CORE-SW1 / CORE-SW2 | Layer 3 core in the MDF: SVIs, HSRP gateways, inter-VLAN ACLs, DHCP relay, OSPF 10 |
| DSW-LVL1 to DSW-LVL4 | One distribution switch per floor, dual-homed to both cores |
| ASW-LVLn-01 / -02 | Two 48-port PoE+ access switches per floor |

### Links and redundancy
- **Core:** CORE-SW1 and CORE-SW2 joined by a two-link EtherChannel (Po1) carrying transit VLAN 900 (172.31.0.8/30).
- **Distribution uplinks:** two-link static EtherChannel 802.1Q trunks to each core (Po1 to CORE-SW1, Po2 to CORE-SW2).
- **Access downlinks:** Po11 to ASW-LVLn-01, Po12 to ASW-LVLn-02.
- **Gateways:** HSRP virtual gateway for every VLAN, shared by both cores.
- **Spanning tree:** Rapid PVST+. CORE-SW1 is root for VLANs 10, 30, 50, 70, 90, 110 and 900; CORE-SW2 for 20, 40, 60, 80, 100 and 120.

## VLAN and IP plan

| VLAN | Name | Subnet | Addressing |
| --- | --- | --- | --- |
| 10 | STUDENT | 10.0.0.0/22 | DHCP |
| 20 | GUEST | 10.0.4.0/23 | DHCP |
| 30 | LAB-PCs | 10.0.8.0/24 | DHCP |
| 40 | STAFF | 10.0.6.128/25 | DHCP |
| 50 | CCTV | 10.0.7.0/25 | Static |
| 60 | VOIP | 10.0.6.0/26 | DHCP |
| 70 | BIOMETRICS | 10.0.6.64/26 | Static |
| 80 | AP-MGMT | 10.0.7.128/26 | DHCP |
| 90 | SERVERS | 10.0.9.0/26 | Static |
| 100 | NET-MGMT | 10.0.7.192/27 | Static |
| 110 | PRINTERS | 10.0.7.224/28 | DHCP |
| 120 | BLDG-MGMT | 10.0.9.64/28 | Static |
| 666 | BLACKHOLE | – | Unused ports, shut down |
| 900 | CORE-TRANSIT | 172.31.0.8/30 | Static |
| 998 | NATIVE-UNUSED | – | Native VLAN on all trunks |

DHCP is relayed from each SVI to DHCP-SRV (`ip helper-address 10.0.9.15`). The building range (10.0.0.0–10.0.9.79) sits inside 10.0.0.0/20.

## Security configuration

- **Default-deny inter-VLAN routing:** inbound extended ACL on every core SVI (`V10-STUDENT-IN` to `V120-BLDG-MGMT-IN`), each ending in `deny ip any any`. Printers and building-management devices are reply-only (`established`).
- **Perimeter:** EDGE-FW admits only replies to inside sessions, DNS replies and echo-replies.
- **Trunk hardening:** native VLAN 998, DTP off (`switchport nonegotiate`), allowed VLANs pruned.
- **Layer 2:** DHCP snooping (limit 15 pps), BPDU guard on access ports, root guard on access-facing EtherChannels, port security on camera, reader, printer and AP ports, unused ports in VLAN 666 and shut.
- **Management:** SSH v2 only, restricted by the `MGMT-ONLY` access-class to the NET-MGMT subnet and the jump host (10.0.7.220); RADIUS logins with local fallback; syslog and NTP to CAMPUS-SRV.

## Tests run in GNS3

| Test | Result |
| --- | --- |
| DHCP in every user VLAN | Client gets an address from its own pool through the relay |
| Inter-VLAN policy | Allowed flows work (e.g. Student to printer); all others dropped |
| Core failure | Shutting CORE-SW1 moves every gateway to CORE-SW2 by HSRP within 5 s |
| Uplink failure | Shutting a DSW uplink brings the standby uplink forward by Rapid PVST+ |
| Management access | SSH works only from the jump host; Telnet and other subnets refused |
| Port security | A second MAC address on a camera port shuts the port |

## Not in the GNS3 model

Multi-factor authentication on the jump host and alert rules on the syslog server are designed but not modelled; they must be set up with USP ITS before go-live.

## Repository layout

Running configurations are exported after each change and committed here.
