# CCNA Lab 3 — Pure IPv6 Network: RIPng + OSPFv3 + EIGRPv6 with Redistribution

> **Author:** XENOS
> **Tool:** Cisco Packet Tracer
> **Score:** 99 / 100
> **Topics:** IPv6 Addressing · RIPng · OSPFv3 · EIGRPv6 · Route Redistribution · IPv6 Tunnel
> **Difficulty:** Advanced

---

## 📋 Lab Overview

This lab is a full IPv6-only network spanning three routing domains connected through a central ISP. Each domain runs a different IPv6 routing protocol. Route redistribution is configured at the domain boundaries to achieve end-to-end reachability across all six subnets.

| Domain | Protocol | Routers |
|--------|---------|---------|
| Left Site | RIPng | R3, R4 |
| Core / WAN | OSPFv3 | R1, R2, ISP |
| Right Site | EIGRPv6 | R5, R6 |

**Key design challenge:** three different routing protocols must exchange routes at two redistribution points (R3↔R1 and R2↔R5), and all hosts across all zones must reach each other using IPv6 only.

---

## 🗺️ Network Topology

### Subnets and Zones

| Prefix | Zone | Hosts |
|--------|------|-------|
| `FC00:2::/64` | Blue — Left | PC4, PC5 |
| `FC00:3::/64` | Yellow — Left | Server0 |
| `FC00:1::/64` | Green — Left | PC2, PC3 (Switch0) |
| `FC00:4::/64` | Orange — Right | PC0, PC1 (Switch2) |
| `FC00:5::/64` | Blue — Right | Server1 |

### Point-to-Point Links

| Link | Prefix | Routers |
|------|--------|---------|
| R4 ↔ R3 | `FD00:2::/64` | RIPng domain internal |
| R3 ↔ R1 | `FD00:1::/64` | RIPng ↔ OSPFv3 boundary |
| R1 ↔ R2 | `FC00:100::/64` | OSPFv3 core (IPv6 tunnel) |
| R2 ↔ R5 | `FD00:3::/64` | OSPFv3 ↔ EIGRPv6 boundary |
| R5 ↔ R6 | `FD00:4::/64` | EIGRPv6 domain internal |

---

## ⚙️ Configuration

### 1. IPv6 Addressing (all routers)

IPv6 unicast routing must be enabled before any IPv6 routing protocol works.

```
! Enable IPv6 routing globally (every router)
ipv6 unicast-routing

! Interface addressing example (R4)
interface GigabitEthernet0/0
 ipv6 address FD00:2::1/64
 ipv6 enable
 no shutdown

interface GigabitEthernet0/1
 ipv6 address FC00:2::1/64
 ipv6 enable
 no shutdown

interface GigabitEthernet0/2
 ipv6 address FC00:3::1/64
 ipv6 enable
 no shutdown
```

---

### 2. RIPng — Left Domain (R3, R4)

RIPng is the IPv6 version of RIP. It uses link-local addresses as next-hops and multicast group FF02::9 for updates. It is enabled per interface, not per network statement.

```
! R4
ipv6 router rip RIPNG_DOMAIN

interface GigabitEthernet0/0
 ipv6 rip RIPNG_DOMAIN enable

interface GigabitEthernet0/1
 ipv6 rip RIPNG_DOMAIN enable

interface GigabitEthernet0/2
 ipv6 rip RIPNG_DOMAIN enable

! R3 (same process — enable on all interfaces in RIPng domain)
ipv6 router rip RIPNG_DOMAIN

interface GigabitEthernet0/0
 ipv6 rip RIPNG_DOMAIN enable

interface GigabitEthernet0/1
 ipv6 rip RIPNG_DOMAIN enable
```

> RIPng uses a hop count metric (max 15 hops). The routing table on R4 shows [120/2] for remote subnets — AD 120, metric 2 hops.

---

### 3. OSPFv3 — Core Domain (R1, R2, ISP)

OSPFv3 is the IPv6 version of OSPF. It runs over link-local addresses and is configured per interface. Process ID and router ID are still required.

```
! R1
ipv6 router ospf 1
 router-id 1.1.1.1

interface GigabitEthernet0/0
 ipv6 ospf 1 area 0

interface GigabitEthernet0/1
 ipv6 ospf 1 area 0

! R2
ipv6 router ospf 1
 router-id 2.2.2.2

interface GigabitEthernet0/0
 ipv6 ospf 1 area 0

interface GigabitEthernet0/1
 ipv6 ospf 1 area 0

! ISP
ipv6 router ospf 1
 router-id 3.3.3.3

interface GigabitEthernet0/0
 ipv6 ospf 1 area 0

interface GigabitEthernet0/1
 ipv6 ospf 1 area 0
```

The red segment in the topology between R1 and R2 (`FC00:100::/64`) is an IPv6 tunnel — OSPFv3 runs through it to form neighbor adjacency between the two core routers.

---

### 4. EIGRPv6 — Right Domain (R5, R6)

EIGRPv6 uses the same DUAL algorithm as EIGRP but operates over IPv6. It also requires a router ID (32-bit value in dotted decimal format) even though the network is pure IPv6.

```
! R5
ipv6 router eigrp 100
 router-id 5.5.5.5
 no shutdown

interface GigabitEthernet0/0
 ipv6 eigrp 100

interface GigabitEthernet0/1
 ipv6 eigrp 100

! R6
ipv6 router eigrp 100
 router-id 6.6.6.6
 no shutdown

interface GigabitEthernet0/0
 ipv6 eigrp 100

interface GigabitEthernet0/1
 ipv6 eigrp 100

interface GigabitEthernet0/2
 ipv6 eigrp 100
```

---

### 5. Route Redistribution

