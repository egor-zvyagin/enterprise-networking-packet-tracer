# Enterprise Multi-VLAN Routing, Switching, HSRP Gateway Redundancy & OSPF/EIGRP Dynamic Redistribution (Cisco IOS / Packet Tracer)

**Author:** Egor Zvyagin | Melbourne, VIC  
**Profiles:** [LinkedIn](https://linkedin.com/in/egor-zvyagin-843628168) | [GitHub](https://github.com/egor-zvyagin) | [Credly](https://credly.com/users/egor-zvyagin)  
**Specialisation:** Enterprise Routing & Switching, Cisco IOS, VLSM/CIDR, OSPFv2, EIGRP, HSRPv2 Redundancy, PPP CHAP WAN, IPv6 Tunnelling & Network Forensics

---

## Executive Summary

This repository documents an enterprise-grade hybrid campus and WAN routing architecture designed in **Cisco Packet Tracer** and configured using **Cisco IOS CLI**. The deployment integrates hierarchical networking best practices across Core, Distribution, and Edge layers to deliver high availability, deterministic first-hop redundancy, multi-protocol dynamic route convergence, and transitionary dual-stack IPv6 overlay connectivity.

The project validates core network engineering proficiencies across three production implementations:
1. **Perimeter Security & High Availability:** Point-to-Point Protocol (PPP) WAN links authenticated via Challenge Handshake Authentication Protocol (CHAP), and Hot Standby Router Protocol version 2 (HSRPv2) with preemption and active uplink tracking.
2. **Multi-Protocol Dynamic Routing & Mutual Redistribution:** Simultaneous operation of Open Shortest Path First (OSPFv2 Process 22 Area 0) and Enhanced Interior Gateway Routing Protocol (EIGRP AS 111), linked via deterministic two-way route redistribution and default quad-zero (`0.0.0.0/0`) route propagation.
3. **IPv6 Transition Engineering & Fault Isolation:** Triage, root-cause analysis, and resolution of three pre-configured transmission failures across an IPv6-over-IPv4 manual overlay tunnel (`tunnel mode ipv6ip`).

---

## Topology Architecture & Addressing Layout

```
                                  +---------------------+
                                  |    ISP Provider     |
                                  | Loopback: 8.8.8.8/32|
                                  +----------+----------+
                                             |
                                  PPP Serial | S0/0/1: .226/30
                                  CHAP (WIN) | S0/0/0: .225/30
                                             |
                                  +----------+----------+
                                  |    Gateway Router   |
                                  | Router-ID: 11.1.1.1 |
                                  +----+-----------+----+
                                       |           |
            OSPFv2 Area 0 (G0/1: .1/24)|           |OSPFv2 Area 0 (G0/2: .1/24)
            Transit: 172.16.1.0/24     |           |Transit: 172.16.2.0/24
                                       |           |
                     +-----------------+           +-----------------+
                     |                                               |
           +---------+----------+                          +---------+----------+
           | Multilayer Switch  |                          |  Standby Router    |
           |      (MLS)         |                          |   (Standby_Rtr)    |
           | Router-ID: 4.4.4.2 |                          | Router-ID: 4.4.4.3 |
           | HSRP Prio: 110     |                          | HSRP Prio: 100     |
           | Tracking: G0/2     |                          | Mode: Standby      |
           +---------+----------+                          +---------+----------+
                     \                                               /
                      \_________ HSRPv2 Group 22 VIP: 4.4.4.254 _____/
                                                |
                                      +---------+----------+
                                      |    FTP Server      |
                                      | IP: 4.4.4.1/24     |
                                      | Def GW: 4.4.4.254  |
                                      +--------------------+

  [ EIGRP AS 111 Branch ]
  Gateway (S0/0/1: 10.10.1.1/30) <=====> (S0/1/1: 10.10.1.2/30) RtrEIGRP (RID: 9.9.9.9)
                                                                       |
                                                                 Branch LAN: 10.20.1.0/24

  [ IPv6 Tunnel Network ]
  R0v6 (S0/0/0: 192.0.2.1/30) <====== Tunnel0 (2001:DB8:CAFE:1::/64) ======> R1v6 (S0/0/0: 192.0.2.2/30)
  Loopback0: 2001:DB8:ACAD:1000::1/64                                      Loopback1: 2001:DB8:ACAD:2000::1/64
```

### IP Subnetting & Interface Allocation Matrix

| Device Name | Interface | IPv4 / IPv6 Address | Subnet Mask / Prefix | Function / Description |
|:---|:---|:---|:---|:---|
| **Gateway** | `Serial0/0/0` | `209.165.200.225` | `255.255.255.252` (`/30`) | WAN Link to ISP (PPP CHAP) |
| **Gateway** | `Serial0/0/1` | `10.10.1.1` | `255.255.255.252` (`/30`) | WAN Interconnect to EIGRP Domain |
| **Gateway** | `GigabitEthernet0/1` | `172.16.1.1` | `255.255.255.0` (`/24`) | Core Transit Link to MLS (OSPF Area 0) |
| **Gateway** | `GigabitEthernet0/2` | `172.16.2.1` | `255.255.255.0` (`/24`) | Core Transit Link to Standby Router (OSPF Area 0) |
| **ISP** | `Serial0/0/1` | `209.165.200.226` | `255.255.255.252` (`/30`) | Upstream Service Provider Interface |
| **ISP** | `Loopback0` | `8.8.8.8` | `255.255.255.255` (`/32`) | Simulated Global DNS Destination |
| **MLS** | `GigabitEthernet0/1` | `4.4.4.2` | `255.255.255.0` (`/24`) | HSRP Primary Gateway (VIP: `4.4.4.254`) |
| **MLS** | `GigabitEthernet0/2` | `172.16.1.2` | `255.255.255.0` (`/24`) | Core Transit Uplink to Gateway |
| **Standby_Rtr** | `GigabitEthernet0/0` | `4.4.4.3` | `255.255.255.0` (`/24`) | HSRP Standby Gateway (VIP: `4.4.4.254`) |
| **Standby_Rtr** | `GigabitEthernet0/1` | `172.16.2.2` | `255.255.255.0` (`/24`) | Redundant Uplink to Gateway |
| **RtrEIGRP** | `Serial0/1/1` | `10.10.1.2` | `255.255.255.252` (`/30`) | Branch Uplink to Headquarters Gateway |
| **RtrEIGRP** | `GigabitEthernet0/0` | `10.20.1.1` | `255.255.255.0` (`/24`) | Regional Branch LAN Default Gateway |
| **FTP Server** | `NIC` | `4.4.4.1` | `255.255.255.0` (`/24`) | Corporate FTP Host (Default GW: `4.4.4.254`) |
| **R0v6** | `Tunnel0` | `2001:DB8:CAFE:1::1` | `/64` | IPv6 Manual Tunnel Endpoint |
| **R0v6** | `Loopback0` | `2001:DB8:ACAD:1000::1`| `/64` | Target Verification IPv6 Loopback |
| **R1v6** | `Tunnel0` | `2001:DB8:CAFE:1::2` | `/64` | IPv6 Manual Tunnel Endpoint |
| **R1v6** | `Loopback1` | `2001:DB8:ACAD:2000::1`| `/64` | Ingress Verification IPv6 Loopback |

---

## Technical Specifications & Engineering Implementation

### 1. PPP with CHAP WAN Authentication & HSRPv2 High Availability
* **PPP Encapsulation & Mutual CHAP:** Standard HDLC framing on WAN links replaced with Point-to-Point Protocol (`encapsulation ppp`). Secure two-way challenge handshake authentication configured between `Gateway` and `ISP` using pre-shared secrets (`WIN`), preventing unauthorized carrier link injection.
* **HSRPv2 Gateway Redundancy:** Configured HSRP Group 22 on `MLS` and `Standby_Rtr` sharing Virtual IP `4.4.4.254`.
  - `MLS` configured with priority `110` and `preempt` enabled, designating it the active default gateway.
  - Interface tracking (`standby 22 track GigabitEthernet0/2 20`) dynamically decreases MLS priority by 20 upon uplink failure, allowing `Standby_Rtr` (priority 100) to preempt and maintain continuous egress traffic for the server farm.
* **Server Farm Gateway Resolution:** `FTP Server` configured with static IP `4.4.4.1/24` pointing directly to virtual gateway `4.4.4.254`, validating transparent layer 3 failover without host reconfiguration.

### 2. Multi-Protocol Dynamic Routing & Two-Way Route Redistribution
* **OSPFv2 Process 22 (Area 0):** Provisioned across `Gateway` (Router-ID: `11.1.1.1`), `MLS` (Router-ID: `4.4.4.2`), and `Standby_Rtr` (Router-ID: `4.4.4.3`). Injected default quad-zero route (`default-information originate`) from the edge gateway across the entire OSPF domain.
* **EIGRP AS 111 (Branch Autonomous System):** Provisioned across `Gateway` (Router-ID: `10.10.10.10`) and `RtrEIGRP` (Router-ID: `9.9.9.9`), advertising branch LAN subnet `10.20.1.0/24`.
* **Two-Way Route Redistribution:**
  - Injected EIGRP routes into OSPF: `redistribute eigrp 111 metric 1 subnets` allowing internal campus routers to reach branch subnets.
  - Injected OSPF routes and default route into EIGRP: `redistribute ospf 22 metric 1 1 1 1 1` and `redistribute static`, supplying branch routers with full corporate and Internet reachability.

### 3. IPv6-Over-IPv4 Manual Overlay Tunnel Troubleshooting (Root-Cause Analysis)
Identified and corrected three critical misconfigurations on pre-existing routers `R0v6` and `R1v6`:
1. **Encapsulation Mode Mismatch:** Tunnel defaulted to generic GRE IPv4. Configured `tunnel mode ipv6ip` on both routers to ensure proper 41-protocol encapsulation of native IPv6 packets.
2. **Invalid Tunnel Source Binding:** Corrected `tunnel source` on `R0v6` from an inactive FastEthernet interface to active WAN link `Serial0/0/0`.
3. **Inverted Tunnel Destination IP:** Corrected `tunnel destination` on `R1v6` from pointing to its own local loopback to remote endpoint `192.0.2.1`.
* **Verification:** Confirmed end-to-end ICMPv6 reachability between `Loopback1` (`2001:DB8:ACAD:2000::1`) and `Loopback0` (`2001:DB8:ACAD:1000::1`) with a 100% success rate.

---

## Cisco IOS Configuration Library (`/configs`)

The complete, unabridged Cisco IOS configuration scripts are located in the [`configs/`](./configs) directory:

1. [`configs/01_Gateway_Router.ios`](./configs/01_Gateway_Router.ios): Core gateway router configuration (PPP CHAP, OSPF Process 22, EIGRP AS 111, mutual redistribution, static default routing).
2. [`configs/02_ISP_Router.ios`](./configs/02_ISP_Router.ios): Internet Service Provider edge router configuration (PPP CHAP response, public Internet loopbacks, return routing).
3. [`configs/03_Multilayer_Switch_MLS.ios`](./configs/03_Multilayer_Switch_MLS.ios): Multilayer switch configuration (`ip routing`, HSRPv2 active gateway with priority 110, preemption, and uplink tracking).
4. [`configs/04_Standby_Router.ios`](./configs/04_Standby_Router.ios): Standby redundancy router configuration (HSRPv2 standby partner with priority 100, OSPF Area 0 integration).
5. [`configs/05_RtrEIGRP.ios`](./configs/05_RtrEIGRP.ios): Branch router configuration (EIGRP AS 111, branch LAN support, serial WAN encapsulation).
6. [`configs/06_IPv6_Tunnel_Routers.ios`](./configs/06_IPv6_Tunnel_Routers.ios): Dual-stack edge routers `R0v6` and `R1v6` configuration and root-cause troubleshooting log.

---

## Privileged EXEC Verification & Evidence Outputs

### 1. OSPFv2 Neighbor Adjacency Table (`Gateway`)
```text
Gateway# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
4.4.4.2           1   FULL/BDR        00:00:34    172.16.1.2      GigabitEthernet0/1
4.4.4.3           1   FULL/DROTHER    00:00:36    172.16.2.2      GigabitEthernet0/2
```

### 2. HSRPv2 Status & Interface Tracking (`MLS`)
```text
MLS# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Gi0/1       22   110 P Active  local           4.4.4.3         4.4.4.254

MLS# show standby
GigabitEthernet0/1 - Group 22 (version 2)
  State is Active
    2 state changes, last state change 00:14:28
  Virtual IP address is 4.4.4.254
  Active virtual MAC address is 0000.0c9f.f016
    Local virtual MAC address is 0000.0c9f.f016 (v2 default)
  Hello time 3 sec, hold time 10 sec
  Preemption enabled
  Active router is local
  Standby router is 4.4.4.3, priority 100 (expires in 8.124 sec)
  Priority 110 (configured 110)
    Track interface GigabitEthernet0/2 state Up decrement 20
```

### 3. EIGRP Neighbor Adjacency & Topology (`Gateway`)
```text
Gateway# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(111)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   10.10.1.2               Se0/0/1                  13 00:18:42   12   200  0  4

Gateway# show ip route eigrp
D EX    10.20.1.0/24 [170/2297856] via 10.10.1.2, 00:18:44, Serial0/0/1
```

### 4. PPP Link Encapsulation & CHAP Status (`Gateway`)
```text
Gateway# show interfaces Serial0/0/0
Serial0/0/0 is up, line protocol is up 
  Hardware is WIC MBRD Serial
  Description: WAN-UPLINK-TO-ISP-PRIMARY
  Internet address is 209.165.200.225/30
  Encapsulation PPP, LCP Open
  Open: IPCP
  PPP CHAP authentication validated with peer 'ISP'
```

### 5. IPv6 Overlay Tunnel End-to-End Ping (`R1v6`)
```text
R1v6# ping ipv6 2001:DB8:ACAD:1000::1 source Loopback1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:ACAD:1000::1, timeout is 2 seconds:
Packet sent with a source address of 2001:DB8:ACAD:2000::1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms
```

---

## Included Deliverables

* [`Enterprise_Multi_VLAN_OSPF_Topology.pka`](./Enterprise_Multi_VLAN_OSPF_Topology.pka): Interactive Cisco Packet Tracer assessment file ready for live inspection, simulation, and topology traversal.
* [`Enterprise_Network_Configuration_and_Verification.docx`](./Enterprise_Network_Configuration_and_Verification.docx): Technical configuration reference handbook containing complete subnet planning, interface mapping, and verification evidence.
* [`configs/`](./configs): Six standalone Cisco IOS `.ios` CLI configuration scripts covering all topology routers and multilayer switches.

---

## Professional Reference & Contact

Authored and maintained by **Egor Zvyagin**  
- **Email:** `egor.dobryak1@gmail.com`  
- **LinkedIn:** [linkedin.com/in/egor-zvyagin-843628168](https://linkedin.com/in/egor-zvyagin-843628168)  
- **GitHub:** [github.com/egor-zvyagin](https://github.com/egor-zvyagin)  
- **Credly:** [credly.com/users/egor-zvyagin](https://credly.com/users/egor-zvyagin)
