# Home Lab Network Engineering
**Curtis | CCNA Certified | March 2026**

Enterprise-grade home lab built to develop and demonstrate hands-on network engineering skills. Running production protocols across physical Cisco hardware, VMware ESXi virtualization, NAS storage, and full SNMP monitoring — documented end-to-end.

---

## 🏗️ Network Topology Overview

```
Internet (ISP)
     │
  Eero AP (WAN DHCP)
     │
  Cisco 1921 (Edge Router) — 192.168.168.168
  NAT/PAT | WAN ACL | NTP Master
     │
  Cisco WS-C3750X-48P (Core Switch) — 192.168.168.1
  Inter-VLAN Routing | OSPF DR | PoE | LACP
  ┌──────┬──────┬──────┬──────┐
  R1     R3     R4    SW01-04  Servers / NAS / Cameras
  │      │      │
  R2 (via serial)
```

**HSRP Virtual Gateway:** 192.168.168.254 (R1 Active / R3 Standby, <10s failover verified)

---

## 🔧 Hardware Inventory

| Category | Device | Qty | Status |
|---|---|---|---|
| Edge Router | Cisco 1921/K9 | 1 | Configured |
| Distribution Routers | Cisco 2811 (R1–R4) | 4 | Fully configured |
| Core Switch | Cisco WS-C3750X-48P | 1 | PoE, OSPF DR |
| Access Switches | Cisco Catalyst 3560-24TS | 4 | SSH, AAA, VLANs |
| Firewall | Cisco ASA 5516-X | 1 | On hold (FTD/license) |
| Servers | Dell PowerEdge R630 | 3 | ESXi, Windows, Linux (pending) |
| NAS | QNAP TS-832PXU-RP + TS-453BU | 2 | RAID 6, iSCSI live |
| GPU | PNY NVIDIA Quadro P2200 | 1 | Installed in S1 |
| UPS | CyberPower CP1500PFCLCD | 1 | PowerPanel Business |
| IP Cameras | Reolink PoE + LPR | 9 | VLAN 30 isolated |

---

## 📡 Routing Protocols

### OSPF — Area 0
Single-area OSPF across the core switch and all four routers.

| Device | Router-ID | Role |
|---|---|---|
| Core Switch | 168.168.168.1 | DR |
| R1 | 1.1.1.1 | BDR |
| R2 | 2.2.2.2 | P2P (serial only) |
| R3 | 3.3.3.3 | DROTHER |
| R4 | 4.4.4.4 | DROTHER |

### EIGRP — AS 90
Running across all four routers via serial and ethernet. R1 is hub. Ethernet paths preferred (cost 156,160) over serial (cost 20,640,000). R3/R4 perform equal-cost load balancing.

### eBGP — AS 1–4
Each router operates in its own autonomous system. R1 is the hub with 5 established BGP sessions.

| Router | AS | Neighbors | Sessions |
|---|---|---|---|
| R1 | AS 1 | R2, R3 (×2), R4 (×2) | 5 Established |
| R2 | AS 2 | R1 via serial | 1 Established |
| R3 | AS 3 | R1 via serial + ethernet | 2 Established |
| R4 | AS 4 | R1 via serial + ethernet | 2 Established |

### HSRP — First Hop Redundancy
- **Virtual IP:** 192.168.168.254
- **Active:** R1 (Priority 110, Preempt enabled)
- **Standby:** R3 (Priority 100, Preempt enabled)
- **Failover:** <10 seconds — tested and verified

---

## 🌐 VLAN Design

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 168 | Management | 192.168.168.0/24 | All device management |
| 20 | Servers | 10.20.0.0/24 | VM workloads, monitoring |
| 30 | Cameras | 10.30.0.0/24 | **Isolated** — cameras + NVR only |
| 40 | Storage | 10.40.0.0/24 | iSCSI and NAS traffic |

---

## 🖥️ Virtualization & Storage

**ESXi Host (S2):** Dell PowerEdge R630 — Dual Xeon E5-2643 v3 | 256GB RAM | ESXi 6.7 Update 3

**iSCSI Datastore:** 1TB thin-provisioned LUN from QNAP NAS2 (RAID 6) via CHAP-authenticated iSCSI — mounted as VMFS6 datastore (NAS2-iSCSI-DS01, 999.75GB)

