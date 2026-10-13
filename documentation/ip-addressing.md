# IP Addressing Plan

## Router-to-router backbone (point-to-point, OSPF Area 0)

| Link | Network | R0 | R1 | R2 |
|---|---|---|---|---|
| R0 and R1 | 10.0.12.0/30 | 10.0.12.1 (Gi0/0) | 10.0.12.2 (Gi0/0) |  |
| R0 and R2 | 10.0.13.0/30 | 10.0.13.1 (Gi0/1) |  | 10.0.13.2 (Gi0/1) |
| R1 and R2 | 10.0.23.0/30 |  | 10.0.23.1 (Gi0/1) | 10.0.23.2 (Gi0/0) |

## OSPF Router IDs

| Router | Router ID | Process |
|---|---|---|
| R0 | 10.10.10.0 | OSPF process 1 |
| R1 | 10.10.10.1 | OSPF process 1 |
| R2 | 10.10.10.2 | OSPF process 1 |

## Department subnets and gateways

See [`vlan-plan.md`](./vlan-plan.md) for the full VLAN-to-subnet mapping. Summary of gateway/HSRP
virtual IPs:

| Subnet | HSRP VIP | Active | Standby |
|---|---|---|---|
| 192.168.10.0/24 | 192.168.10.1 | R0 | R1 |
| 192.168.20.0/24 | 192.168.20.1 | R0 | R1 |
| 192.168.30.0/24 | 192.168.30.1 | R0 | R1 |
| 192.168.110.0/24 | 192.168.110.1 | R0 | R1 |
| 192.168.120.0/24 | 192.168.120.1 | R0 | R1 |
| 192.168.130.0/24 | 192.168.130.1 | R0 | R1 |
| 192.168.199.0/24 | 192.168.199.1 | R0 | R1 |
| 192.168.40.0/24 | 192.168.40.1 | R1 | R0 |
| 192.168.50.0/24 | 192.168.50.1 | R1 | R0 |
| 192.168.60.0/24 | 192.168.60.1 | R1 | R0 |
| 192.168.140.0/24 | 192.168.140.1 | R1 | R0 |
| 192.168.150.0/24 | 192.168.150.1 | R1 | R0 |
| 192.168.160.0/24 | 192.168.160.1 | R1 | R0 |
| 192.168.198.0/24 | 192.168.198.1 | R1 | R0 |

## Server / shared-services segment

| Network | Notes |
|---|---|
| 192.168.100.0/24 | R2 `Gi0/2` = 192.168.100.1 (gateway for this segment). DHCP server at 192.168.100.2, DNS server at 192.168.100.3. both addresses come  from the DHCP server's pool configuration, and all DHCP pools across the network point to these addresses. |
## Switch management IP addresses

| Device | Management VLAN | IP address | Default Gateway |
|---|---|---|---|
| SW-CORE-A | 99 | 192.168.199.11/24 | 192.168.199.2 |
| SW-CORE-B | 99 | 192.168.198.12/24 | 192.168.198.2 |
| sw-it | 99 | 192.168.199.13/24 | 192.168.199.1 |
| SW-LIB | 99 | 192.168.199.14/24 | 192.168.199.1 |
| SW-ENG | 99 | 192.168.199.15/24 | 192.168.199.1 |
| sw-admin | 99 | 192.168.198.16/24 | 192.168.198.1 |
| sw-stu | 99 | 192.168.198.17/24 | 192.168.198.1 |
| sw-sec | 99 | 192.168.198.18/24 | 192.168.198.1 |

## Router subinterface / SVI addressing (full detail)

### R0

| Interface | VLAN tag | IP address | Role |
|---|---|---|---|
| Gi0/2.10 | 10 | 192.168.10.2/24 | Primary gateway (IT data) |
| Gi0/2.20 | 20 | 192.168.20.2/24 | Primary gateway (Library data) |
| Gi0/2.30 | 30 | 192.168.30.2/24 | Primary gateway (Engineering data) |
| Gi0/2.110 | 110 | 192.168.110.2/24 | Primary gateway (IT voice) |
| Gi0/2.120 | 120 | 192.168.120.2/24 | Primary gateway (Library voice) |
| Gi0/2.130 | 130 | 192.168.130.2/24 | Primary gateway (Engineering voice) |
| Gi0/2.199 | 99 | 192.168.199.2/24 | Primary gateway (Site A management) |
| Vlan40 | 40 | 192.168.40.3/24 | Backup gateway (Admin data) |
| Vlan50 | 50 | 192.168.50.3/24 | Backup gateway (Student data) |
| Vlan60 | 60 | 192.168.60.3/24 | Backup gateway (Security data) |
| Vlan140 | 140 | 192.168.140.3/24 | Backup gateway (Admin voice) |
| Vlan150 | 150 | 192.168.150.3/24 | Backup gateway (Student voice) |
| Vlan160 | 160 | 192.168.160.3/24 | Backup gateway (Security voice) |
| Vlan198 | 99 | 192.168.198.3/24 | Backup gateway (Site B management) |

### R1

| Interface | VLAN tag | IP address | Role |
|---|---|---|---|
| Gi0/2.40 | 40 | 192.168.40.2/24 | Primary gateway (Admin data) |
| Gi0/2.50 | 50 | 192.168.50.2/24 | Primary gateway (Student data); `STUDENT-ISOLATION` ACL applied inbound |
| Gi0/2.60 | 60 | 192.168.60.2/24 | Primary gateway (Security data) |
| Gi0/2.140 | 140 | 192.168.140.2/24 | Primary gateway (Admin voice) |
| Gi0/2.150 | 150 | 192.168.150.2/24 | Primary gateway (Student voice) |
| Gi0/2.160 | 160 | 192.168.160.2/24 | Primary gateway (Security voice) |
| Gi0/2.198 | 99 | 192.168.198.2/24 | Primary gateway (Site B management) |
| Vlan10 | 10 | 192.168.10.3/24 | Backup gateway (IT data) |
| Vlan20 | 20 | 192.168.20.3/24 | Backup gateway (Library data) |
| Vlan30 | 30 | 192.168.30.3/24 | Backup gateway (Engineering data) |
| Vlan110 | 110 | 192.168.110.3/24 | Backup gateway (IT voice) |
| Vlan120 | 120 | 192.168.120.3/24 | Backup gateway (Library voice) |
| Vlan130 | 130 | 192.168.130.3/24 | Backup gateway (Engineering voice) |
| Vlan199 | 99 | 192.168.199.3/24 | Backup gateway (Site A management) |

### R2

| Interface | IP address | Role |
|---|---|---|
| Gi0/0 | 10.0.23.2/30 | Link to R1 |
| Gi0/1 | 10.0.13.2/30 | Link to R0 |
| Gi0/2 | 192.168.100.1/24 | Link to servers |
