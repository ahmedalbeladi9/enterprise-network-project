# Network Design

## 1. Architecture overview

The network models a two-site enterprise campus:

- **Site A** : IT, Library, and Engineering departments, distributed by **SW-CORE-A**, routed
  primarily by **R0**.
- **Site B** : Administration, Student/Lab, and Security departments, distributed by **SW-CORE-B**,
  routed primarily by **R1**.

A third router, **R2**, does not carry any department VLANs. It forms the third part of a full-mesh
transit triangle with R0 and R1, and provides the only path out to a shared-services / server segment
(`192.168.100.0/24`) that hosts the DHCP, DNS and web servers used by every department subnet. All inter-router links
run OSPF in a single area (Area 0).

Each access switch (`sw-it`, `SW-LIB`, `SW-ENG`, `sw-admin`, `sw-sec`, `sw-stu`) connects to its site's
core switch over a single 802.1Q trunk carrying its department's data and voice VLANs plus the shared
management VLAN. The two core switches are themselves interconnected by a 2-port LACP EtherChannel.

## 2. Device roles

| Device | Role |
|---|---|
| R0 | Primary router-on-a-stick gateway for Site A department VLANs; standby HSRP gateway for Site B VLANs via a backup trunk to SW-CORE-B; OSPF speaker |
| R1 | Primary router-on-a-stick gateway for Site B department VLANs; standby HSRP gateway for Site A VLANs via a backup trunk to SW-CORE-A; OSPF speaker; hosts the `STUDENT-ISOLATION` ACL |
| R2 | Transit router linking R0 and R1 to each other's backup paths and to the shared-services segment; originates OSPF connectivity to `192.168.100.0/24` |
| SW-CORE-A | Site A distribution/core switch; STP root for Site A VLANs, secondary root for Site B VLANs; DHCP snooping and ARP inspection trust boundary for Site A |
| SW-CORE-B | Site B distribution/core switch; STP root for Site B VLANs, secondary root for Site A VLANs |
| sw-it | Access switch for the IT department (VLAN 10 data / VLAN 110 voice) |
| SW-LIB | Access switch for the Library department (VLAN 20 data / VLAN 120 voice) |
| SW-ENG | Access switch for the Engineering department (VLAN 30 data / VLAN 130 voice) |
| sw-admin | Access switch for Administration (VLAN 40 data / VLAN 140 voice); Dynamic ARP Inspection enabled |
| sw-sec | Access switch for the Security department (VLAN 60 data / VLAN 160 voice); Dynamic ARP Inspection and Root Guard enabled on every access port |
| sw-stu | Access switch for Students/Lab (VLAN 50 data only); Dynamic ARP Inspection enabled; dual trunk uplinks |

## 3. Layer 2 / Layer 3 boundary

Each department access switch is purely Layer 2: it trunks its VLANs up to its site's core switch and
carries no SVI other than the shared management VLAN (99), which is for switch management and SSH access. Routing between VLANs happens exclusively at
R0/R1 via router-on-a-stick subinterfaces (`Gi0/2.x`) for the primary path, and via SVIs on the router's
internal EtherSwitch module for the backup path. The core switches themselves do not route between
VLANs, they are pure Layer 2 distribution points that simply forward each VLAN's trunk traffic up to
whichever router is currently active for that VLAN.

## 4. OSPF

- **Process:** OSPF process 1, single area (**Area 0**) on all three routers, appropriate for this
  scale of network, since a single area avoids the added complexity of multi-area design while still gaining OSPF's fast convergence and loop-free SPF-calculated paths over
  a static or distance-vector alternative.
- **Router IDs:** R0 = `10.10.10.0`, R1 = `10.10.10.1`, R2 = `10.10.10.2` (manually assigned).
- **Passive interfaces:** all three routers use `passive-interface default` and then explicitly
  un-suppress only the router-to-router backbone links (R0/R1: `Gi0/0`, `Gi0/1`; R2: `Gi0/0`, `Gi0/1`).
  This means OSPF hello packets are never sent out on any VLAN subinterface, SVI, or the server-facing
  link, those networks are still advertised into OSPF, but no neighbor adjacency can form on them.
  This is good practice: it stops OSPF from running on end-user-facing segments
  while still routing those subnets.