**Virtual Machines:**

| VM | OS | IP | Purpose |
|---|---|---|---|
| zabbix-monitor | Ubuntu 24.04 LTS | 10.20.0.10 | Zabbix 7.0 monitoring platform |

---

## 📊 Monitoring — Zabbix 7.0

**14 hosts monitored | 2,295+ items | 1,263 triggers | All green**

SNMP polling configured on all Cisco IOS devices, QNAP NAS units, and ESXi hypervisor.

| Host | IP | Template | Items |
|---|---|---|---|
| Core Switch WS-C3750X | 192.168.168.1 | Cisco Catalyst by SNMP | 592 |
| Edge Router (1921) | 192.168.168.168 | Cisco IOS by SNMP | 82 |
| R1 / R2 / R3 / R4 | .2 / 10.1.12.2 / .3 / .4 | Cisco IOS by SNMP | 92–111 each |
| Access SW 01–04 | 192.168.168.51–54 | Cisco IOS by SNMP | 271–290 each |
| QNAP NAS1 + NAS2 | 192.168.168.40–41 | Linux by SNMP | 107–116 each |
| ESXi R630-2 | 192.168.168.16 | Linux by SNMP | 44 |

---

## 🔒 Security

- **Camera VLAN isolation:** Extended ACL restricts all inter-VLAN traffic to camera subnet — only management VLAN (192.168.168.0/24) can reach NVR
- **WAN ACL:** Inbound ACL on edge router blocking RFC 1918 spoofed traffic — **36,000+ packets blocked** in first days of operation
- **Device hardening:** AAA/TACACS, SSH v2, type 9 scrypt passwords, exec-timeout, NTP sync on all devices
- **NAT:** PAT overload for all internal VLANs + 2 static NAT entries (RDP to S1, NVR external access)

---

## 🧯 Troubleshooting Log

Real incidents encountered, diagnosed, and resolved during lab build:

| Incident | Root Cause | Resolution |
|---|---|---|
| ACL lockout — core switch | Incorrect ACL blocked all SSH | Console recovery, removed ACL |
| DHCP binding failure (NVR) | MAC reservation missing `01` prefix on client-identifier | Added `01ec.71db.333a.46` format |
| Camera VLAN — no internet | 3 simultaneous issues: trunk config, NAT ACL, missing return route | Fixed all three independently |
| OSPF no adjacency (R1↔SW) | `passive-interface` set on R1 Fa0/0 | Removed passive-interface |
| HSRP failover — SSH lost | Needed console access during active router change | Documented: always attach console for HSRP testing |
| BGP stale neighbor (AS 16) | Left by previous device owner | Removed with `no neighbor 16.16.16.16` |
| iSCSI — no space for LUN | NAS2 DataVol1 was thick provisioned (8.80GB free) | Converted to thin — freed 4.60TB |
| QNAP SNMP blocked | QuFirewall re-enabled, missing 10.20.0.0/24 allow rule | Added monitoring subnet to QuFirewall |
| Zabbix ICMP ping failure | Missing return route for 10.20.0.0/24 on routers | Added `ip route 10.20.0.0 255.255.255.0 192.168.168.1` |
| iSCSI CHAP bash error | `!` in password interpreted as history expansion | Removed special character from CHAP password |

---

## 📁 Repository Contents

```
├── configs/
│   ├── edge-router.txt
│   ├── core-switch.txt
│   ├── r1.txt — r4.txt
│   └── access-sw-01.txt — access-sw-04.txt
├── diagrams/
│   ├── network-topology.png
│   └── vlan-design.png
├── zabbix/
│   └── monitoring-setup.md
└── docs/
    └── addressing-reference.md
```

---

## 🎯 Certifications & Background

- **CCNA** — Cisco Certified Network Associate (active)
- **CCNP Enterprise** — In progress
- 16 years Law Enforcement (Deputy Sheriff / DC Supervisor) — security clearance eligible

---

## 🚧 In Progress

- Ansible automation on Linux VM (S3 — RAM sourcing)
- ASA 5516-X integration (SmartNet / Eve-NG)
- Blue Iris + LPR camera setup
- Windows Server AD/DNS/DHCP on S1
- GitHub: push all device configs

