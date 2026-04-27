# CCNA Lab 1 — Static Routing & Subnetting Design

> **Author:** XENOS
> **Tool:** Cisco Packet Tracer
> **Topics:** IPv4 Subnetting · Static Routing · WAN Links · ISP Simulation
> **Difficulty:** Intermediate

---

## 📋 Lab Overview

This lab simulates a real-world multi-area network involving an ISP zone, WAN point-to-point links, and three internal LAN segments. The core objectives are:

- Design an IP addressing scheme that satisfies specific host requirements per subnet
- Configure static routes across multiple routers to achieve full end-to-end connectivity
- Simulate ISP connectivity using two separate WAN uplinks

---

## 🗺️ Network Topology

The topology is divided into eight logical zones:

| Zone | Subnet | Purpose |
|------|--------|---------|
| ISP / External | `195.105.195.0/29` | DNS server, CCNA server, ISP routers |
| WAN Link 1 | `100.0.0.0/30` | ISP1 ↔ Edge Router |
| WAN Link 2 | `200.0.0.0/30` | ISP2 ↔ Edge Router |
| Core Link A | `172.16.1.8/30` | Router-to-Router |
| Core Link B | `172.16.1.4/30` | Router-to-Router |
| Core Link C | `172.16.1.0/30` | Router-to-Router |
| LAN A | `172.16.0.192/26` | 50 hosts |
| LAN B | `172.16.0.128/26` | 62 hosts |
| LAN C | `172.16.0.0/25` | 100 hosts |

---

## 📐 Subnetting Design

The internal address space `172.16.0.0/16` was subnetted to fit exact host requirements.

### LAN Subnets

| Subnet | CIDR | Subnet Mask | Network | Broadcast | Usable Range | Hosts | Required |
|--------|------|-------------|---------|-----------|--------------|-------|----------|
| LAN C | `/25` | `255.255.255.128` | `172.16.0.0` | `172.16.0.127` | `.1 – .126` | **126** | 100 ✅ |
| LAN B | `/26` | `255.255.255.192` | `172.16.0.128` | `172.16.0.191` | `.129 – .190` | **62** | 62 ✅ |
| LAN A | `/26` | `255.255.255.192` | `172.16.0.192` | `172.16.0.255` | `.193 – .254` | **62** | 50 ✅ |

> **Why /26 for 50 hosts?**
> The next smaller mask /27 gives only 30 usable hosts — insufficient.
> /26 gives 62 usable hosts, satisfying the requirement with headroom.

### Point-to-Point Links (Router-to-Router)

| Link | CIDR | Network | Usable IPs | Broadcast |
|------|------|---------|------------|-----------|
| Core A | `172.16.1.8/30` | `172.16.1.8` | `.9 – .10` | `172.16.1.11` |
| Core B | `172.16.1.4/30` | `172.16.1.4` | `.5 – .6` | `172.16.1.7` |
| Core C | `172.16.1.0/30` | `172.16.1.0` | `.1 – .2` | `172.16.1.3` |

### WAN & ISP Subnets

| Link | CIDR | Network | Usable IPs | Broadcast |
|------|------|---------|------------|-----------|
| WAN 1 (ISP1) | `100.0.0.0/30` | `100.0.0.0` | `.1 – .2` | `100.0.0.3` |
| WAN 2 (ISP2) | `200.0.0.0/30` | `200.0.0.0` | `.1 – .2` | `200.0.0.3` |
| ISP Zone | `195.105.195.0/29` | `195.105.195.0` | `.1 – .6` | `195.105.195.7` |

> **/29 for the ISP zone** → 6 usable addresses covering ISP1, ISP2, DNS server, and CCNA server.

---

## ⚙️ Static Routing Configuration

### Edge Router

