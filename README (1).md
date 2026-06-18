# 🏥 City Smart Hospital Network Design

A fully simulated enterprise-grade hospital network built in **Cisco Packet Tracer**, covering VLANs, inter-VLAN routing, OSPF, DHCP, NAT/PAT, DNS, redundancy/failover, and wireless access.

---

## 📁 Repository Contents

```
.
├── project.pkt                  # Cisco Packet Tracer simulation file
└── Networks_Project.pdf         # Full project report (PDF)
```

> **Note:** The `.pkt` file requires **Cisco Packet Tracer** to open. It was saved using Packet Tracer 8.x. Opening it in a different version may cause compatibility issues.

---

## 🗺️ Network Topology Overview

The hospital campus is divided into **4 internal zones** served by dedicated routers, all connected to a central data core, plus an external ISP link.

```
                        [ISP_R] ── Internet (8.8.8.8)
                           |
                      [Core_Data_R]  ← Zone 4: Data Center (VLAN 80)
                     /      |      \
               [Admin_R] [Medical_R] [Outpatient_R]
                  |     \      |          |
              Zone 1   Backup Zone 2    Zone 3
                       Serial
                        Link
```

### Router Interconnections

| Link | Interface Type | Purpose |
|------|---------------|---------|
| Admin_R ↔ Core_Data_R | GigabitEthernet | Primary |
| Medical_R ↔ Core_Data_R | GigabitEthernet | Primary |
| Outpatient_R ↔ Core_Data_R | GigabitEthernet | Primary |
| Medical_R ↔ Admin_R | Serial DCE | **Backup / Redundancy** |
| Core_Data_R ↔ ISP_R | Serial/GigEth | Internet uplink |
| Admin_R ↔ Radiology_R | Serial DCE | Remote Clinic (Phase 2) |

---

## 🌐 IP Addressing Scheme

All internal subnets use **FLSM /24** from the `172.16.0.0/16` private block. The third octet matches the VLAN ID for readability.

### VLAN Subnets

| VLAN | Department | Network | Gateway |
|------|-----------|---------|---------|
| 10 | Patient Records | 172.16.10.0/24 | 172.16.10.1 |
| 20 | Finance | 172.16.20.0/24 | 172.16.20.1 |
| 30 | HR | 172.16.30.0/24 | 172.16.30.1 |
| 40 | General Ward | 172.16.40.0/24 | 172.16.40.1 |
| 50 | ICU | 172.16.50.0/24 | 172.16.50.1 |
| 60 | Consultation Rooms | 172.16.60.0/24 | 172.16.60.1 |
| 70 | Guest Wi-Fi | 172.16.70.0/24 | 172.16.70.1 |
| 80 | Data Center Servers | 172.16.80.0/24 | 172.16.80.1 |

### Point-to-Point Links

| Connection | Subnet |
|-----------|--------|
| Admin_R – Core_Data_R | 172.16.100.0/24 |
| Medical_R – Core_Data_R | 172.16.101.0/24 |
| Outpatient_R – Core_Data_R | 172.16.102.0/24 |
| Medical_R – Admin_R (backup) | 172.16.103.0/24 |
| Radiology_R – Admin_R | 172.16.104.0/24 |
| Core_Data_R – ISP_R | 203.0.113.0/30 (public) |

### Server Static Addresses

| Server | IP Address |
|--------|-----------|
| DNS Server | 172.16.80.10 |
| Internal Web Server (www.cityhospital.local) | 172.16.80.20 |
| External Web Server (www.google.com sim.) | 8.8.8.8 |

---

## ⚙️ Features Implemented

### Switching & VLANs
- All intra-zone switches configured with named VLANs
- Access ports assigned per department
- Trunk ports (802.1Q) between switches and routers

### Inter-VLAN Routing — Router-on-a-Stick
Each zone router uses sub-interfaces with `encapsulation dot1Q <vlan-id>` to route between its VLANs over a single physical trunk link.

### DHCP
Each router serves as the DHCP server for its attached VLANs. The router sub-interface IP is excluded from each pool. All pools point to the central DNS server at `172.16.80.10`.

### OSPF (Single-Area, Area 0)
All internal routers participate in OSPF Area 0. A default static route on `Core_Data_R` pointing to `ISP_R` is injected into OSPF using `default-information originate`.