- **Neighbor relationships:** R0–R1, R0–R2, and R1–R2 are expected to form full adjacencies across the
  three `/30` backbone links, since all three networks are explicitly un-passive and in the same area.
- **Why OSPF fits here:** with three routers and 14 department subnets spread across two sites plus a
  server segment, OSPF gives automatic failover if a backbone link or a router goes down, without
  requiring static routes to be maintained by hand on every router as VLANs are added or subnets
  change, this scales far better than static routing as the network grows.

## 5. HSRP (First Hop Redundancy)

Every department VLAN subnet has **two potential Layer 3 gateways** (one primary, one standby)
running HSRP:

- Site A VLANs (10/20/30/110/120/130/199): **R0 is active** (`standby <group> priority 110`,
  `preempt`), **R1 is standby** (no priority configured, so it defaults to 100).
- Site B VLANs (40/50/60/140/150/160/198): **R1 is active** (priority 110, preempt), **R0 is standby**
  (default priority 100).

`preempt` is configured on every HSRP group's primary router, so when the primary router recovers after
a failure it automatically reclaims the active role rather than leaving the (lower-priority) standby
router active indefinitely.

**How this protects end devices:** every host on a department VLAN is configured with a single default
gateway IP(HSRP virtual IP `192.168.10.1`). That virtual IP is not tied to one physical
router, whichever router currently holds the active role for that VLAN answers ARP for the virtual IP
and virtual MAC. If the active router (or its uplink to that VLAN) fails, the standby router detects
the loss of HSRP hellos, takes over the active role, and begins answering for the same virtual IP/MAC
within seconds and the end devices never need to change their configured gateway.

## 6. Switching architecture

- **Access ports:** configured with `switchport mode access`, an access VLAN, and (where phones are
  deployed) a voice VLAN, letting one physical port carry both a PC (untagged, data VLAN) and an IP
  phone (802.1Q-tagged, voice VLAN).
- **Trunk ports:** every inter-switch and switch-to-router link is `switchport mode trunk` with an
  explicit `switchport trunk allowed vlan` list and use **VLAN 999** as **native VLAN**, which is
  standard hardening against native-VLAN-mismatch and double-tagging
  VLAN-hopping attacks.
- **Rapid PVST+:** `spanning-tree mode rapid-pvst` is configured on every switch, giving per-VLAN
  spanning trees with fast convergence.
- **STP root placement:** SW-CORE-A is configured as the primary root (priority 24576) for the Site A
  VLANs and secondary root (28672) for the Site B VLANs; SW-CORE-B is the mirror image, primary root
  for Site B VLANs, secondary root for Site A VLANs. This deliberately anchors the root bridge at the
  distribution layer (never at an access switch) and gives each site a defined backup root if its own
  core switch fails.
- **EtherChannel / LACP:** the two Fast Ethernet links between SW-CORE-A and SW-CORE-B are bundled into
  `Port-channel1` using `channel-group 1 mode active` (LACP) on both ends, giving a single logical
  trunk with combined bandwidth and link-level redundancy between the two core switches.
- **PortFast + BPDU Guard:** every user-facing access port has `spanning-tree portfast` (skips the
  listening/learning delay so PCs and phones get network access immediately) paired with
  `spanning-tree bpduguard enable` (immediately error-disables the port if it receives a BPDU like if
  someone plugs in an unauthorized switch or hub, protecting the STP topology from being altered by an
  edge device).
- **Root Guard:** enabled on every access port on `sw-sec` (`spanning-tree guard root`). This prevents
  a device connected to that switch from ever being allowed to become the STP root, even if it
  advertises a superior (lower) bridge priority, protecting the security department's edge from a
  rogue-switch root-bridge takeover attack.
