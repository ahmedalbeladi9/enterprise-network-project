# Network Security

This document covers every security mechanism in the configuration files under
`/configurations`. 

## DHCP Snooping

- **Problem it solves:** prevents a rogue or accidental DHCP server on the network from handing out
  bogus IP configuration (wrong gateway, wrong DNS) to clients, which is a common vector for
  man-in-the-middle attacks.
- **Implementation:** `ip dhcp snooping` is enabled globally on SW-CORE-A, SW-CORE-B, `sw-it`, `SW-LIB`, `SW-ENG`,
  `sw-admin`, `sw-sec`, and `sw-stu`, each scoped to that switch's own data/voice VLANs.
   Uplink/trunk ports toward the core are
  marked `ip dhcp snooping trust`, so only DHCP server replies arriving from the trusted
  core-facing direction are accepted, DHCP server responses arriving on an access port are dropped.
- **What it prevents:** rogue DHCP servers, and by building the DHCP snooping binding table, it also
  provides the trust database that Dynamic ARP Inspection depends on.
- **Why it matters in an enterprise network:** any user-facing port is a potential attack surface,
  DHCP snooping keeps address assignment authoritative and trusted only at the distribution/core layer.

## Dynamic ARP Inspection (DAI)

- **Problem it solves:** ARP has no authentication, so any device can claim to own any IP address
  (ARP spoofing/poisoning), enabling on-path man-in-the-middle attacks.
- **Implementation:** `ip arp inspection vlan <id>`. DAI validates ARP packets against the DHCP snooping binding table,
  so an ARP reply claiming an IP/MAC pairing that doesn't match a legitimate DHCP lease is dropped.
  Trunk/uplink ports are marked `ip arp inspection trust` network-wide, consistent with DHCP snooping
  trust boundaries.
- **What it prevents:** ARP spoofing/cache poisoning within the protected VLANs, which otherwise allows
  traffic interception or denial of service on the local segment.

## Port Security

- **Problem it solves:** limits which/how many devices can connect to a given access port, preventing
  unauthorized devices (or MAC flooding attacks against the switch's CAM table) from gaining network
  access.
- **Implementation:** every configured access port network-wide uses `switchport port-security`,
  `switchport port-security maximum 2` (one PC + one IP phone), and
  `switchport port-security violation restrict` on a violation, offending traffic is dropped and
  logged, but the port itself stays up (rather than `shutdown`, which would err-disable it and require
  manual/`errdisable recovery` intervention).
- **What it prevents:** unauthorized device connection, MAC address flooding/CAM table exhaustion
  attacks, and unauthorized switch/hub daisy-chaining beyond the two permitted devices per port.
- **Why it matters:** it's a low-overhead first line of defense at every physical access point in the
  building.

## BPDU Guard

- **Problem it solves:** an unauthorized switch or hub plugged into an access port could participate in
  spanning tree and potentially become the root bridge, disrupting the entire Layer 2 topology.
- **Implementation:** `spanning-tree bpduguard enable` is paired with `spanning-tree portfast` on every
  access port network-wide.
- **What it prevents:** if a BPDU is ever received on a PortFast-enabled access port, the port is
  immediately error-disabled, assuming that only end hosts (which never send BPDUs) should be
  connected there.

## Root Guard

- **Problem it solves:** even without BPDU Guard, a connected switch advertising a superior (lower)
  bridge priority could become the new STP root, silently changing the traffic flow across the whole
  network.
- **Implementation:** `spanning-tree guard root` is configured on the SW-CORE ports facing the access layer. This puts the protection where it actually stops the problem: if a switch connected downstream tries to advertise superior BPDU, the core-facing port blocks the takeover, preventing that downstream switch from influencing the intended STP root.
- **What it prevents:** a rogue or misconfigured switch attached to a Root-Guard-protected port from
  ever being elected root, the port is instead put into a root-inconsistent state until the superior
  BPDUs stop.

## VLAN Segmentation

- **Problem it solves:** without segmentation, all devices on a flat network share one broadcast domain
  and can reach each other directly, which is both a performance and a security problem.
- **Implementation:** each department has its own data VLAN and (where phones exist) voice VLAN,
  documented in [`vlan-plan.md`](./vlan-plan.md). Inter-VLAN traffic must pass through a router (R0/R1),
  where it can be filtered.
- **What it prevents:** broadcast storms across the whole enterprise, and unrestricted lateral movement
  between departments that have no business need to talk to each other directly.

## Network Isolation Policy (`STUDENT-ISOLATION`)

- **Problem it solves:** the Student/Lab VLAN is the least trusted segment on the network (shared,
  general-access machines) and should not have direct reach into administrative or security systems.
- **Implementation:** an extended ACL, applied **inbound** on R1's `Gi0/2.50` (the Student VLAN's
  primary gateway interface):
  ```
  ip access-list extended STUDENT-ISOLATION
   deny ip 192.168.50.0 0.0.0.255 192.168.40.0 0.0.0.255
   deny ip 192.168.50.0 0.0.0.255 192.168.60.0 0.0.0.255
   deny tcp 192.168.50.0 0.0.0.255 192.168.100.0 0.0.0.255 eq 22
   permit ip any any
  ```
- **What it prevents:** Student VLAN hosts from reaching the Administration VLAN (192.168.40.0/24) or
  the Security VLAN (192.168.60.0/24) at all, and specifically blocks SSH (TCP/22) from the Student VLAN
  to the shared-services/server segment (192.168.100.0/24) while still permitting general outbound
  connectivity (internet, DHCP relay, etc.) via the trailing `permit ip any any`.
- **Scope note:** this ACL only restricts the Student VLAN's access to Admin, Security, and SSH-to-server
  traffic, it does not restrict Student access to, for example, the Library or Engineering VLANs. This
  reflects a deliberate least-trust posture toward the two most sensitive segments rather than a full
  default-deny policy.

## Restricted Management Access

- **Problem it solves:** device management interfaces (SSH) should only be reachable from authorized
  management subnets, not from every host on the network.
- **Implementation:** a standard ACL, `MGMT-ONLY`, permitting only `192.168.198.0/24` and
  `192.168.199.0/24`, is applied via `access-class MGMT-ONLY in` on `line vty 0 4` of every switch and
  on R0/R1. All devices additionally use
  `ip ssh version 2` and `transport input ssh` on those same lines, so only SSHv2 sessions originating
  from the management subnets are accepted on the first 5 VTY lines.
- **What it prevents:** management-plane access from arbitrary user VLANs, and unencrypted (Telnet)
  management sessions on the protected lines.


## Voice/Data Separation

- **Problem it solves:** IP phones and PCs have different trust levels and traffic patterns, mixing
  them on one VLAN complicates QoS and increases the blast radius of a compromised device.
- **Implementation:** `switchport voice vlan <n>` on access ports (see `vlan-plan.md`) tags phone
  traffic into its own VLAN, separate from the PC's untagged data VLAN, on the same physical port.
- **What it prevents:** a compromised PC directly snooping or interfering with voice traffic on the
  same wire, and simplifies applying different QoS/ACL treatment to voice versus data traffic later.

