# Homelab Network Topology

## WAN & Edge
- **ISP Modem** (public IP)
- **pfSense Router/Firewall** – 10.0.0.0/24  
  - DHCP Pool:  
    - Dynamic: 10.0.0.3–10.0.0.99  
    - Static: 10.0.0.100–10.0.0.254  
  - Wireless AP: 10.1.0.0/24 (DNS restricted for safe browsing)  
  - Site-to-site VPN (IPsec) with friend: 172.17.0.0/22  
    - Remote Domain Controller: 172.17.10.13  
  - NAT upstream for Cisco Router

---

## Routing Layer
- **Cisco Router** (NATed to pfSense)  
  - Subinterface Gi0/1.2 → VLAN 2 → 192.168.2.0/24  
  - Subinterface Gi0/1.3 → VLAN 3 → 192.168.3.0/24  
- **Cisco Switch**  
  - Port 2 (VLAN 2) → Proxmox Server

---

## Proxmox Server (VLAN 2: 192.168.2.0/24)
- Host: 192.168.2.2  

### VMs
1. **Zabbix Network Monitoring** – 192.168.2.100  
   - Monitors: pfSense, Cisco Router, Cisco Switch, MySQL VMs, WAN, TrueNAS, Windows DC (remote), Oxidized, Gitea (AWS)  
2. **MySQL-Rocky** (DB for Zabbix) – 192.168.2.3  
3. **MySQL-Rocky-BA** (DB backup) – 192.168.2.4  
4. **Windows Workstation** – 192.168.2.5  
   - Domain joined to remote Windows DC (172.17.10.13) via VPN  
5. **TrueNAS** – 192.168.2.6  
   - SMB network storage  
6. **Oxidized** – pulls configs from pfSense & Cisco Router, pushes backups to AWS Gitea  

---

## Additional Notes
- All Proxmox VMs are behind the Cisco Router and accessible via port forwarding (SSH/NAT).  
- **Network Printer** – 10.0.0.146 (on pfSense LAN).  
- Zabbix provides monitoring for all core infra components + WAN + VPN-connected remote resources.  

