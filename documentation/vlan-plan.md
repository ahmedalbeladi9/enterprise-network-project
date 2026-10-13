# VLAN Plan

All VLAN IDs below are taken directly from the device configuration files under `/configurations`.

## Department / user VLANs

| VLAN ID | Purpose | Department | Data/Voice | Subnet | Default Gateway (HSRP VIP) | Active Router | Access Switch |
|---|---|---|---|---|---|---|---|
| 10 | IT data | IT | Data | 192.168.10.0/24 | 192.168.10.1 | R0 (priority 110) | sw-it |
| 110 | IT voice | IT | Voice | 192.168.110.0/24 | 192.168.110.1 | R0 (priority 110) | sw-it |
| 20 | Library data | Library | Data | 192.168.20.0/24 | 192.168.20.1 | R0 (priority 110) | SW-LIB |
| 120 | Library voice | Library | Voice | 192.168.120.0/24 | 192.168.120.1 | R0 (priority 110) | SW-LIB |
| 30 | Engineering data | Engineering | Data | 192.168.30.0/24 | 192.168.30.1 | R0 (priority 110) | SW-ENG |
| 130 | Engineering voice | Engineering | Voice | 192.168.130.0/24 | 192.168.130.1 | R0 (priority 110) | SW-ENG |
| 40 | Administration data | Administration | Data | 192.168.40.0/24 | 192.168.40.1 | R1 (priority 110) | sw-admin |
| 140 | Administration voice | Administration | Voice | 192.168.140.0/24 | 192.168.140.1 | R1 (priority 110) | sw-admin |
| 50 | Student / lab data | Student | Data | 192.168.50.0/24 | 192.168.50.1 | R1 (priority 110) | sw-stu |
| 60 | Security department data | Security | Data | 192.168.60.0/24 | 192.168.60.1 | R1 (priority 110) | sw-sec |
| 160 | Security department voice | Security | Voice | 192.168.160.0/24 | 192.168.160.1 | R1 (priority 110) | sw-sec |

## Infrastructure VLANs

| VLAN ID | Purpose | Subnet(s) | Notes |
|---|---|---|---|
| 99 | Management | 192.168.199.0/24 (Site A) and 192.168.198.0/24 (Site B) | The same VLAN tag (99) is used on both sites' access/trunk configuration, but two different subnets are configured on it depending on which side of the network a device sits. |
| 999 | Native |  | Configured as `switchport trunk native vlan 999` on every trunk in the network. No SVI exists and no access port is assigned to it a native VLAN, which is standard practice to reduce VLAN-hopping risk via native VLAN mismatch or double-tagging attacks. |

## Non-VLAN routed segment referenced by this project

| Network | Purpose | Notes |
|---|---|---|
| 192.168.100.0/24 | Server / shared-services segment (labeled `To-SW-CORE-C` on R2's `Gi0/2`) | Hosts the DHCP server targeted by `ip helper-address 192.168.100.2` on all R0/R1 department subinterfaces. |

## Voice VLAN handling

Access ports on sw-it, SW-LIB, SW-ENG, sw-admin, and sw-sec are configured with
`switchport voice vlan <n>`, allowing a single access port to carry both a PC (data VLAN, untagged) and
an IP phone (voice VLAN, 802.1Q-tagged) simultaneously, the standard Cisco two-device-per-port model.
`sw-stu` access ports have no voice VLAN configured, consistent with a student/lab segment with no
deployed IP phones.