```
! Primary and backup default routes (floating static)
ip route 0.0.0.0 0.0.0.0 100.0.0.1         ! ISP1 — primary (AD 1)
ip route 0.0.0.0 0.0.0.0 200.0.0.1 10       ! ISP2 — backup  (AD 10)

! Internal LAN reachability
ip route 172.16.0.0   255.255.255.128 172.16.1.9   ! LAN C — 100 hosts
ip route 172.16.0.128 255.255.255.192 172.16.1.9   ! LAN B — 62 hosts
ip route 172.16.0.192 255.255.255.192 172.16.1.5   ! LAN A — 50 hosts
```

### Core Router A (LAN C — 100 hosts)

```
ip route 0.0.0.0 0.0.0.0 172.16.1.10
ip route 172.16.0.128 255.255.255.192 172.16.1.10
ip route 172.16.0.192 255.255.255.192 172.16.1.10
ip route 195.105.195.0 255.255.255.248 172.16.1.10
```

### Core Router B (LAN B — 62 hosts)

```
ip route 0.0.0.0 0.0.0.0 172.16.1.6
ip route 172.16.0.0   255.255.255.128 172.16.1.6
ip route 172.16.0.192 255.255.255.192 172.16.1.6
ip route 195.105.195.0 255.255.255.248 172.16.1.6
```

### Core Router C (LAN A — 50 hosts)

```
ip route 0.0.0.0 0.0.0.0 172.16.1.2
ip route 172.16.0.0   255.255.255.128 172.16.1.2
ip route 172.16.0.128 255.255.255.192 172.16.1.2
ip route 195.105.195.0 255.255.255.248 172.16.1.2
```

### ISP Routers

```
! ISP1
ip route 172.16.0.0 255.255.0.0 100.0.0.2

! ISP2
ip route 172.16.0.0 255.255.0.0 200.0.0.2
```

---

## ✅ Verification Commands

```
show ip route
show ip route static
show ip interface brief

ping 172.16.0.1          ! LAN C gateway
ping 172.16.0.129        ! LAN B gateway
ping 172.16.0.193        ! LAN A gateway
ping 195.105.195.1       ! DNS server

traceroute 172.16.0.1
```

Expected `show ip route` on Edge Router:

```
S    172.16.0.0/25    [1/0] via 172.16.1.9
S    172.16.0.128/26  [1/0] via 172.16.1.9
S    172.16.0.192/26  [1/0] via 172.16.1.5
S*   0.0.0.0/0        [1/0] via 100.0.0.1
```

---

## 🔑 Key Concepts Applied

### 1. VLSM (Variable Length Subnet Masking)
Rather than assigning the same subnet size to every segment, VLSM was used to allocate addresses efficiently:
- Larger subnets (/25, /26) for host-dense LANs
- Minimal /30 subnets for point-to-point router links
- /29 for the ISP zone with multiple devices

### 2. Floating Static Route
ISP2 was configured with an Administrative Distance of `10` (vs. ISP1's default `1`), making it a **floating static route** — it only enters the routing table if the ISP1 path fails. This implements basic WAN redundancy without any dynamic routing protocol.

### 3. Summarization Awareness
All three LAN subnets (`172.16.0.0/25`, `/26`, `/26`) fall within `172.16.0.0/24`. ISP-facing static routes can optionally be collapsed into a single summary:

```
ip route 172.16.0.0 255.255.255.0 <next-hop>
```

This reduces routing table size on ISP routers.

---

## 📁 Files

| File | Description |
|------|-------------|
| `lab 1.pkt` | Cisco Packet Tracer project file |
| `README.md` | This write-up |
| `lab 1.png` | Image of the project |
ك
ط
---

## 🧠 Lessons Learned

- Always subnet starting from the **largest host requirement** (100 → 62 → 50) to avoid address waste
- `/30` is the correct mask for any router-to-router link — /29 wastes 4 addresses per link
- Floating static routes provide basic failover with zero protocol overhead
- Always run `show ip route` on every single router before declaring the lab complete

---

*Write-up by XENOS — part of the CCNA Projects Series*
