# Troubleshooting Incident Log

Home Lab — Curtis | CCNA | March 2026

This log documents real incidents encountered during the build and operation of this home lab. Each entry includes the symptom, root cause, and resolution — written as a reference and as interview material demonstrating methodical troubleshooting.

---

## Incident Index

| # | Incident | Category | Severity |
|---|---|---|---|
| 01 | ACL Lockout — Core Switch | Security / Access | High |
| 02 | DHCP MAC Binding Failure — NVR | DHCP / Switching | Medium |
| 03 | Camera VLAN — No Internet (3 simultaneous issues) | Routing / NAT / VLANs | High |
| 04 | WAN ACL — RFC 1918 Spoofed Traffic | Security / WAN | Medium |
| 05 | OSPF No Adjacency — R1 to Core Switch | Routing / OSPF | High |
| 06 | HSRP Failover — SSH Lost During Test | HSRP / Access | Medium |
| 07 | EIGRP NHRP Stale Errors | Routing / EIGRP | Low |
| 08 | ESXi vmk0 Migration Blocked | Virtualization | Medium |
| 09 | iSCSI CHAP Bash History Expansion Error | Storage / CLI | Low |
| 10 | QNAP QuFirewall Blocking SNMP | Monitoring / Firewall | Medium |
| 11 | iSCSI — No Space for LUN Creation | Storage | High |
| 12 | BGP Stale Neighbor — AS 16 | Routing / BGP | Medium |
| 13 | Zabbix ICMP Ping Failure — All Routers | Monitoring / Routing | Medium |

---

## Incident 01 — ACL Lockout: Core Switch

**Category:** Security / Access  
**Severity:** High

**Symptom:**  
All SSH access to the core switch (192.168.168.1) was completely blocked. No remote management possible.

**Root Cause:**  
An incorrectly configured ACL was applied to the VTY lines on the core switch. The ACL denied traffic before the permit statement could match, locking out all SSH sessions.

**Resolution:**  
- Connected via console cable directly to the core switch
- Entered privileged exec mode via console
- Removed the incorrectly applied ACL from the VTY lines
- Reloaded the device without saving to restore last known good config
- Rebuilt and re-verified the ACL logic before reapplying

**Lesson Learned:**  
Always verify ACL logic with `show access-lists` and test against a known source before applying to VTY lines. Keep a console cable connected during any ACL changes on remote devices. Never save config until SSH access is re-confirmed.

---

## Incident 02 — DHCP MAC Binding Failure: NVR

**Category:** DHCP / Switching  
**Severity:** Medium

**Symptom:**  
The Reolink NVR (VLAN 30) was not receiving its DHCP reservation. The device received a dynamic address instead of the expected 10.30.0.20.

**Root Cause:**  
The DHCP reservation on the core switch was configured using the bare MAC address (`ec71.db33.3a46`). Cisco IOS DHCP server requires the `client-identifier` format, which prepends a `01` hardware-type prefix byte to the MAC address.

**Resolution:**  
Updated the DHCP reservation to use the correct client-identifier format:
```
ip dhcp pool NVR
   host 10.30.0.20 255.255.255.0
   client-identifier 01ec.71db.333a.46
```

**Lesson Learned:**  
Cisco IOS DHCP reservations use `client-identifier` with a `01` prefix (ARP hardware type for Ethernet), not the raw MAC address. This is a common gotcha when migrating from other DHCP server platforms.

---

## Incident 03 — Camera VLAN 30: No Internet (3 Simultaneous Issues)

**Category:** Routing / NAT / VLANs  
**Severity:** High

**Symptom:**  
Devices on VLAN 30 (10.30.0.0/24) had no internet connectivity after the VLAN was brought online.

**Root Cause:**  
Three independent configuration errors existed simultaneously:

