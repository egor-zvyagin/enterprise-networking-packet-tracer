# Enterprise Multi-VLAN Routing, Switching & Network Security Topology (Cisco IOS / Packet Tracer)

**Author:** Egor Zvyagin | Melbourne, VIC  
**Profiles:** [LinkedIn](https://linkedin.com/in/egor-zvyagin-843628168) | [GitHub](https://github.com/egor-zvyagin) | [Credly](https://credly.com/users/egor-zvyagin)  
**Specialisation:** Enterprise Routing & Switching, Cisco IOS, VLSM/CIDR, OSPF, NAT/PAT & Layer 3/4 Network Security

---

## Executive Summary

This repository presents a fully provisioned and validated enterprise campus network topology designed in **Cisco Packet Tracer** and configured using **Cisco IOS CLI**. The architecture incorporates hierarchical network design principles (Core, Distribution, and Access layers), robust Layer 2 isolation, scalable Layer 3 dynamic routing, centralized network services, and hardened perimeter security controls.

The project demonstrates complete end-to-end network implementation: calculating optimal variable-length subnet masks (VLSM), establishing redundant dynamic routing via Open Shortest Path First (OSPF), provisioning IP infrastructure services (DHCP Relay, Dynamic NAT/PAT), and securing inter-segment traffic via Extended Access Control Lists (ACLs).

---

## Topology Architecture & Component Overview

```
                                  [ WAN / Internet ]
                                          |
                              [ Edge / Border Router ]
                               (NAT/PAT, Default Route)
                                     /         \
                         [ Serial / OSPF ]   [ Serial / OSPF ]
                                   /             \
                       [ HQ Distribution Router ]  [ Branch Office Router ]
                                  |                         |
                       [ 802.1Q Trunk Link ]      [ 802.1Q Trunk Link ]
                                  |                         |
                       [ Distribution Switch ]     [ Branch Access Switch ]
                              /        \                    |
                     [ Trunk ]          [ Trunk ]       [ Access ]
                        /                    \              |
            [ Core Access Switch 1 ]  [ Core Access Switch 2 ]   [ Endpoints ]
                   |                           |
            +------+------+             +------+------+
            |             |             |             |
        [ VLAN 10 ]   [ VLAN 20 ]   [ VLAN 30 ]   [ VLAN 99 ]
        Management     Executive       Staff        Native/Mgmt
```

---

## Technical Specifications & Configuration Highlights

### 1. IPv4 / IPv6 Subnetting & Addressing Scheme (VLSM / CIDR)
* **Variable-Length Subnet Masking (VLSM):** Segmented private IPv4 address allocations (`10.0.0.0/8` and `192.168.0.0/16`) to maximize address efficiency and eliminate broadcast domain saturation.
* **Point-to-Point WAN Links:** Subnetted dedicated `/30` and `/31` prefixes between router serial and Gigabit Ethernet interconnections, conserving IP space.
* **Dual-Stack IPv6 Deployment:** Configured global unicast (`2001:db8::/64`) and link-local (`fe80::/64`) addresses across all router interfaces with modified EUI-64 identifier formatting.

### 2. Layer 2 Switching & VLAN Segmentation
* **802.1Q VLAN Tagging:** Configured high-bandwidth trunk links across switches using standard 802.1Q framing, carrying tagged multi-VLAN traffic between distribution and access layers.
* **Native VLAN Security:** Reassigned the default untagged Native VLAN from VLAN 1 to an isolated, non-routed VLAN (VLAN 99) across all trunk connections to mitigate VLAN hopping and double-tagging attacks.
* **Port Security Hardening:** Restricted access switchports to dynamic sticky MAC address learning (`switchport port-security mac-address sticky`) with strict violation shutdown triggers.

### 3. Inter-VLAN Routing & Layer 3 Integration
* **Router-on-a-Stick (ROAS):** Created logical sub-interfaces on routing hardware (`GigabitEthernet0/0/0.10`, `.20`, `.30`) mapped directly to respective 802.1Q VLAN IDs via `encapsulation dot1Q <vlan-id>`.
* **Default Gateway Assignment:** Assigned the initial usable IP of each subnet as the virtual default gateway, ensuring deterministic first-hop routing for all endpoint tiers.

### 4. Dynamic Routing (Single-Area OSPFv2 & OSPFv3)
* **OSPF Adjacencies:** Formed deterministic dynamic routing adjacencies across interior routers within OSPF Area 0, eliminating static route maintenance across multi-hop subnets.
* **Metric & Cost Calculations:** Tuned interface bandwidth parameters (`auto-cost reference-bandwidth 1000`) to guarantee Gigabit Ethernet links take precedence over legacy FastEthernet or Serial pathways.
* **Passive Interfaces:** Enforced `passive-interface` declarations on all user-facing LAN sub-interfaces, suppressing unnecessary OSPF Hello multicast packets and protecting LANs from rogue router advertisements.
* **Default Route Redistribution:** Injected quad-zero default routes (`ip route 0.0.0.0 0.0.0.0 [Next-Hop]`) across the OSPF domain via `default-information originate`.

### 5. Infrastructure Network Services
* **DHCP Relay Services:** Configured `ip helper-address` on router VLAN sub-interfaces, forwarding broadcast DHCP requests across Layer 3 boundaries to a centralized enterprise DHCP server.
* **Network Address Translation (NAT/PAT):** Deployed Port Address Translation (NAT Overload) on the boundary router, mapping hundreds of private internal hosts to a single routable public IP via an inside source access list and overload statement (`ip nat inside source list 1 interface GigabitEthernet0/0/1 overload`).

### 6. Perimeter Network Security & Access Control Lists (ACLs)
* **Extended Access Control Lists (ACLs):** Filtered Layer 3/4 traffic by source IP, destination IP, protocol, and port number:
  - Permitted internal subnets outbound HTTP (`80`), HTTPS (`443`), and DNS (`53`) access while blocking incoming unsolicited internet sessions.
  - Restricted administrative SSH access (`port 22`) exclusively to authorized Management VLAN workstations.
  - Isolated guest subnets from internal file repositories, databases, and network management consoles.

---

## Cisco IOS Configuration Extracts

### 1. Router Sub-Interface Inter-VLAN Routing & DHCP Relay
```cisco
interface GigabitEthernet0/0/0.10
 description Management VLAN Gateway
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.99.10
!
interface GigabitEthernet0/0/0.20
 description Corporate Operations Gateway
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.99.10
!
interface GigabitEthernet0/0/0.30
 description Guest WiFi Gateway
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.99.10
```

### 2. OSPFv2 Routing Process Configuration
```cisco
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 1000
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/0/0.10
 passive-interface GigabitEthernet0/0/0.20
 passive-interface GigabitEthernet0/0/0.30
 default-information originate
```

### 3. Dynamic PAT (NAT Overload) & Edge Security ACL
```cisco
ip access-list standard NAT_LAN_PERMIT
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.255
!
interface GigabitEthernet0/0/0
 ip nat inside
!
interface GigabitEthernet0/0/1
 ip nat outside
!
ip nat inside source list NAT_LAN_PERMIT interface GigabitEthernet0/0/1 overload
```

---

## Verification & Diagnostic Commands

The topology was systematically verified across all operational layers using standard Cisco IOS privileged exec commands:

* `show ip interface brief`: Validates physical and line protocol status across all physical ports and virtual sub-interfaces.
* `show vlan brief`: Confirms port-to-VLAN assignments and active broadcast domain status.
* `show interfaces trunk`: Asserts 802.1Q trunk negotiation, native VLAN assignment, and allowed VLAN lists across uplinks.
* `show ip route`: Verifies OSPF routing table population, administrative distances (`[110/cost]`), and active gateway pathways.
* `show ip nat translations`: Inspects real-time translations mapping inside local IP:Port pairs to inside global addresses.
* `show access-lists`: Asserts hit counters on configured security filtering rules.

---

## Included Deliverables

* [`Enterprise_Multi_VLAN_OSPF_Topology.pka`](./Enterprise_Multi_VLAN_OSPF_Topology.pka): Interactive Cisco Packet Tracer assessment topology ready for simulation, configuration inspection, and packet traversal verification.
* [`Enterprise_Network_Configuration_and_Verification.docx`](./Enterprise_Network_Configuration_and_Verification.docx): Complete technical configuration handbook, containing IP address assignment tables, interface maps, and verification evidence.

---

## Licensing & Operational Use

Authored and maintained by Egor Zvyagin for professional portfolio demonstration, technical interview verification, and network engineering reference.
