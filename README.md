# Homelab Network Topology

**Purpose:** Segmented homelab network: firewall & edge, routed VLANs, Proxmox-hosted services, monitoring and automated config backups. This README documents addressing, routing/VLAN design, monitoring, and backup flows. See `HomeLab.jpg` in the repo for the visual topology referenced throughout.

---

## At a glance

| Entity | Device / Subnet | Purpose |
|---|---:|---|
| WAN / Edge | `ISP Modem` → `pfSense` | Internet gateway, DHCP, NAT, IPsec site-to-site VPN |
| pfSense Router | `10.0.0.0/24` | Default LAN, printer, general admin workstation |
| Cisco Router | VLANs `192.168.2.0/24`, `192.168.3.0/24` | Infrastructure / servers (Proxmox + VMs), Workstations |
| Wireless | Access Point - Subnet `10.1.0.0/24` | Wireless access (DNS filtering / safe-browsing) |
| Offsite - AWS | E2C Virtual Machine | Gitea on AWS for config backups |
| Offsite - Cluster| Windows Server `172.17.0.6/22` (site-to-site VPN)| Remote Proxmox + Windows DC |

---

## 1 — WAN/Edge Router (pfSense)

**Role:** primary gateway to the Internet, DHCP server for `10.0.0.0/24` and `10.1.0.0/24` networks, firewall, and site-to-site IPsec termination.

**Key configuration & behavior**
- **External connection:** ISP modem provides the public IP; pfSense sits behind it as the internal gateway.
- **LAN subnet:** `10.0.0.0/24`
  - DHCP pools:
    - Dynamic: `10.0.0.3–10.0.0.99`
    - Static/reserved: `10.0.0.100–10.0.0.254` (reserved for servers/printers)
    - `10.0.0.146` — Network printer
    - `10.0.0.125` — Local admin workstation
- **Wireless subnet:** `10.1.0.0/24` — DNS content filtering service applied.
- **Site-to-site IPsec VPN:** connects to remote homelab `172.17.0.0/22`. Remote Domain Controller: `172.17.10.13`. Tunnel provides routing between local networks and remote AD/DC for authentication and resource access.
- **Internal Routing:** pfSense routes traffic to the Cisco router for internal networks (192.168.2.0/24, etc.).

**Design notes:**
- pfSense centralizes perimeter security and VPN termination.
- Wireless is segmented to reduce risk to core infrastructure and enforce safe-browsing by DNS policy.

---

## 2 — Internal Router (Cisco)

**Devices:** Cisco ISR4331 (router), Catalyst WS-C3750 (switch)

**Design & VLANs**
- **Router subinterfaces (dot1Q tagging):**
  - `Gi0/1.2` → VLAN 2 → `192.168.2.0/24` (server/infrastructure)
  - `Gi0/1.3` → VLAN 3 → `192.168.3.0/24` (workstations)
- **Switch trunking:** switch uplink uses 802.1Q trunk to carry VLAN 2 and VLAN 3 to the Cisco router. Port connecting Proxmox is an access port assigned to VLAN 2.
- **Inter-VLAN routing & ACLs:** Router enforces routing policies, ACLs, and NAT upstream to pfSense router and Internet egress.

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
- Cisco router represents successful internal routing and network segmentation; reduces attack surface and simplifies network management.

---

## 3 — Server Infrastructure (VLAN 2 — `192.168.2.0/24`)

**Proxmox Host**
- **Host IP:** `192.168.2.2` (Proxmox VE host; VMs bridged to VLAN 2)

**VM inventory (IPs & roles)**
- `192.168.2.100` — **Zabbix**  
  - Monitors: pfSense, Cisco ISR, Catalyst switch, TrueNAS, MySQL servers, Oxidized, WAN/VPN endpoints, offsite Windows DC, AWS Gitea. Uses SNMP, agent checks, and ICMP/TCP probes.
- `192.168.2.3` — **MySQL (Zabbix DB)**  
- `192.168.2.4` — **MySQL Backup** (replica / scheduled snapshot target)  
- `192.168.2.5` — **Windows Workstation** (domain-joined via site-to-site VPN)  
- `192.168.2.6` — **TrueNAS** (SMB shares for backups/ISOs)  
- `192.168.2.7` — **Oxidized** (pulls configs from pfSense & Cisco; pushes to AWS Gitea)

**Access & admin**
- VMs are reachable via SSH through controlled NAT/port-forward entries on the Cisco router. Proxmox, Zabbix, TrueNas, etc. web UIs are not publicly exposed.

---

## 4 — VLAN 3 (End-user / Workstation)

- `192.168.3.2` — End-user workstation
- **Purpose:** user tasks and testing for successful establishment of multiple VLANs.

---

## 5 — External Resources

- **AWS EC2 (Gitea)** — remote Git server for network config backups pushed by Oxidized server residing locally on VLAN 2.
- **Remote Proxmox cluster** — reachable via pfSense IPsec tunnel (`172.17.0.0/22`); Windows DC at `172.17.10.13` provides AD services to domain-joined clients.
- **Purpose:** Successful establishment of site-to-site persistent VPN tunnel, remote domain controller, and AWS EC2 implementation with elastic IP.

---

## 6 — Monitoring & Backups

**Zabbix**
- Uses SNMP, Zabbix agent, ICMP/TCP checks to monitor device uptime, resource metrics and services.
- Alerts configured for tunnel failure, host down, service degradation.

**Oxidized**
- Polls pfSense and Cisco routers (SSH) on schedule, backup configs locally, and then pushes configs to offsite Git repository (AWS EC2 running Rocky Linux - Gitea).

**Backups**
- MySQL primary replicates/dumps to MySQL backup at `192.168.2.4` 

---

## 7 — Access, Security & Hardening (summary)

- **Perimeter:** pfSense enforces stateful firewalling and terminates IPsec VPNs.
- **Segmentation:** Servers (VLAN 2) isolated from workstations (VLAN 3), pfSense LAN (10.0.0.0/24), and wireless (10.1.0.0/24).
- **Least privilege:** Management ports are not exposed publicly.
- **Config backups:** Oxidized → Gitea (offsite) provides versioning and rollback for device configs.
- **Monitoring:** Zabbix provides alerts and trend analysis.

---

## 8 — Decommissioned

- **AWS VPN Tunnel:** This Site-to-Site VPN (AWS VPN Connection vpn-0988432276dde398e) was fully configured between the Cisco router and an AWS Virtual Private Gateway. The connection used two redundant IPSec tunnels. The VPN has now been decommissioned: the VPN Connection object was removed from AWS:

Example (redacted) router snippets that were in place prior to decommissioning:

```text
crypto isakmp policy 200
  encryption aes 128
  authentication pre-share
  group 2
  lifetime 28800
  hash sha
exit

crypto keyring keyring-vpn-0988432276dde398e-0
  local-address 76.112.64.147
  pre-shared-key address 18.116.105.57 key <REDACTED-PSK-1>
exit

interface Tunnel1
  ip address 169.254.128.154 255.255.255.252
  tunnel source 76.112.64.147
  tunnel destination 18.116.105.57
  tunnel protection ipsec profile ipsec-vpn-0988432276dde398e-0
  ip tcp adjust-mss 1379
  no shutdown
exit
```
---