Redistribution is required at two boundary routers to allow routes from one protocol to enter another.

#### Boundary 1 — R3: RIPng ↔ OSPFv3

```
! R3 redistributes OSPFv3 routes into RIPng
ipv6 router rip RIPNG_DOMAIN
 redistribute ospf 1 metric 5

! R3 redistributes RIPng routes into OSPFv3
ipv6 router ospf 1
 router-id 3.3.3.3
 redistribute rip RIPNG_DOMAIN include-connected
```

#### Boundary 2 — R2: OSPFv3 ↔ EIGRPv6

```
! R2 redistributes EIGRPv6 routes into OSPFv3
ipv6 router ospf 1
 redistribute eigrp 100 include-connected

! R2 redistributes OSPFv3 routes into EIGRPv6
ipv6 router eigrp 100
 router-id 2.2.2.2
 redistribute ospf 1 metric 10000 100 255 1 1500
```

> Redistribution metric for EIGRP must include all five K-values: bandwidth (kbps), delay (tens of microseconds), reliability, load, MTU.

---

## 📊 Routing Table Analysis (R4 — `show ipv6 route`)

```
C    FC00:2::/64   [0/0]   via Gi0/1, directly connected     ! Blue zone
C    FC00:3::/64   [0/0]   via Gi0/2, directly connected     ! Yellow zone (Server0)
C    FD00:2::/64   [0/0]   via Gi0/0, directly connected     ! Link to R3

R    FC00:1::/64   [120/2] via FE80::..., Gi0/0              ! Green zone — learned RIPng
R    FC00:4::/64   [120/2] via FE80::..., Gi0/0              ! Right orange — via redistribution
R    FC00:5::/64   [120/2] via FE80::..., Gi0/0              ! Right server — via redistribution
R    FC00:100::/64 [120/2] via FE80::..., Gi0/0              ! Core tunnel subnet
R    FD00:3::/64   [120/2] via FE80::..., Gi0/0              ! R2-R5 link
R    FD00:4::/64   [120/2] via FE80::..., Gi0/0              ! R5-R6 link
```

All remote subnets including those from OSPFv3 and EIGRPv6 domains appear in R4's table with RIPng metric [120/2] — they entered R4's domain through redistribution at R3, then propagated one more hop to R4, giving metric 2.

---

## ✅ Verification Commands

```
! IPv6 routing table
show ipv6 route
show ipv6 route rip
show ipv6 route ospf
show ipv6 route eigrp

! Protocol neighbors
show ipv6 rip database
show ipv6 ospf neighbor
show ipv6 eigrp neighbors

! Interface IPv6 status
show ipv6 interface brief

! End-to-end reachability
ping FC00:5::2 source FC00:2::2     ! Left PC → Right Server
ping FC00:3::2 source FC00:4::2     ! Right PC → Left Server
ping FC00:1::2 source FC00:5::2     ! Green zone → Right Server

! Trace the path
traceroute FC00:5::2 source FC00:2::2
```

---

## 🔑 Key Concepts Applied

### RIPng vs OSPFv3 vs EIGRPv6

| Feature | RIPng | OSPFv3 | EIGRPv6 |
|---------|-------|--------|---------|
| Type | Distance Vector | Link State | Advanced Distance Vector |
| Metric | Hop count (max 15) | Cost (bandwidth-based) | Composite (BW + delay + ...) |
| AD | 120 | 110 | 90 (internal) |
| Updates | Periodic (30s) | Event-triggered (LSAs) | Event-triggered (partial) |
| Convergence | Slow | Fast | Very fast |
| Next-hop | Link-local address | Link-local address | Link-local address |
| Config style | Per-interface | Per-interface | Per-interface |

### Link-Local Addresses as Next-Hops
All three IPv6 routing protocols use link-local addresses (FE80::/10) as next-hops in the routing table. This is why R4's table shows `via FE80::201:63FF:FE05:5E02` rather than a global prefix — link-local addresses are never routed, only used on the local segment.

### Router ID in Pure IPv6 Networks
OSPFv3 and EIGRPv6 still require a 32-bit router ID in dotted decimal format (e.g., 1.1.1.1) even in a pure IPv6 environment. If no IPv4 addresses exist on the router, the router ID must be configured manually — otherwise the protocol process fails to start.

### Redistribution Metric Requirement
When redistributing into EIGRP, all five metric components must be specified (bandwidth, delay, reliability, load, MTU). Omitting any value causes the redistributed routes to be rejected by EIGRP neighbors.

### `include-connected` in OSPFv3 Redistribution
When redistributing into OSPFv3, the `include-connected` keyword is needed to also advertise the directly connected interfaces of the redistributing router. Without it, the boundary router's own subnets would be invisible to the OSPFv3 domain.

---

## 📁 Files

| File | Description |
|------|-------------|
| `lab 3.pkt` | Cisco Packet Tracer project file |
| `README.md` | This write-up |
| `lab 3.png` | |

---

## 🧠 Lessons Learned

- `ipv6 unicast-routing` must be the first command on every router in a pure IPv6 lab — without it, no IPv6 routing protocol will function
- EIGRPv6 requires `no shutdown` under the process — unlike EIGRP for IPv4, the process starts in a shutdown state by default
- OSPFv3 router ID is a 32-bit value — always configure it manually in pure IPv6 networks to avoid startup failures
- RIPng redistribution into OSPFv3 creates external type 2 (E2) routes — their metric does not increase as they traverse the OSPFv3 domain
- Two-way redistribution (A into B and B into A) can cause routing loops if not controlled — prefix lists or route maps should be used in production to limit what gets redistributed in each direction

---

*Write-up by XENOS — part of the CCNA Projects Series*
