# Zabbix Monitoring Setup

Home Lab — Curtis | CCNA | March 2026

---

## Overview

Zabbix 7.0 LTS is deployed as a VM on ESXi, monitoring 14 devices across the lab via SNMP. All hosts are currently green with 2,295+ monitored items and 1,263 configured triggers.

---

## Installation Details

| Parameter | Value |
|---|---|
| Version | Zabbix 7.0.24 LTS |
| VM Host | ESXi S2 (Dell PowerEdge R630) |
| VM OS | Ubuntu 24.04.4 LTS |
| VM Specs | 4 vCPU, 8GB RAM, 100GB disk |
| VM IP | 10.20.0.10 (VLAN 20 — Servers) |
| Web UI | http://10.20.0.10/zabbix |
| Database | MySQL — db: zabbix, user: zabbix |
| Web Server | Apache 2.4 with PHP 8.3 |
| Datastore | NAS2-iSCSI-DS01 (iSCSI, VMFS6) |

---

## Installation Steps (Ubuntu 24.04)

### 1. Install Zabbix repository

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update
```

### 2. Install Zabbix server, frontend, and agent

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

### 3. Set up MySQL database

```bash
mysql -uroot -p
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by '<password>';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;

zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

### 4. Configure Zabbix server

Edit `/etc/zabbix/zabbix_server.conf`:
```
DBPassword=<your-password>
```

### 5. Start services

```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

### 6. Complete web setup

Navigate to `http://10.20.0.10/zabbix` and complete the web-based setup wizard.

---

## SNMP Configuration

### Community String
All devices use SNMP community string: `[REDACTED]` (read-only)

### Cisco IOS Devices (Routers and Switches)

Add to running config on each device:

```
snmp-server community [REDACTED] RO
snmp-server location Home-Lab
snmp-server contact curtis
```

Verify with:
```
show snmp
show snmp community
```

### ESXi Host

SNMP must be enabled via esxcli (cannot be done through vSphere GUI on standalone host):

```bash
esxcli system snmp set --communities [REDACTED] --enable true
esxcli system snmp get
```

Verify SNMP is listening:
```bash
esxcli network ip connection list | grep 161
```

### QNAP NAS Units

1. Log into QNAP web UI → Control Panel → Network & File Services → SNMP
2. Enable SNMPv1/v2
3. Set community string to match lab standard
4. **Critical:** Also configure QuFirewall to allow SNMP from monitoring subnet:
   - Control Panel → Security → Security Level → QuFirewall
   - Add allow rules for all internal subnets:
     - 192.168.168.0/24
     - 10.20.0.0/24
     - 10.30.0.0/24
     - 10.40.0.0/24

---

## Return Routes (Critical)

Zabbix polls from 10.20.0.10 (VLAN 20). All routers need a return route to this subnet or ICMP ping will fail even when SNMP works.

Add to all routers (R1, R2, R3, R4) and edge router:

```
ip route 10.20.0.0 255.255.255.0 192.168.168.1
```

> **Note:** This was discovered during setup when all routers showed ICMP unavailable in Zabbix despite SNMP polling working correctly. See Incident 13 in the troubleshooting log.

---

## Monitored Hosts

| Host | IP | Template | Items | Status |
|---|---|---|---|---|
| Core Switch WS-C3750X | 192.168.168.1 | Cisco Catalyst 3750V2-48TS by SNMP | 592 | ✅ Green |
| edge-rtr | 192.168.168.168 | Cisco IOS by SNMP | 82 | ✅ Green |
| R1 | 192.168.168.2 | Cisco IOS by SNMP | 111 | ✅ Green |
| R2 | 10.1.12.2 | Cisco IOS by SNMP | 92 | ✅ Green |
| R3 | 192.168.168.3 | Cisco IOS by SNMP | 102 | ✅ Green |
| R4 | 192.168.168.4 | Cisco IOS by SNMP | 93 | ✅ Green |
| access-sw-01 | 192.168.168.51 | Cisco IOS by SNMP | 289 | ✅ Green |
| access-sw-02 | 192.168.168.52 | Cisco IOS by SNMP | 290 | ✅ Green |
| access-sw-03 | 192.168.168.53 | Cisco IOS by SNMP | 271 | ✅ Green |
| access-sw-04 | 192.168.168.54 | Cisco IOS by SNMP | 281 | ✅ Green |
| QNAP NAS1 TS-453BU | 192.168.168.40 | Linux by SNMP | 116 | ✅ Green |
| QNAP NAS2 TS-832PXU-RP | 192.168.168.41 | Linux by SNMP | 107 | ✅ Green |
| ESXi R630-2 | 192.168.168.16 | Linux by SNMP | 44 | ✅ Green |
| Zabbix server | 127.0.0.1 | Zabbix agent (built-in) | 153 | ✅ Green |

**Totals: 14 hosts | 2,295+ items | 1,263 triggers | All green**

---

## Template Notes

### Cisco IOS by SNMP
Used for all Cisco routers and access switches. Monitors interface status, CPU, memory, OSPF neighbors, and more. Applied directly from Zabbix built-in template library.

### Cisco Catalyst 3750V2-48TS by SNMP
Applied to the core switch. Provides more detailed switching-specific metrics including PoE port monitoring (592 items). This template provides significantly more visibility than the standard Cisco IOS template.

### Linux by SNMP
Applied to QNAP NAS units and ESXi host. While these are not standard Linux systems, the Linux by SNMP template works correctly for SNMP-enabled QNAP and ESXi devices.

### Zabbix Agent (Built-in)
The Zabbix server monitors itself via the local agent. 153 items covering server performance, queue depth, and internal Zabbix metrics.

---

## Known Issues and Workarounds

### Edge Router Temperature False Positive
The Cisco IOS template generates a temperature alert on the edge router (Cisco 1921) that does not reflect an actual hardware problem. This trigger has been suppressed in Zabbix.

**Workaround:** Created a maintenance window / trigger override to suppress the false positive temperature alert on host `edge-rtr`.

---

## Useful Verification Commands

### On Zabbix VM
```bash
# Check Zabbix server status
systemctl status zabbix-server

# Test SNMP connectivity to a device
snmpwalk -v2c -c [REDACTED] 192.168.168.1

# Check Zabbix server log
tail -f /var/log/zabbix/zabbix_server.log
```

### On Cisco Devices
```bash
# Verify SNMP is configured
show snmp
show snmp community

# Verify return route to Zabbix subnet
show ip route 10.20.0.0
```

---

*Last updated: March 2026*
