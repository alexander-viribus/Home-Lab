# Homelab Network Topology

**Purpose:** Employer-facing documentation of a segmented homelab network: firewall & edge, routed VLANs, Proxmox-hosted services, monitoring and automated config backups. This README documents addressing, routing/VLAN design, monitoring, and backup flows. See `diagram.jpg` in the repo for the visual topology referenced throughout.

---

## At a glance

| Layer | Device / Subnet | Purpose |
|---|---:|---|
| WAN / Edge | `ISP Modem` → `pfSense` | Internet gateway, DHCP, NAT, IPsec site-to-site VPN |
| LAN (pfSense) | `10.0.0.0/24` | Default LAN, printer, general admin workstation |
| Wireless | `10.1.0.0/24` | Wireless AP (DNS filtering / safe-browsing) |
| Routed VLANs | VLAN 2: `192.168.2.0/24` | Infrastructure / servers (Proxmox + VMs) |
| Routed VLANs | VLAN 3: `192.168.3.0/24` | End-user workstation(s) |
| Offsite / cloud | `172.17.0.0/22` (remote site) / AWS | Remote Proxmox + Windows DC; Gitea on AWS for config backups |

---

## 1 — WAN & Edge (pfSense)

**Role:** primary gateway to the Internet, DHCP server, edge firewall, NAT, and site-to-site IPsec termination.

**Key configuration & behavior**
- **External connection:** ISP modem provides the public IP; pfSense sits behind it as the internal gateway.
- **LAN subnet:** `10.0.0.0/24`
  - DHCP pools:
    - Dynamic: `10.0.0.3–10.0.0.99`
    - Static/reserved: `10.0.0.100–10.0.0.254` (reserved for servers/printers)
- **Wireless subnet:** `10.1.0.0/24` — DNS filtering applied (e.g., pfBlockerNG or filtered resolver).
- **Site-to-site IPsec VPN:** connects to remote homelab `172.17.0.0/22`. Remote Domain Controller: `172.17.10.13`. Tunnel provides routing between local networks and remote AD/DC for authentication and resource access.
- **NAT / Port forwards:** pfSense handles upstream NAT. Selected inbound ports (SSH, management ports) are forwarded to the Cisco router which then NATs/forwards to internal VMs (controlled exposure).

**Design notes (employer-focused):**
- pfSense centralizes perimeter security and VPN termination.
- Wireless is segmented to reduce risk to core infrastructure and enforce safe-browsing by DNS policy.

---

## 2 — Routing Layer (Cisco)

**Devices:** Cisco ISR4331 (router), Catalyst WS-C3750 (switch)

**Design & VLANs**
- **Router subinterfaces (dot1Q tagging):**
  - `Gi0/1.2` → VLAN 2 → `192.168.2.0/24` (server/infrastructure)
  - `Gi0/1.3` → VLAN 3 → `192.168.3.0/24` (workstations)
- **Switch trunking:** switch uplink uses 802.1Q trunk to carry VLAN 2 and VLAN 3 to the Cisco router. Port connecting Proxmox is an access port assigned to VLAN 2.
- **Inter-VLAN routing & ACLs:** Router enforces routing policies and ACLs between VLANs; pfSense remains upstream for NAT and Internet egress.

**Illustrative Cisco snippets (example only):**
```text
interface GigabitEthernet0/1
 no ip address
!
interface GigabitEthernet0/1.2
 encapsulation dot1q 2
 ip address 192.168.2.1 255.255.255.0
!
interface GigabitEthernet0/1.3
 encapsulation dot1q 3
 ip address 192.168.3.1 255.255.255.0
```
```text
interface GigabitEthernet1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 2,3
```

**Design notes:**
- Router handles inter-VLAN routing and ACL enforcement between VLANs; segmentation reduces attack surface and simplifies network management.

---

## 3 — Server Infrastructure (VLAN 2 — `192.168.2.0/24`)

**Proxmox Host**
- **Host IP:** `192.168.2.2` (Proxmox VE host; VMs bridged to VLAN 2)

