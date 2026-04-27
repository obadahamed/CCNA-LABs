# CCNA Lab 2 — Multi-Site Network: OSPF, GRE, VLANs, HSRP, EtherChannel, DHCP & DNS

> **Author:** XENOS
> **Tool:** Cisco Packet Tracer
> **Score:** 89 / 100
> **Topics:** OSPF · GRE Tunnel · VLANs · Inter-VLAN Routing · HSRP · EtherChannel (LACP/PAgP) · DHCP · DNS · Multilayer Switching
> **Difficulty:** Advanced

---

## 📋 Lab Overview

This lab simulates a full enterprise two-site network. Site A and Site B are connected through an ISP using OSPF as the dynamic routing protocol and a GRE tunnel for logical site-to-site connectivity. Each site has a structured Layer 2/Layer 3 design with VLANs, redundant uplinks via EtherChannel, first-hop redundancy via HSRP, centralized DHCP, and DNS services.

| Technology | Role |
|-----------|------|
| OSPF (Single Area) | Dynamic routing across all routers and multilayer switches |
| GRE Tunnel | Logical overlay between R1 and R2 over the public WAN |
| VLANs | Layer 2 segmentation at both sites |
| Inter-VLAN Routing | Performed by multilayer switches via SVIs |
| HSRP | Default gateway redundancy for end hosts |
| EtherChannel — LACP | Bundled uplinks on Site A switches |
| EtherChannel — PAgP | Second bundle on Site A switches |
| DHCP Server | Centralized IP assignment for Site B VLANs |
| DNS Server | Name resolution hosted in Site A VLAN 30 |

---

## 🗺️ Network Topology

### WAN / ISP Layer

| Link | Subnet | Devices |
|------|--------|---------|
| ISP ↔ R2 | `200.0.0.0/30` | ISP — R2 |
| ISP ↔ R1 | `100.0.0.0/30` | ISP — R1 Gi0/0 |
| R1 ↔ MSS | `192.168.105.0/30` | R1 Gi0/1 — MSS |
| R2 ↔ Distribution SW | `172.16.23.0/30` | R2 — Dist Switch |
| GRE Tunnel R1↔R2 | `192.168.100.0/30` | Tunnel10 on R1 and R2 |

### Site B — Right Side

| Link | Subnet |
|------|--------|
| MSS ↔ R1 | `192.168.105.0/30` |
| MSS ↔ MS3 | `192.168.135.0/30` |
| MSS ↔ MS2 | `192.168.145.0/30` |
| MS3 ↔ MS2 | `192.168.123.0/30` |
| MS3 downlink | `192.168.113.0/30` |
| MS2 downlink | `192.168.124.0/30` |
| DHCP Server | `192.168.5.0/30` |

### Site A VLANs — Left Side

| VLAN | Subnet | Hosts |
|------|--------|-------|
| VLAN 30 | `172.16.30.0/24` | DNS Server |
| VLAN 40 | `172.16.40.0/24` | PC10, PC11 |
| VLAN 50 | `172.16.50.0/24` | PC9 |
| VLAN 60 | `172.16.60.0/24` | PC8 |
| VLAN 70 | `172.16.70.0/24` | PC7 |
| VLAN 80 | `172.16.80.0/24` | CCNA Server |

### Site B VLANs — Right Side

| VLAN | Subnet | Hosts |
|------|--------|-------|
| VLAN 10 | `192.168.10.0/24` | PC1, PC3, PC5, PC7 |
| VLAN 20 | `192.168.20.0/24` | PC2, PC4, PC6, PC8 |

---

## ⚙️ Configuration

### 1. OSPF — Dynamic Routing