1. **Trunk misconfiguration** — VLAN 30 was not permitted on the trunk link between the core switch and edge router
2. **NAT ACL missing** — The NAT overload (PAT) ACL on the edge router did not include the 10.30.0.0/24 subnet, so traffic from VLAN 30 was not being translated
3. **Missing return route** — The edge router had no route back to 10.30.0.0/24, so even translated return traffic couldn't be forwarded correctly

**Resolution:**  
Each issue resolved independently and verified before moving to the next:
1. Added VLAN 30 to the trunk `allowed vlan` list on the core switch uplink port
2. Added `permit ip 10.30.0.0 0.0.0.255 any` to the NAT ACL on the edge router
3. Added `ip route 10.30.0.0 255.255.255.0 192.168.168.1` on the edge router

**Lesson Learned:**  
When a new VLAN has no connectivity, work layer by layer: L2 trunk → L3 routing → NAT → return routes. Multiple simultaneous issues are common when standing up a new VLAN end-to-end. Isolate each layer before drawing conclusions.

---

## Incident 04 — WAN ACL: RFC 1918 Spoofed Traffic

**Category:** Security / WAN  
**Severity:** Medium (informational / monitoring)

**Symptom:**  
After deploying an inbound WAN ACL on the edge router, hit counters immediately began incrementing on deny statements for RFC 1918 private address space.

**Root Cause:**  
Not a misconfiguration — expected behavior. Internet-facing interfaces routinely receive packets with spoofed source addresses in the RFC 1918 ranges (10.x.x.x, 172.16.x.x, 192.168.x.x), used in DDoS amplification and scanning attempts.

**Resolution:**  
No resolution required. Confirmed ACL was functioning correctly.  
**36,889 spoofed RFC 1918 packets blocked within the first days of deployment.**

WAN ACL deny statements in place:
```
deny ip 10.0.0.0 0.255.255.255 any
deny ip 172.16.0.0 0.15.255.255 any
deny ip 192.168.0.0 0.0.255.255 any
```

**Lesson Learned:**  
Inbound WAN ACLs are essential even on home/lab connections. The volume of unsolicited inbound traffic attempting to spoof internal addresses confirms that internet-facing interfaces should always have anti-spoofing ACLs applied.

---

## Incident 05 — OSPF No Adjacency: R1 to Core Switch

**Category:** Routing / OSPF  
**Severity:** High

**Symptom:**  
R1 was not forming an OSPF neighbor adjacency with the core switch. `show ip ospf neighbor` on R1 showed no entry for the core switch despite both devices being configured for OSPF Area 0 on the same subnet.

**Root Cause:**  
The `passive-interface` command was configured on R1's Fa0/0 interface — the interface connecting R1 to the core switch. Passive interfaces suppress OSPF hello packets, preventing adjacency formation.

**Resolution:**  
```
R1(config-router)# no passive-interface FastEthernet0/0
```
OSPF adjacency formed immediately after removal. R1 became BDR as expected.

**Lesson Learned:**  
`passive-interface` is commonly used to suppress OSPF hellos on user-facing or external interfaces — but it must never be applied to interfaces that need to form adjacencies. When OSPF neighbors won't form, always check passive-interface configuration first with `show ip ospf interface`.

---

## Incident 06 — HSRP Failover: SSH Lost During Test

**Category:** HSRP / Access  
**Severity:** Medium

**Symptom:**  
During a planned HSRP failover test, SSH connectivity to the network was lost when R1 (the active router) was taken offline. Could not reconnect until the test was aborted.

**Root Cause:**  
SSH sessions were established directly to R1's physical IP (192.168.168.2) rather than through the HSRP virtual IP (192.168.168.254). When R1 went offline, those sessions were lost with no automatic failover at the SSH layer.

**Resolution:**  
Reconnected via console cable. This incident revealed a procedural gap: HSRP tests require console access to at least one device before the test begins in case of management loss.

**Lesson Learned:**  
- Always SSH to the HSRP virtual IP (192.168.168.254) for management tasks, not physical IPs
- Attach console cable to at least one device before any HSRP or redundancy test
- Verified: HSRP failover completes in under 10 seconds; preemption returns R1 to Active automatically

