# Verification Commands

These are the relevant `show` commands for verifying each technology implemented in this project.

## General reachability / routing

```
show ip interface brief
show ip route
show ip protocols
```

## OSPF

```
show ip ospf neighbor
show ip ospf interface
show ip ospf interface brief
show ip ospf database
```

## HSRP

```
show standby brief
show standby <interface>
```

## VLANs and trunking

```
show vlan brief
show interfaces trunk
show interfaces <if> switchport
```

## Spanning Tree / Rapid PVST+

```
show spanning-tree summary
show spanning-tree root
show spanning-tree vlan <id>
show spanning-tree inconsistentports
show interfaces status err-disabled
```

## EtherChannel / LACP

```
show etherchannel summary
show etherchannel port-channel
show lacp neighbor
```

## DHCP Snooping

```
show ip dhcp snooping
show ip dhcp snooping binding
```

## Dynamic ARP Inspection

```
show ip arp inspection
show ip arp inspection statistics
show ip arp inspection interfaces
```

## Port Security

```
show port-security
show port-security interface <if>
show port-security address
```

## Access Control Lists

```
show access-lists
show ip interface <if>   (confirm which ACL is applied inbound/outbound)
```

## SSH Management

```
show ip ssh
show ssh
show running-config | section line vty
```

