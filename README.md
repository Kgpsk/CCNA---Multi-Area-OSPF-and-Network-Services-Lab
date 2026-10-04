# Multi-Area OSPF and Network Services Lab

A complete enterprise-style network built in Cisco Packet Tracer for my CCNA class, demonstrating multi-area OSPF, DHCP relay, DNS, HTTP, and FTP services.

Author: Kushan Sameera
Course: CCNA - Cisco Certified Network Associate
Tool: Cisco Packet Tracer

---

## Overview

This lab simulates a real-world enterprise network divided into three OSPF areas:

- Area 10 - Client VLANs
- Area 0 - Backbone and server farm
- Area 20 - Client VLANs on a Layer 3 switch

The project demonstrates dynamic routing, router-on-a-stick, DHCP relay across areas, and centralized network services.

---

## Topology

Area 10 connects to Area 0 which connects to Area 20.

Devices used:

- Router0 - Internal router for Area 10
- Router1 - Area Border Router between Area 10 and Area 0
- Router2 - Internal router for Area 0, hosts the server farm
- Router3 - Area Border Router between Area 0 and Area 20
- Cisco 3560 Layer 3 Switch - Area 20
- 2960 Switches - Access layer
- Four servers - DHCP, HTTP, DNS, FTP
- Multiple PCs across all areas

---

## IP Addressing Plan

Area 10 Client VLANs on Router0:

- VLAN 10 - 192.168.1.0/24 - Gateway 192.168.1.1
- VLAN 20 - 192.168.2.0/24 - Gateway 192.168.2.1
- VLAN 30 - 192.168.3.0/24 - Gateway 192.168.3.1
- VLAN 40 - 192.168.4.0/24 - Gateway 192.168.4.1
- VLAN 50 - 192.168.5.0/24 - Gateway 192.168.5.1

Backbone Links:

- Router0 to Router1 - 10.1.1.0/30 - Area 10
- Router1 to Router2 - 10.1.1.4/30 - Area 0
- Router2 to Router3 - 10.1.1.8/30 - Area 0

Area 20 VLANs on Router3 and 3560:

- VLAN 100 - 172.16.1.0/29 - Gateway 172.16.1.1
- VLAN 200 - 172.16.1.8/29 - Gateway 172.16.1.9
- VLAN 300 - 172.16.1.16/29 - Gateway 172.16.1.17
- VLAN 400 - 172.16.1.24/29 - Gateway 172.16.1.25
- VLAN 500 - 172.16.1.32/29 - Gateway 172.16.1.33

Server Farm in Area 0:

- Server0 - DHCP - 200.100.10.10
- Server1 - HTTP - 200.100.10.11
- Server2 - DNS - 200.100.10.12
- Server3 - FTP - 200.100.10.13

---

## OSPF Configuration

Router0 - Area 10

router ospf 1
 network 192.168.0.0 0.0.7.255 area 10
 network 10.1.1.0 0.0.0.3 area 10

Router1 - Area Border Router

router ospf 1
 network 10.1.1.0 0.0.0.3 area 10
 network 10.1.1.4 0.0.0.3 area 0

Router2 - Area 0

router ospf 1
 network 10.1.1.4 0.0.0.3 area 0
 network 10.1.1.8 0.0.0.3 area 0
 network 200.100.10.0 0.0.0.255 area 0

Router3 - Area Border Router

router ospf 1
 network 10.1.1.8 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.7 area 20
 network 172.16.1.8 0.0.0.7 area 20
 network 172.16.1.16 0.0.0.7 area 20
 network 172.16.1.24 0.0.0.7 area 20
 network 172.16.1.32 0.0.0.7 area 20

---

## Router on a Stick Configuration

On Router0 sub-interfaces:

interface g0/0.10
 encapsulation dot1q 10
 ip address 192.168.1.1 255.255.255.0
 ip helper-address 200.100.10.10

Repeat for sub-interfaces 20, 30, 40, and 50 with their matching VLANs and IP addresses.

---

## DHCP Relay Configuration

Configured on every client-facing sub-interface on Router0 and Router3:

ip helper-address 200.100.10.10

