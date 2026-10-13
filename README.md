# Secure Multi-Site Enterprise Network using Cisco Packet Tracer

A two-site enterprise campus network built and simulated in Cisco Packet Tracer, demonstrating VLAN
segmentation, redundant Layer 3 gateways, single-area OSPF routing, Rapid PVST+ switching, and a
layered Layer 2/Layer 3 security baseline (DHCP Snooping, Dynamic ARP Inspection, Port Security, BPDU
Guard, Root Guard, and restricted SSH management).

## Overview

This project models an enterprise network split across two department sites, Site A (IT, Library,
Engineering) and Site B (Administration, Student/Lab, Security). each with its own distribution/core
switch and its own primary Layer 3 gateway router. A third router provides transit connectivity and a
path to a shared-services segment. Every department VLAN has a redundant default gateway via HSRP, and
every inter-switch/router link is a hardened, explicitly-scoped 802.1Q trunk.

The full architecture, VLAN plan, IP addressing, and every security mechanism are documented under
[`/documentation`](./documentation) and summarized below.

## Objectives

- Design a segmented, multi-department enterprise LAN using VLANs and voice/data separation.
- Provide first-hop gateway redundancy for every department subnet using HSRP.
- Route between all VLANs and two physical sites using single-area OSPF.
- Harden Layer 2 against common attacks: rogue DHCP servers, ARP spoofing, MAC flooding, unauthorized
  switches, and STP root-bridge takeover.
- Restrict device management to SSH from defined management subnets only.
- Apply a targeted network isolation policy for the least-trusted segment (Student/Lab).
- Practice documenting and reviewing a network the way it would be reviewed as production
  infrastructure.

## Network Architecture

See [`documentation/network-design.md`](./documentation/network-design.md) for the full write-up, including a device roles, 
OSPF and HSRP configuration, and the switching
architecture. In short:

- **R0** is the primary gateway (router-on-a-stick) for all Site A VLANs and the HSRP standby gateway
  for Site B VLANs.
- **R1** mirrors R0: primary gateway for Site B VLANs, standby for Site A VLANs.
- **R2** is a transit router connecting R0 and R1 to each other and to a shared-services segment that
  hosts the DHCP server used network-wide.
- **SW-CORE-A** and **SW-CORE-B** are the Layer 2 distribution switches for each site, interconnected by
  a 2-port LACP EtherChannel, and each configured as the STP root for their own site's VLANs and
  secondary root for the other site's VLANs.
- Six access switches (`sw-it`, `SW-LIB`, `SW-ENG`, `sw-admin`, `sw-sec`, `sw-stu`) connect end-user
  devices, each trunked up to its site's core switch.

## Network Topology

![Enterprise Network Topology](screenshots/network-topology.png)


## Technologies Used

- Cisco IOS (routers and switches)
- VLANs, 802.1Q trunking, voice VLANs
- Inter-VLAN routing (router-on-a-stick + router-based SVIs)
- OSPF (single area)
- HSRP
- Rapid PVST+
- EtherChannel / LACP
- DHCP Snooping
- Dynamic ARP Inspection (DAI)
- Port Security
- BPDU Guard
- Root Guard
- SSH (SSHv2) device management
- Standard and extended ACLs (management restriction and VLAN isolation)
- DHCP relay (`ip helper-address`)

## VLAN and IP Addressing

Full tables: [`documentation/vlan-plan.md`](./documentation/vlan-plan.md) and
[`documentation/ip-addressing.md`](./documentation/ip-addressing.md).

| VLAN | Department | Subnet | Gateway |
|---|---|---|---|
| 10 / 110 | IT (data/voice) | 192.168.10.0/24 / 192.168.110.0/24 | R0 primary |
| 20 / 120 | Library (data/voice) | 192.168.20.0/24 / 192.168.120.0/24 | R0 primary |
| 30 / 130 | Engineering (data/voice) | 192.168.30.0/24 / 192.168.130.0/24 | R0 primary |
| 40 / 140 | Administration (data/voice) | 192.168.40.0/24 / 192.168.140.0/24 | R1 primary |
| 50  | Student/Lab (data) | 192.168.50.0/24  | R1 primary |
| 60 / 160 | Security (data/voice) | 192.168.60.0/24 / 192.168.160.0/24 | R1 primary |
| 99 | Management | 192.168.199.0/24 (Site A) / 192.168.198.0/24 (Site B) | R0 / R1 |