```
! R1
router ospf 1
 network 100.0.0.0 0.0.0.3 area 0
 network 192.168.100.0 0.0.0.3 area 0
 network 192.168.105.0 0.0.0.3 area 0

! R2
router ospf 1
 network 200.0.0.0 0.0.0.3 area 0
 network 192.168.100.0 0.0.0.3 area 0
 network 172.16.23.0 0.0.0.3 area 0

! MSS
router ospf 1
 network 192.168.5.0 0.0.0.3 area 0
 network 192.168.105.0 0.0.0.3 area 0
 network 192.168.135.0 0.0.0.3 area 0
 network 192.168.145.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.255.255 area 0
 network 192.168.20.0 0.0.255.255 area 0
```

---

### 2. GRE Tunnel (R1 ↔ R2)

```
! R1
interface Tunnel10
 ip address 192.168.100.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 200.0.0.2

! R2
interface Tunnel10
 ip address 192.168.100.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 100.0.0.2
```

> The tunnel subnet `192.168.100.0/30` appears as directly connected (C) in R1's routing table, confirming the tunnel is operational.

---

### 3. VLANs & Inter-VLAN Routing

```
! VLAN creation (all switches)
vlan 10
 name Data
vlan 20
 name Voice
vlan 30
 name DNS-Server
vlan 40
 name Users-A
! repeat for remaining VLANs

! SVIs on multilayer switches
interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 no shutdown
interface vlan 20
 ip address 192.168.20.2 255.255.255.0
 no shutdown

! Enable Layer 3 routing
ip routing

! Access ports
interface range fa0/1 - 10
 switchport mode access
 switchport access vlan 10

! Trunk ports between switches
interface gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

---

### 4. EtherChannel — LACP & PAgP

```
! LACP bundle (active mode on both ends)
interface range fa0/1 - 2
 channel-group 1 mode active
 switchport trunk encapsulation dot1q
 switchport mode trunk

! PAgP bundle (desirable mode on both ends)
interface range fa0/3 - 4
 channel-group 2 mode desirable
 switchport trunk encapsulation dot1q
 switchport mode trunk

! Logical port-channel interface
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all
```

---

### 5. HSRP — First-Hop Redundancy

MS3 is Active for VLAN 10, MS2 is Active for VLAN 20 — this distributes load across both switches.

```
! MS3 — Active for VLAN 10
interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt

! MS2 — Standby for VLAN 10
interface vlan 10
 ip address 192.168.10.3 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 100

! MS2 — Active for VLAN 20
interface vlan 20
 ip address 192.168.20.3 255.255.255.0
 standby 20 ip 192.168.20.1
 standby 20 priority 110
 standby 20 preempt

! MS3 — Standby for VLAN 20
interface vlan 20
 ip address 192.168.20.2 255.255.255.0
 standby 20 ip 192.168.20.1
 standby 20 priority 100
```

---

### 6. DHCP — Centralized Server + Relay

```
! ip helper-address on SVIs (relay DHCP broadcasts to server)
interface vlan 10
 ip helper-address 192.168.5.2
interface vlan 20
 ip helper-address 192.168.5.2
```

| Pool | Network | Default Gateway | DNS Server |
|------|---------|----------------|-----------|
| VLAN10_POOL | `192.168.10.0/24` | `192.168.10.1` (HSRP VIP) | `172.16.30.x` |
| VLAN20_POOL | `192.168.20.0/24` | `192.168.20.1` (HSRP VIP) | `172.16.30.x` |

> The default gateway pushed by DHCP is the HSRP virtual IP, not the physical SVI address. Hosts keep their gateway even if the Active switch fails.

---

### 7. DNS Server (Site A — VLAN 30)

DNS server at `172.16.30.0/24` provides name resolution for both sites. It is reachable from Site B via OSPF routes propagated through the GRE tunnel.

---

## ✅ Verification Commands

```
! OSPF
show ip ospf neighbor
show ip route ospf

! GRE
show interface tunnel 10
ping 192.168.100.2 source tunnel 10

! VLANs
show vlan brief
show interfaces trunk

! EtherChannel
show etherchannel summary
show etherchannel port-channel