### NAT Overload (PAT)
Configured on `Core_Data_R` to translate all internal `172.16.0.0/16` traffic to the public interface IP (`203.0.113.x`) for Internet access. OSPF is **not** run on the ISP-facing link.

### DNS & Web Services
- DNS server at `172.16.80.10` resolves `www.cityhospital.local` → `172.16.80.20`
- Internal web server hosts the hospital intranet page
- External web server at `8.8.8.8` simulates `www.google.com`

### Wireless (Zone 3 — Outpatient)
- PT-AP Access Point connected to an access port on the Outpatient switch
- Guest Wi-Fi devices isolated in **VLAN 70**, preventing access to internal medical systems

---

## 🔁 Phase 2: Advanced Scenarios

### Part A — Remote Radiology Clinic Expansion
A new `Radiology_R` router was added, connected via Serial DCE to `Admin_R`, using subnet `172.16.104.0/24`. DHCP and OSPF were configured so Radiology PCs can reach `www.cityhospital.local`.

### Part B — Disaster & Failover Test
The primary GigabitEthernet link between `Medical_R` and `Core_Data_R` was physically deleted to simulate a fiber cut. OSPF reconverged and rerouted ICU traffic through the backup serial path:

```
Medical_R → Admin_R → Core_Data_R
```

Traceroute output confirmed the path change. OSPF dead timers can be accelerated using the **Fast Forward Time** button in Packet Tracer.

### Part C — Traffic Analysis (Simulation Mode)
Packet Tracer Simulation Mode was used to capture the full request for `www.cityhospital.local`:

| Step | Protocol | Transport | Port |
|------|----------|-----------|------|
| 1 | DNS Query | **UDP** | 53 |
| 2 | DNS Response | **UDP** | 53 |
| 3 | TCP 3-Way Handshake | TCP | — |
| 4 | HTTP Request | **TCP** | 80 |
| 5 | HTTP Response | **TCP** | 80 |

DNS uses **UDP** (lightweight, connectionless, low overhead for small query/response pairs). HTTP uses **TCP** (requires reliable, ordered delivery for web page content).

---

## ✅ Verification Summary

| Test | Result |
|------|--------|
| DHCP address assignment | ✅ All VLANs receive addresses |
| Ping — internal hosts | ✅ Cross-VLAN and cross-zone |
| Ping — Radiology_2 → Web Server (172.16.80.20) | ✅ 0% packet loss |
| DNS resolution (www.cityhospital.local) | ✅ Resolves correctly |
| HTTP — browser access to intranet | ✅ Page loads from ICU PC |
| NAT translations | ✅ PAT entries visible in translation table |
| OSPF neighbor adjacencies | ✅ All in FULL state |
| Failover (cable cut) | ✅ OSPF reconverged via backup link |
| Traceroute before failure | 4 hops via Core_Data_R direct |
| Traceroute after failure | Rerouted: Medical_R → Admin_R → Core_Data_R |

---

## 🛠️ How to Open the Simulation

1. Install [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (version 8.x recommended).
2. Open `project.pkt` from within Packet Tracer (`File → Open`).
3. To test failover (Part B), delete the GigEth cable between `Medical_R` and `Core_Data_R`, then click **Fast Forward Time** several times to trigger OSPF convergence.
4. To run traffic analysis (Part C), switch to **Simulation Mode** and send an HTTP request from any edge PC to `www.cityhospital.local`.

---

## 📚 Key Concepts Demonstrated

- **FLSM vs VLSM** — trade-offs between simplicity and address efficiency
- **Router-on-a-Stick** — single-interface inter-VLAN routing
- **OSPF** — dynamic routing with automatic failover
- **NAT/PAT** — many-to-one address translation for Internet access
- **Redundancy design** — backup serial links in a critical-infrastructure network
- **Protocol analysis** — TCP vs UDP behavior at the transport layer




## 👥 Team Members

| Name | Student ID | Contribution |
|------|-----------|-------------|
| Mai Ibrahim | 202200504 | Topology design & physical connections |
| Hassan Ahmed | 202202121 | IP addressing, VLANs, trunking, Router-on-a-Stick |
| Aya Gaber | 202200288 | DHCP, DNS server, Web server, internal web access |
| Israa Atta | 202001430 | OSPF routing & Remote Radiology Clinic integration |
| Ahmed Gamal | 202200133 | NAT overload, ISP connectivity, failover testing, report |