## Routing

Single-area OSPF (Area 0) runs across R0, R1, and R2 on their backbone links, with every VLAN subnet
advertised as a passive network. Details, router IDs, and a sanitized configuration excerpt are in
[`documentation/network-design.md`](./documentation/network-design.md#4-ospf).

## High Availability

Every department subnet has two HSRP-capable gateways (a primary at priority 110 with `preempt`, and a
standby at the default priority). The Site A and Site B core switches are also interconnected redundantly
via a 2-port LACP EtherChannel. See
[`documentation/network-design.md`](./documentation/network-design.md#5-hsrp-first-hop-redundancy).

## Switching

Rapid PVST+ with deliberately placed root bridges at the distribution layer, an unused native VLAN
(999) on every trunk, PortFast + BPDU Guard on all access ports, and Root Guard on the Security
department's access ports. See
[`documentation/network-design.md`](./documentation/network-design.md#6-switching-architecture).

## Security

DHCP Snooping, Dynamic ARP Inspection, Port Security, BPDU Guard, Root Guard, VLAN segmentation,
Student VLAN isolation, and restricted SSH management are all covered in depth, including what each
mechanism protects against, in [`documentation/security.md`](./documentation/security.md).

## Management

All devices are managed exclusively over SSHv2, with the primary VTY lines restricted to the
management subnets (`192.168.198.0/24`, `192.168.199.0/24`) via a standard ACL
(`access-class MGMT-ONLY in`). See [`documentation/security.md`](./documentation/security.md) for
more details.

## Project Structure

```
secure-multi-site-enterprise-network-project/
│
├── README.md
├── topology/
│   ├── secure-multi-site-network.pkt
├── configurations/
│   ├── routers/
│   │   ├── R0.txt
│   │   ├── R1.txt
│   │   └── R2.txt
│   └── switches/
│       ├── SW-CORE-A.txt
│       ├── SW-CORE-B.txt
│       ├── sw-it.txt
│       ├── SW-LIB.txt
│       ├── SW-ENG.txt
│       ├── sw-admin.txt
│       ├── sw-sec.txt
│       └── sw-stu.txt
├── documentation/
│   ├── network-design.md
│   ├── vlan-plan.md
│   ├── ip-addressing.md
│   ├── security.md
│   └── troubleshooting.md
├── screenshots/
│   └── Full topology screenshot.png
│   ├── DHCP server pools.png
│   ├── HSRP roles.png
│   └── OSPF adjacencies.png
└── verification/
    └── verification-commands.md
```

## Configuration Files

Each device's  running-configuration is stored individually under `/configurations`, split
into `routers/` and `switches/`, named after the device's hostname. These are the actual configurations
from the project.

## Verification and Testing

A full list of relevant `show` commands (like routing, OSPF, HSRP, VLANs/trunking, STP, EtherChannel,
DHCP Snooping, DAI, Port Security, and ACLs) is documented in
[`verification/verification-commands.md`](./verification/verification-commands.md). 

## Learning Outcomes

This project demonstrates practical, CCNA/ network-engineering-level skills in:

- Enterprise VLAN design and voice/data separation
- Router-on-a-stick and SVI-based inter-VLAN routing
- Single-area OSPF design, passive-interface hardening, and router ID planning
- First-hop redundancy (HSRP) design and priority/preemption planning
- Rapid PVST+ root bridge placement and Layer 2 loop prevention
- EtherChannel/LACP link aggregation
- Layer 2 attack mitigation: DHCP Snooping, Dynamic ARP Inspection, Port Security, BPDU Guard, Root
  Guard
- Management-plane hardening via SSH-only access and source-restricted ACLs
- Reviewing and documenting an existing network configuration.