**VM inventory (IPs & roles)**
- `192.168.2.100` — **Zabbix**  
  - Monitors: pfSense, Cisco ISR, Catalyst switch, TrueNAS, MySQL servers, Oxidized, WAN/VPN endpoints, offsite Windows DC, AWS Gitea. Uses SNMP, agent checks and ICMP/TCP probes.
- `192.168.2.3` — **MySQL (Zabbix DB)**  
- `192.168.2.4` — **MySQL Backup** (replica / scheduled snapshot target)  
- `192.168.2.5` — **Windows Workstation** (domain-joined via VPN)  
- `192.168.2.6` — **TrueNAS** (SMB shares for backups/ISOs)  
- `192.168.2.7` — **Oxidized** (pulls configs from pfSense & Cisco; pushes to AWS Gitea)

**Access & admin**
- VMs are reachable via SSH through controlled NAT/port-forward entries on the Cisco router. Proxmox web UI is not publicly exposed — use jump hosts or VPN for administration.

---

## 4 — VLAN 3 (End-user / Workstation)

- `192.168.3.2` — End-user workstation
- Purpose: user tasks and testing. Routed access to VLAN 2 is controlled via ACLs on the router.

---

## 5 — LAN (pfSense LAN / `10.0.0.0/24`)

- `10.0.0.146` — Network printer
- `10.0.0.125` — Local admin workstation
- Note: this is the pfSense-managed home LAN. Access to VLANs controlled by firewall rules.

---

## 6 — External Resources

- **AWS EC2 (Gitea)** — remote Git server for Oxidized backups (offsite versioning & audit history).
- **Remote Proxmox cluster** — reachable via pfSense IPsec tunnel (`172.17.0.0/22`); Windows DC at `172.17.10.13` provides AD services to domain-joined clients.

---

## 7 — Monitoring & Backups

**Zabbix**
- Uses SNMP, Zabbix agent, ICMP/TCP checks to monitor device uptime, resource metrics and services.
- Alerts configured for tunnel failure, host down, service degradation; integrate with email/Slack/webhooks.

**Oxidized**
- Polls network devices (SSH) on schedule (or on-change) and commits configs to a Git repository (AWS Gitea) for versioning and rollback.

**Backups**
- MySQL primary replicates/dumps to `192.168.2.4` and snapshots to TrueNAS. Critical configs are versioned in Gitea offsite.

---

## 8 — Access, Security & Hardening (summary)

- **Perimeter:** pfSense enforces stateful firewalling and terminates IPsec VPNs.
- **Segmentation:** Servers (VLAN 2) isolated from workstations (VLAN 3) and wireless (10.1.0.0/24).
- **Least privilege:** Management ports are not exposed publicly; remote admin uses NATed SSH with logging or VPN/jump hosts.
- **Config backups:** Oxidized → Gitea (offsite) provides versioning and rollback for device configs.
- **Monitoring:** Zabbix provides alerts and trend analysis.
- **Recommendations:** enable MFA for admin accounts, strict router ACLs for inter-VLAN flows, and scheduled updates/patching for all network and VM hosts.

---

## 9 — Quick verification & troubleshooting commands

**Cisco router / switch**
```text
show ip interface brief
show running-config interface GigabitEthernet0/1
show vlan brief
show ip route
```

**pfSense (GUI / diagnostics)**
- GUI: Status → System Logs, Status → IPsec
- CLI/Diagnostics test commands: `ping`, `traceroute`; `ipsec statusall` on pfSense shell if available

**Monitoring host / admin shell**
```bash
# test SSH reachability
nc -vz 192.168.2.7 22

# SNMP walk example
snmpwalk -v2c -c <community> 192.168.2.1 SNMPv2-MIB::sysDescr.0

# test web UI
curl -I http://192.168.2.100
```

---

## 10 — Operational summary (for employers / auditors)

- **Why this design:** VLAN segmentation reduces blast radius, centralizes monitoring and backups, and allows secure cross-site services via IPsec.
- **High-value features:** automated configuration backups (Oxidized → Gitea), centralized monitoring (Zabbix), site-to-site VPN for AD and replication workflows.
- **Change control:** device config commits in Gitea provide an auditable history of changes; Zabbix confirms availability and trends.

---

## Change log
- `v1.0` — Initial README: network layout, subnets, devices, monitoring & backup flows.

---
