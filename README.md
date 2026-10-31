# IT Skills — Multi-Site Enterprise Network

![Cisco Packet Tracer](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7)
![Routing](https://img.shields.io/badge/Routing-OSPF%20Area%200-2E75B6)
![Switching](https://img.shields.io/badge/Switching-VLAN%20%7C%20802.1Q-1F3864)
![Security](https://img.shields.io/badge/Security-Extended%20ACLs-C0392B)

A six-site enterprise network built and verified in Cisco Packet Tracer. **IT Skills** is a fictional multinational company with its headquarters in Germany and five branch offices in Egypt, the United Kingdom, India, Saudi Arabia (KSA) and Dubai. The branches connect to HQ over point-to-point WAN links, every site is segmented with VLANs, and DNS, web and TFTP services are centralized at headquarters.

> This is a lab simulation built for learning and portfolio purposes. The `.pkt` file is the authoritative implementation; this README summarizes its design and verification results.

## Contents

- [Topology](#topology)
- [At a glance](#at-a-glance)
- [Architecture](#architecture)
- [Addressing plan](#addressing-plan)
- [WAN and OSPF](#wan-and-ospf)
- [Network services](#network-services)
- [Guest VLAN security](#guest-vlan-security)
- [Verification](#verification)
- [Useful commands](#useful-commands)
- [Limitations and future work](#limitations-and-future-work)
- [Skills demonstrated](#skills-demonstrated)
- [Repository structure](#repository-structure)
- [How to open the project](#how-to-open-the-project)

## Topology

![Packet Tracer logical topology](images/Multi_Site_Enterprise_Network_Topology.png)

## At a glance

| Area | Implementation |
|---|---|
| Sites | 6: Germany HQ, Egypt, UK, India, KSA, Dubai |
| Routing | OSPF process 1, area 0 |
| Segmentation | 9 VLANs per site, 802.1Q trunks, native VLAN 99 |
| Inter-VLAN routing | Multilayer core switch (SVIs) at every site |
| Addressing | /16 per site, /24 per VLAN, /30 per WAN link |
| Services | DNS, HTTP and TFTP at HQ (VLAN 70); DHCP on the core switches |
| Security | Extended ACL applied inbound on each site's Guest VLAN interface |

## Architecture

The WAN is a **hub-and-spoke** design with Germany HQ as the hub. Each branch edge router connects to `GER-EDGE` over a dedicated serial /30 link, so traffic between branches transits headquarters. `GER-EDGE` also has an upstream link to an ISP router.

Every site uses the same three layers:

- **Access:** user PCs and a wireless access point on an access switch (`SW1`). HQ uses two, `GER-SW1` and `GER-SW2`.
- **Core:** a multilayer switch hosting the VLAN interfaces, performing inter-VLAN routing, serving DHCP and running OSPF.
- **Edge:** a router connecting the site to the WAN, linked to the core over a routed transit subnet (`10.<site>.254.0/30`).

| Site | Address block | Edge router | Core switch |
|---|---|---|---|
| Germany HQ | 10.10.0.0/16 | GER-EDGE | GER-CORE |
| Egypt | 10.20.0.0/16 | EGY-EDGE | EGY-CORE |
| United Kingdom | 10.30.0.0/16 | UK-EDGE | UK-CORE |
| India | 10.40.0.0/16 | IND-EDGE | IND-CORE |
| KSA | 10.50.0.0/16 | KSA-EDGE | KSA-CORE |
| Dubai | 10.60.0.0/16 | DXB-EDGE | DXB-CORE |

## Addressing plan

Every site follows the same convention: **`10.<site>.<VLAN ID>.0/24`**, with the core SVI as the default gateway at `.1`. For example, Germany VLAN 20 is `10.10.20.0/24` (gateway `10.10.20.1`) and Dubai VLAN 20 is `10.60.20.0/24` (gateway `10.60.20.1`).

| VLAN | Name | Purpose |
|---|---|---|
| 10 | MANAGEMENT | Network management |
| 20 | IT | IT department |
| 30 | HR | Human Resources |
| 40 | FINANCE | Finance |
| 50 | SALES | Sales |
| 60 | TRAINING | Training users |
| 70 | SERVERS | Central servers (HQ) |
| 80 | GUEST | Guest devices |
| 99 | NATIVE-MGMT | Native VLAN on trunks |

DHCP pools run on each core switch for the client VLANs. Addresses `.1`–`.20` are excluded, so the first lease is `.21` (for example, the India IT PC holds `10.40.20.21`). Servers use static addresses.

## WAN and OSPF

All routers and core switches run OSPF in a single area (process 1, area 0). Router IDs follow a fixed pattern to make neighbor tables easy to read.

| Site | WAN subnet | GER-EDGE interface | Edge router ID | Core router ID |
|---|---|---|---|---|
| Germany HQ | n/a | n/a | 1.1.1.1 | 11.11.11.11 |
| Egypt | 10.255.0.0/30 | Serial0/0/0 | 2.2.2.2 | 12.12.12.12 |
| United Kingdom | 10.255.0.4/30 | Serial0/0/1 | 3.3.3.3 | 13.13.13.13 |
| India | 10.255.0.8/30 | Serial0/1/0 | 4.4.4.4 | 14.14.14.14 |
| KSA | 10.255.0.12/30 | Serial0/1/1 | 5.5.5.5 | 15.15.15.15 |
| Dubai | 10.255.0.16/30 | Serial0/2/0 | 6.6.6.6 | 16.16.16.16 |

`GER-EDGE` has six OSPF neighbors, all in FULL state: the five branch edge routers and `GER-CORE`. The ISP link is not an OSPF neighbor.

![GER-EDGE OSPF neighbors](images/show_ip_ospf_neighbor_in_GER_Edge.png)

## Network services

| Device | Address | Role |
|---|---|---|
| GER-DNS | 10.10.70.10/24 | Central DNS. `www.itskills.com` resolves to `10.10.70.20` |
| GER-WEB | 10.10.70.20/24 | Company website, tested over HTTP |
| GER-TFTP | 10.10.70.30/24 | TFTP service |

The servers sit in VLAN 70 behind `10.10.70.1` (the VLAN 70 SVI on `GER-CORE`). Branches reach them through OSPF-learned routes via `GER-EDGE`.

![Company website loaded from a branch PC](images/show_the_project_website_in_Egypt_branch.png)

## Guest VLAN security

Each site applies an extended ACL named `<SITE>-GUEST-IN` inbound on its VLAN 80 interface. Guests are kept away from internal private networks, while still reaching approved central services and non-private destinations.

```
Extended IP access list DXB-GUEST-IN
    permit icmp any host 10.60.80.1
    permit udp any host 10.10.70.10 eq domain
    permit tcp any host 10.10.70.10 eq domain
    permit tcp any host 10.10.70.20 eq www
    permit tcp any host 10.10.70.20 eq 443
    deny ip any 10.0.0.0 0.255.255.255
    deny ip any 172.16.0.0 0.15.255.255
    deny ip any 192.168.0.0 0.0.255.255
    permit ip any any
```

**Design notes**

- **Order matters.** The central servers sit inside `10.0.0.0/8`, so their permit entries must come before the private-range denies. Everything else internal, including the TFTP server and all internal VLANs, falls under the deny.
- **Scope.** This is an internal segmentation control, not a full Internet security policy. The final `permit ip any any` allows every destination outside the private ranges.
- **HTTPS.** Port 443 is permitted, but only HTTP access was tested.
- **DHCP.** Initial lease broadcasts pass the final permit. Unicast renewals to the gateway address would match the `10.0.0.0/8` deny, so a production policy should add an explicit DHCP client-to-server entry.

## Verification

| Test | Result |
|---|---|
| OSPF neighbors on `GER-EDGE` | 6 neighbors in FULL state (5 branch edges + `GER-CORE`) |
| `KSA-PC-IT` to `10.40.20.21` (India) | 4/4 replies, 0% loss, average 13 ms, TTL 123 |
| `KSA-PC-IT` to `10.10.70.10` (DNS) | 4/4 replies, 0% loss, average 6 ms, TTL 124 |
| `EGY-PC-IT` browsing `http://www.itskills.com` | Page loads (DNS and HTTP across the WAN) |
| VLAN database on `DXB-CORE` | All 9 VLANs active |
| Trunk on `DXB-CORE` Gi0/2 | 802.1Q, native VLAN 99, VLANs 10–80 and 99 allowed |
| SVIs on `EGY-CORE` | VLANs 10–80 and 99 up/up, plus the routed uplink |
| Guest ACL on `DXB-CORE` | 9 entries as designed |
| Guest PCs at all six sites | Gateway ping succeeds, private hosts blocked, website loads |

**Reading the TTL.** Packet Tracer PCs start at TTL 128. A reply TTL of 123 on the KSA-to-India ping means five routed hops (KSA-CORE, KSA-EDGE, GER-EDGE, IND-EDGE, IND-CORE), which confirms that branch-to-branch traffic transits the HQ hub as designed.

![Cross-branch ping from KSA to India](images/cross_branch_ping.png)

![Guest ACL on DXB-CORE](show_access-lists_in_DXB_Core.png)

## Useful commands

```
show ip ospf neighbor
show ip route ospf
show ip interface brief
show vlan brief
show interfaces trunk
show access-lists <SITE>-GUEST-IN
show ip dhcp binding
```

## Limitations and future work

- ISP routing and Internet access are outside the tested scope.
- HTTPS is permitted in the Guest ACL but was not tested.
- Planned improvements: SSH-only management with AAA, OSPF authentication on the WAN links, port security with DHCP snooping and dynamic ARP inspection, and redundant WAN and core paths with first-hop redundancy.

## Skills demonstrated

OSPF · VLANs and 802.1Q trunking · Inter-VLAN routing on Layer 3 switches · DHCP · DNS, HTTP and TFTP services · Extended ACLs · IP addressing and subnetting · Cisco IOS CLI verification and troubleshooting

## Repository structure

```
.
├── IT_Skills_Multisite_Network_Final.pkt
├── README.md
├── docs/
│   └── IT_Skills_Multisite_Network_Documentation.pdf
└── images/
    ├── topology.png
    ├── ospf-neighbors.png
    ├── website.png
    ├── cross-branch-ping.png
    └── guest-acl.png
```

## How to open the project

1. Install Cisco Packet Tracer.
2. Open `IT_Skills_Multisite_Network_Final.pkt`.
3. Wait for OSPF to converge (a minute or two in Realtime mode), then run the commands above from the device CLIs.

## Author

**Mostafa Ramadan**

- LinkedIn: [linkedin.com/in/eng-mostafa-ramadan](https://www.linkedin.com/in/eng-mostafa-ramadan)
- GitHub: [github.com/Eng-MostafaRamadan](https://github.com/Eng-MostafaRamadan)