! HSRP
show standby brief
show standby vlan 10

! DHCP
show ip dhcp pool
show ip dhcp binding

! End-to-end reachability
ping 172.16.30.1 source 192.168.10.2
ping 192.168.10.1 source 172.16.30.1
```

---

## 📊 Routing Table Analysis (R1)

```
C    100.0.0.0/30        directly connected, GigabitEthernet0/0
C    192.168.100.0/30    directly connected, Tunnel10
C    192.168.105.0/30    directly connected, GigabitEthernet0/1

O    172.16.23.0/30  [110/3] via 100.0.0.1
O    172.16.30.0/24  [110/4] via 100.0.0.1    ! DNS Server VLAN
O    172.16.40.0/24  [110/4] via 100.0.0.1
O    172.16.50.0/24  [110/4] via 100.0.0.1
O    172.16.60.0/24  [110/4] via 100.0.0.1
O    172.16.70.0/24  [110/4] via 100.0.0.1
O    172.16.80.0/24  [110/4] via 100.0.0.1

O    192.168.5.0/30   [110/2] via 192.168.105.1    ! DHCP server link
O    192.168.10.0/24  [110/4] via 192.168.105.1    ! Site B VLAN 10
O    192.168.20.0/24  [110/4] via 192.168.105.1    ! Site B VLAN 20
O    192.168.113.0/30 [110/3] via 192.168.105.1
O    192.168.123.0/30 [110/3] via 192.168.105.1
O    192.168.124.0/30 [110/3] via 192.168.105.1
O    192.168.135.0/30 [110/2] via 192.168.105.1
O    192.168.145.0/30 [110/2] via 192.168.105.1
O    200.0.0.0/30     [110/2] via 100.0.0.1
```

All 16 remote subnets learned via OSPF — full convergence confirmed.

---

## 🔑 Key Concepts Applied

### OSPF Cost
OSPF cost = 10⁸ / bandwidth. The [110/X] notation shows AD=110 (OSPF default) and cumulative cost. Cost 2 = one intermediate hop, cost 3 = two hops, cost 4 = three hops.

### GRE Tunnel
GRE encapsulates Site A VLAN traffic inside IP packets routed over the ISP. OSPF runs inside the tunnel, making both sites appear as one OSPF domain without exposing internal subnets to the ISP.

### HSRP Load Balancing
Two HSRP groups with opposite Active/Standby assignments distribute Layer 3 traffic across MS2 and MS3. Both uplinks carry real traffic under normal conditions — not just during failover.

### DHCP + HSRP Integration
Hosts receive the HSRP virtual IP as default gateway from DHCP. When the Active switch fails, HSRP promotes the Standby and the gateway IP never changes from the host's perspective.

### EtherChannel vs STP
STP would block redundant physical links between switches to prevent loops. EtherChannel bundles them into one logical link — STP sees a single path and allows full bandwidth plus physical redundancy.

---

## 📁 Files

| File | Description |
|------|-------------|
| `lab 2.pkt` | Cisco Packet Tracer project file |
| `README.md` | This write-up |
| `lab 2.png` | Image of the project | 
---

## 🧠 Lessons Learned

- OSPF neighbor adjacency requires matching hello/dead timers, area IDs, and MTU — start troubleshooting with `show ip ospf neighbor`
- GRE tunnel endpoints must be reachable before the tunnel interface comes up — verify WAN connectivity first
- `ip helper-address` is mandatory when the DHCP server is on a different subnet — without it, broadcast DHCP requests never leave the local segment
- HSRP `priority` alone does not reclaim Active status after recovery — `preempt` must be configured explicitly
- EtherChannel mode must be compatible on both ends: active↔active or active↔passive for LACP; desirable↔desirable or desirable↔auto for PAgP; mismatched modes leave the bundle in err-disabled state

---

*Write-up by XENOS — part of the CCNA Projects Series*