---

## Incident 07 — EIGRP NHRP Stale Errors

**Category:** Routing / EIGRP  
**Severity:** Low

**Symptom:**  
NHRP-related error messages appeared in the EIGRP log. Neighbor statements referenced addresses that did not correspond to any active device.

**Root Cause:**  
Stale EIGRP neighbor statements left in the configuration by the previous owner of the Cisco 2811 routers. The neighbor entries pointed to devices that no longer existed on the network.

**Resolution:**  
Identified all stale neighbor statements using `show running-config | section router eigrp` across all routers. Removed each stale entry:
```
no neighbor [stale-ip-address]
```

**Lesson Learned:**  
Always audit full running configs on used/second-hand Cisco equipment before deployment. Previous owner configurations can cause subtle errors that are hard to trace. A full `write erase` and fresh config is preferable when starting a new lab build.

---

## Incident 08 — ESXi vmk0 Migration Blocked

**Category:** Virtualization  
**Severity:** Medium

**Symptom:**  
Attempted to migrate the ESXi management kernel port (vmk0) from the DS-ServerNet vSwitch to vSwitch0 to consolidate networking. Both the vSphere GUI and CLI (esxcli) blocked the operation.

**Root Cause:**  
ESXi 6.7 does not permit migration of vmk0 (the management VMkernel port) between vSwitches without vCenter Server. The operation is intentionally restricted in standalone ESXi host management to prevent management network loss.

**Resolution:**  
Operation left as-is. vmk0 remains on DS-ServerNet. This is a known ESXi limitation — the migration requires vCenter to maintain a management connection throughout the process.

**Lesson Learned:**  
Standalone ESXi has intentional restrictions around vmk0 migration. Future resolution: deploy vCenter Server (can be done via VCSA on the existing ESXi host) to unlock this and other advanced vSphere management features. Added to lab backlog.

---

## Incident 09 — iSCSI CHAP Password: Bash History Expansion Error

**Category:** Storage / CLI  
**Severity:** Low

**Symptom:**  
When entering the iSCSI CHAP password (`ZabbixDB2026!`) in a bash command on the ESXi host, the command failed with a bash error rather than executing correctly.

**Root Cause:**  
The `!` character at the end of the password triggered bash history expansion. In bash, `!` followed by text attempts to recall a previous command from history, causing the command to fail or execute unintended commands.

**Resolution:**  
Removed the `!` character from the CHAP password. Updated password to `ZabbixDB2026` (no special character). Re-entered the CHAP credentials on both the ESXi iSCSI software adapter and the QNAP NAS target configuration.

**Lesson Learned:**  
Avoid `!` in passwords used in bash CLI operations. If special characters are required, wrap the entire argument in single quotes (single quotes prevent all bash interpretation) or escape the character with a backslash. Always test credential entry in a non-production context first.

---

## Incident 10 — QNAP QuFirewall Blocking SNMP

**Category:** Monitoring / Firewall  
**Severity:** Medium

**Symptom:**  
After re-enabling QuFirewall on both QNAP NAS units, Zabbix began showing red/problem badges for both NAS hosts. SNMP polling from the Zabbix VM (10.20.0.10) was being blocked.

**Root Cause:**  
QuFirewall was re-enabled with only the 192.168.168.0/24 management subnet in the allow rules. The Zabbix monitoring VM sits on 10.20.0.0/24 (VLAN 20 — Servers), which was not included in the QuFirewall allow list.

**Resolution:**  
Added 10.20.0.0/24 to the QuFirewall allow rules on both NAS units:
- QNAP NAS1 (192.168.168.40) — QuFirewall updated
- QNAP NAS2 (192.168.168.41) — QuFirewall updated

Final QuFirewall allow rules on both units:
```
Allow: 192.168.168.0/24
Allow: 10.20.0.0/24
Allow: 10.30.0.0/24
Allow: 10.40.0.0/24
```