This forwards DHCP broadcasts from Area 10 and Area 20 clients to the DHCP server in Area 0.

---

## Services Configured

DHCP - Server0 hosts pools for VLANs 10 to 50 and VLANs 100 to 500.

DNS - Server2 hosts A records for HTTP and FTP hostnames.

HTTP - Server1 serves a custom index page.

FTP - Server3 allows file transfer with login cisco and password cisco.

---

## Issues Faced and Fixes Applied

This section documents the real troubleshooting steps taken while building this lab.

### Issue 1 - OSPF Area Mismatch on Router2 and Router3

Problem:
Router2 kept flooding the console with:
%OSPF-4-ERRRCV: Received invalid packet: mismatch area ID, from backbone area must be virtual-link but not found from 10.1.1.9

Cause:
The Serial link between Router2 and Router3 (10.1.1.8/30) was configured in Area 20 on one end and Area 0 on the other. OSPF neighbors must be in the same area on the same link.

Fix:
Rebuilt the OSPF process on both routers and placed the Serial link in Area 0 on both ends.

router ospf 1
 network 10.1.1.8 0.0.0.3 area 0

Result:
Neighbors reached FULL state. Errors stopped.

### Issue 2 - Duplicate OSPF Network Statement Override

Problem:
Even after adding "network 10.1.1.8 0.0.0.3 area 0", the interface still showed Area 20 in "show ip ospf interface serial 0/0/0".

Cause:
An older, stale "network 10.1.1.8 0.0.0.3 area 20" statement was still in the config. The first matching network statement wins, so the old one kept overriding the new one.

Fix:
Wiped OSPF completely and rebuilt from scratch:

no router ospf 1
router ospf 1
 network 10.1.1.8 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.7 area 20
 network 172.16.1.8 0.0.0.7 area 20
 network 172.16.1.16 0.0.0.7 area 20
 network 172.16.1.24 0.0.0.7 area 20
 network 172.16.1.32 0.0.0.7 area 20

Then:
clear ip ospf process

Result:
OSPF adjacency reached FULL.

### Issue 3 - DHCP Clients Not Receiving IP Addresses

Problem:
PCs in Area 10 did not receive IPs from the DHCP server in Area 0.

Cause:
No "ip helper-address" on the client-facing sub-interfaces. DHCP broadcasts do not cross routers by default.

Fix:
Added on every client-facing sub-interface on Router0 and Router3:

ip helper-address 200.100.10.10

Result:
PCs received IP, gateway, and DNS automatically via DHCP.

### Issue 4 - Missing Route to Server Farm on Router3

Problem:
PC10 could ping its gateway 172.16.1.1 but ping to 200.100.10.10 returned "Destination host unreachable".

Cause:
Router2 was not advertising the 200.100.10.0/24 network into OSPF.

Fix:
On Router2:

router ospf 1
 network 200.100.10.0 0.0.0.255 area 0

Result:
Router3 learned the route (O IA) and clients could reach the servers.

### Issue 5 - Serial Link Status


---

## Verification Commands

show ip ospf neighbor
show ip route ospf
show ip ospf interface brief
show ip ospf database
show running-config section router ospf
show running-config include helper
show vlan brief
show interfaces trunk
ipconfig release
ipconfig renew

---

## What This Lab Demonstrates

- Multi-area OSPF design with one backbone and two non-backbone areas
- ABR configuration on Router1 and Router3
- Router on a stick with 802.1Q encapsulation
- DHCP relay across OSPF areas
- Centralized services in Area 0
- Real-world troubleshooting of OSPF area mismatches
- End-to-end connectivity across all three areas

---

## Lessons Learned

- OSPF neighbors must be in the same area on the same link
- The first matching network statement wins in OSPF
- DHCP broadcasts need ip helper-address to cross routers
- Every network you want advertised must have a network statement
- Always verify with show ip ospf interface and show ip ospf neighbor

---

## About the Author

Kushan Sameera

CCNA student passionate about networking and sharing knowledge with the community.

Built this lab from scratch as part of my CCNA coursework to demonstrate real-world networking concepts and troubleshooting.

If this helped you, feel free to star or share.
