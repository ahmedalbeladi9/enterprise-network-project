# Troubleshooting Guide

Each entry lists of issues, the commands to check, likely causes given this specific design, and
recommended steps.

## VLAN connectivity problems

**issue:** a host can't reach its default gateway or other hosts on the same VLAN.

**Check:**
```
show vlan brief
show interfaces status
show mac address-table vlan <id>
```

**Likely causes:**
- Access port assigned to the wrong VLAN, or still on the default VLAN 1.
- Port-security violation put the port into a restricted state (`switchport port-security violation
  restrict` drops offending traffic silently rather than shutting the port).

**Steps:** confirm the port's VLAN with `show vlan brief`, confirm it's not err-disabled, and check
`show port-security interface <if>` for violation counts.

## Trunk problems

**issue:** VLANs that should be reachable across a trunk are not unexpected traffic on the native
VLAN.

**Check:**
```
show interfaces trunk
show interfaces <if> switchport
```

**Likely causes:**
- Mismatched `switchport trunk allowed vlan` lists on either end of a link (compare against the
  specific list documented for that link in `network-design.md`).
- Native VLAN mismatch: every trunk in this design should show native VLAN **999** anything else
  indicates a misconfigured or replaced port.

## OSPF neighbor problems

**issue:** two routers that should be adjacent aren't forming a neighbor relationship routes
missing from `show ip route`.

**Check:**
```
show ip ospf neighbor
show ip ospf interface
show ip protocols
```

**Likely causes:**
- The interface is still passive. All three routers use `passive-interface default`, only `Gi0/0` and
  `Gi0/1` (the router-to-router backbone links) are explicitly un-suppressed. Any other interface will
  never form a neighbor by design.
- Area or network-statement mismatch between the two ends.

## HSRP problems

**issue:** clients lose their default gateway during a router failure/recovery, or two routers both
believe they are active.

**Check:**
```
show standby brief
show standby <interface>
```

**Likely causes:**
- Priority mismatch: the primary router for a given VLAN should show priority 110,
  the backup should show the HSRP default(100).
- `preempt` missing on the intended primary router without it a recovered primary will not
  automatically reclaim the active role, leaving the lower-priority router active until it fails.

## STP problems

**issue:** unexpected blocked ports, topology changes, or an access-layer device becoming root.

**Check:**
```
show spanning-tree summary
show spanning-tree root
show spanning-tree vlan <id>
```

**Likely causes:**
- Confirm the root bridge for each VLAN matches the documented design (SW-CORE-A root for Site A VLANs
  / secondary for Site B VLANs, and the mirror on SW-CORE-B).
- An access port received a BPDU and should have been error-disabled by BPDU Guard, check
  `show interfaces status err-disabled` if a port that should be up isn't.

## EtherChannel problems

**issue:** the SW-CORE-A ↔ SW-CORE-B link is not aggregating, or is only using one physical link.

**Check:**
```
show etherchannel summary
show etherchannel port-channel
```

**Likely causes:**
- LACP mode mismatch both ends must use `channel-group 1 mode active` (or one active/one passive)
  this design uses active/active on both switches.
- VLAN allowed-list mismatch between the two physical members of the channel, or between the
  Port-channel interface and its members.

## DHCP problems

**issue:** clients fail to obtain an IP address.

**Check:**
```
show ip interface brief
```

**Likely causes:**
- `ip helper-address 192.168.100.2` relies on reachability to the shared-services segment via R2,
  confirm OSPF has a route to `192.168.100.0/24` from the affected VLAN's gateway router.
- The DHCP server itself is outside the scope of the provided configuration files

## DHCP Snooping issues

**issue:** legitimate DHCP server replies are being dropped.

**Check:**
```
show ip dhcp snooping
show ip dhcp snooping binding
```

**Likely causes:**
- The port facing the legitimate DHCP server path is not marked `ip dhcp snooping trust`. In this
  design, every uplink toward the core should be trusted every access port should not be.
- DHCP snooping isn't enabled for the VLAN in question, check `ip dhcp snooping vlan <list>` on the
  local switch.

## Dynamic ARP Inspection issues

**issue:** legitimate hosts losing connectivity after obtaining a DHCP lease ARP packets being
dropped.

**Check:**
```
show ip arp inspection
show ip arp inspection statistics
```

**Likely causes:**
- The host's ARP traffic doesn't match a binding in the DHCP snooping table (like it has a static IP
  instead of a DHCP lease, DAI has no way to validate it without an ARP ACL exception).

## Port Security violations

**issue:** a port stops passing traffic for one or more MAC addresses after multiple devices are
connected.

**Check:**
```
show port-security interface <if>
show port-security address
```

**Likely causes:**
- More than 2 MAC addresses seen on the port (the configured maximum). Since violation mode is
  `restrict`, the port stays up but excess traffic is dropped and a violation counter increments,
  check the counter rather than assuming the port is down.

## SSH management problems

**issue:** unable to SSH into a device.

**Check:**
```
show ip ssh
show running-config | section line vty
```

**Likely causes:**
- Source IP is outside `192.168.198.0/24` / `192.168.199.0/24`, `access-class MGMT-ONLY in` will
  silently reject the connection.
- Attempting Telnet instead of SSH on `line vty 0 4`, only SSH is permitted there.

## ACL-related connectivity problems

**issue:** Student VLAN hosts unable to reach specific destinations.

**Check:**
```
show access-lists STUDENT-ISOLATION
show ip interface Gi0/2.50   (on R1)
```

**Likely causes:**
- This is expected behavior, not a fault, for traffic from `192.168.50.0/24` toward
  `192.168.40.0/24`, `192.168.60.0/24`, or SSH toward `192.168.100.0/24`, confirm against the documented
  `STUDENT-ISOLATION` rules in `security.md` before treating it as a bug.
- If Student VLAN hosts can't reach destinations that *should* be permitted, confirm the ACL's trailing
  `permit ip any any` hasn't been altered, and that the ACL is applied in the correct direction
  (`in`, on `Gi0/2.50`).