**Lesson Learned:**  
When enabling host-based firewalls on monitored devices, always verify that the monitoring platform's source subnet is explicitly permitted. SNMP uses UDP 161 — easily blocked silently. After any firewall change, immediately verify monitoring connectivity.

---

## Incident 11 — iSCSI: No Space for LUN Creation

**Category:** Storage  
**Severity:** High

**Symptom:**  
Attempted to create a 1TB iSCSI LUN on QNAP NAS2 for the ESXi datastore. The NAS reported only 8.80GB free in the storage pool despite the unit having 8× 1TB SSDs in RAID 6.

**Root Cause:**  
DataVol1 on NAS2 had been created as a **thick-provisioned** volume, which immediately reserved the full allocated capacity from the storage pool. The thick volume was consuming nearly the entire pool, leaving only 8.80GB available.

**Resolution:**  
1. Identified DataVol1 as thick-provisioned via QNAP Storage Manager
2. Converted DataVol1 from thick to **thin provisioning**
3. Conversion freed **4.60TB** back to the available storage pool
4. Successfully created 1TB thin-provisioned iSCSI LUN (`esxilun01`)
5. Mounted on ESXi as NAS2-iSCSI-DS01 (999.75GB, VMFS6)

**Lesson Learned:**  
Thick provisioning reserves physical space immediately regardless of actual data written. For lab environments and iSCSI datastores where capacity management is important, thin provisioning is almost always preferable. Always verify storage pool free space and provisioning type before LUN creation.

---

## Incident 12 — BGP Stale Neighbor: AS 16

**Category:** Routing / BGP  
**Severity:** Medium

**Symptom:**  
All routers showed a BGP neighbor entry for `16.16.16.16` in AS 16 in a persistent `Idle` state. The neighbor never established and generated log noise across R1, R2, R3, and R4.

**Root Cause:**  
BGP neighbor statements for `neighbor 16.16.16.16 remote-as 16` were present in the configuration of all four routers, left by the previous owner of the equipment.

**Resolution:**  
Removed stale neighbor statements from all four routers:
```
router bgp [local-AS]
  no neighbor 16.16.16.16 remote-as 16
```

**Lesson Learned:**  
Second-hand Cisco equipment often contains previous lab or production configurations. BGP stale neighbors in `Idle` state are benign but generate log noise and can complicate `show ip bgp summary` output. Always perform a full config audit on acquired equipment. A `write erase` before initial lab config is best practice.

---

## Incident 13 — Zabbix ICMP Ping Failure: All Routers

**Category:** Monitoring / Routing  
**Severity:** Medium

**Symptom:**  
All routers (R1–R4) showed ICMP ping as unavailable in Zabbix despite SNMP polling working correctly. Zabbix reported the routers as reachable via SNMP but unreachable via ICMP.

**Root Cause:**  
ICMP ping from Zabbix originates from the Zabbix VM's IP address: 10.20.0.10 (VLAN 20, Servers subnet). The routers had no route back to 10.20.0.0/24 — only the management VLAN (192.168.168.0/24) routes were present. SNMP worked because the SNMP polling mechanism was handled differently, but direct ICMP responses had no return path.

**Resolution:**  
Added a static return route on all four routers pointing 10.20.0.0/24 back toward the core switch:
```
ip route 10.20.0.0 255.255.255.0 192.168.168.1
```
Applied to: R1, R2 (via R1 hop), R3, R4. ICMP ping immediately went green in Zabbix for all router hosts.

**Lesson Learned:**  
Monitoring platforms on non-management VLANs require return routes on all monitored devices. SNMP and ICMP behave differently — SNMP responses may succeed while ICMP fails if return routing is asymmetric. When adding monitoring for a new subnet, always add return routes to all monitored devices as part of the setup procedure.

---

*Last updated: March 2026 | 13 incidents documented*
