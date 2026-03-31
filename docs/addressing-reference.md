# Network Addressing Reference

Home Lab — Curtis | CCNA | March 2026

---

## VLAN Design

| VLAN | Name | Subnet | Gateway SVI | Purpose |
|---|---|---|---|---|
| 168 | Management | 192.168.168.0/24 | 192.168.168.1 | All device management traffic |
| 20 | Servers | 10.20.0.0/24 | 10.20.0.1 | VM workloads, Zabbix monitoring |
| 30 | Cameras | 10.30.0.0/24 | 10.30.0.1 | **ISOLATED** — cameras and NVR only |
| 40 | Storage | 10.40.0.0/24 | 10.40.0.1 | iSCSI and NAS storage traffic |

---

## Device IP Reference

### Network Infrastructure

| Device | IP Address | VLAN | Notes |
|---|---|---|---|
| Core Switch SVI | 192.168.168.1 | 168 | Inter-VLAN routing, OSPF DR |
| Edge Router LAN (Gi0/1) | 192.168.168.168 | 168 | NTP master for all devices |
| Edge Router WAN (Gi0/0) | 192.168.4.127 | N/A | DHCP from Eero |
| HSRP Virtual Gateway | 192.168.168.254 | 168 | Active: R1 — Standby: R3 |
| R1 (Fa0/0) | 192.168.168.2 | 168 | HSRP Active, priority 110, BGP AS 1 |
| R2 (Serial0/0/0) | 10.1.12.2 | N/A | BGP AS 2, reachable via R1 |
| R3 (Fa0/0) | 192.168.168.3 | 168 | HSRP Standby, priority 100, BGP AS 3 |
| R4 (Fa0/0) | 192.168.168.4 | 168 | BGP AS 4 |

### Access Switches

| Device | IP Address | VLAN | Uplink Port (Core) |
|---|---|---|---|
| access-sw-01 | 192.168.168.51 | 168 | Gi1/0/21 |
| access-sw-02 | 192.168.168.52 | 168 | Gi1/0/22 |
| access-sw-03 | 192.168.168.53 | 168 | Gi1/0/23 |
| access-sw-04 | 192.168.168.54 | 168 | Gi1/0/25 |

### Servers

| Device | IP Address | VLAN | Notes |
|---|---|---|---|
| Dell R630 S1 | 192.168.168.100 | 168 | Windows Server 2019, Quadro P2200 |
| iDRAC S1 | 192.168.168.10 | 168 | Out-of-band management |
| ESXi S2 | 192.168.168.16 | 168 | ESXi 6.7 host management |
| iDRAC S2 | 192.168.168.11 | 168 | Out-of-band management |

### Storage

| Device | IP Address | VLAN | Notes |
|---|---|---|---|
| QNAP NAS1 (TS-453BU) | 192.168.168.40 | 168 | 4× 1TB SSD, RAID rebuild pending |
| QNAP NAS2 (TS-832PXU-RP) | 192.168.168.41 | 168 | 8× 1TB SSD, RAID 6, iSCSI live |

### Virtual Machines

| Device | IP Address | VLAN | Notes |
|---|---|---|---|
| Zabbix VM (Ubuntu 24.04) | 10.20.0.10 | 20 | Zabbix 7.0 LTS monitoring |

### Cameras and NVR (VLAN 30 — Isolated)

| Device | IP Address | VLAN | Notes |
|---|---|---|---|
| Reolink NVR | 10.30.0.20 | 30 | DHCP reservation |
| IP Camera 1 | 10.30.0.13 | 30 | Reolink PoE |
| IP Camera 2 | 10.30.0.14 | 30 | Reolink PoE |
| IP Camera 3 | 10.30.0.15 | 30 | Reolink PoE |
| IP Camera 4 | 10.30.0.16 | 30 | Reolink PoE |
| IP Camera 5 | 10.30.0.17 | 30 | Reolink PoE |
| IP Camera 6 | 10.30.0.18 | 30 | Reolink PoE |
| IP Camera 7 | 10.30.0.19 | 30 | Reolink PoE |
| LPR Camera (EmpireTech) | Pending | 30 | NW corner — pending install |
| LPR Camera (Reolink RLC-823A) | Pending | 30 | NE corner — installed |

---

## Serial Links (Point-to-Point)

| Link | R1 IP | Remote IP | Notes |
|---|---|---|---|
| R1 → R2 | 10.1.12.1/30 | 10.1.12.2/30 | R1 = DCE, clock rate 125000 |
| R1 → R3 | 10.1.13.1/30 | 10.1.13.2/30 | Point-to-point |
| R1 → R4 | 10.1.14.1/30 | 10.1.14.2/30 | Point-to-point |

---

## NAT Translations

| Type | External IP | Internal IP | Purpose |
|---|---|---|---|
| PAT Overload | WAN IP | 192.168.168.0/24 + all VLANs | Internet for all internal devices |
| Static NAT | 192.168.4.128 | 192.168.168.100 | RDP to Windows Server S1 |
| Static NAT | 192.168.4.150 | 10.30.0.20 | External NVR camera access |

---

## Core Switch Port Map

| Port(s) | Description | Config |
|---|---|---|
| Gi1/0/1 | UPLINK-TO-EDGE-RTR | Trunk |
| Gi1/0/4–11 | IP Cameras (VLAN 30) | Access VLAN 30, PoE |
| Gi1/0/12 | CAM-LPR-EmpireTech (pending) | Access VLAN 30 |
| Gi1/0/13 | QNAP-NAS1 | Trunk |
| Gi1/0/21,22,23,25 | Access switch uplinks SW01–04 | Trunk |
| Gi1/0/27 | R4-UPLINK | Trunk |
| Gi1/0/28 | SVR2-ESXi2-DS-ServerNet | Trunk |
| Gi1/0/35 | iDRAC-S1 (192.168.168.10) | Access VLAN 168 |
| Gi1/0/36 | iDRAC-S2 (192.168.168.11) | Access VLAN 168 |
| Gi1/0/37–38 | QNAP-NAS2 LACP bond (Po2, 2Gbps) | Trunk |
| Gi1/0/39 | R1-UPLINK | Trunk |
| Gi1/0/40 | R3-UPLINK | Trunk |
| Gi1/0/41–48 | Server NICs (ESXi vSwitch0) | Trunk |

---

## SSH / Management Access

| Device | Address | Username | Method |
|---|---|---|---|
| Edge Router | 192.168.168.168 | curtis | SSH |
| Core Switch | 192.168.168.1 | curtis | SSH |
| R1 | 192.168.168.2 | curtis | SSH |
| R2 | 10.1.12.2 | curtis | SSH via R1 hop |
| R3 | 192.168.168.3 | curtis | SSH |
| R4 | 192.168.168.4 | curtis | SSH |
| access-sw-01 | 192.168.168.51 | curtis | SSH |
| access-sw-02 | 192.168.168.52 | curtis | SSH |
| access-sw-03 | 192.168.168.53 | curtis | SSH |
| access-sw-04 | 192.168.168.54 | curtis | SSH |
| Zabbix VM | 10.20.0.10 | curtis | SSH |
| ESXi S2 | 192.168.168.16 | root | SSH + HTTPS web UI |
| QNAP NAS1 | 192.168.168.40:8080 | admin | HTTPS web UI |
| QNAP NAS2 | 192.168.168.41:8080 | admin | HTTPS web UI |
| Windows Server S1 | 192.168.168.100 | Administrator | RDP |
| Zabbix Web UI | http://10.20.0.10/zabbix | Admin | Web browser |

---

*Last updated: March 2026*
