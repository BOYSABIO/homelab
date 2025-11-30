# **Layer 0 — Network & Physical Foundation (Blueprint)**

## **1. Purpose of Layer 0**

Layer 0 defines the **physical**, **network**, and **segmentation** foundations of my homelab.  
It does **not** require hardware changes immediately — it establishes the long-term architecture that all future layers will build on.

Layer 0 is essentially the “blueprint” that defines:
- Future network layout
- VLAN segmentation
- Where services will live
- How devices will communicate
- What hardware roles exist
- How security boundaries are enforced

When Layer 0 becomes physical (later), it forms a secure, enterprise-like environment.

---

# **2. Final Network Topology (Permanent Design)**

```
                    INTERNET
                        |
                     Modem
                        |
               [WAN] Firewall (pfSense/OPNsense)
                        |
               [LAN Trunk] Managed Switch
     -------------------------------------------------
     |           |            |            |         |
 Mgmt VLAN    Trusted      Services      Lab VLAN   IoT
   (10)        (20)          (30)          (40)      (50)
```

---
# **3. VLANs & Subnets (Final)**

|VLAN ID|Name|Purpose|Subnet|
|---|---|---|---|
|**10**|Management|Firewall, Proxmox, Switch, NAS, monitoring|10.xx.xx.0/24|
|**20**|Trusted/Home|Personal devices: PC, laptop|10.aa.bb.0/24|
|**30**|Services|SIEM, GitLab, Jupyter, ML APIs, Databases|10.cc.dd.0/24|
|**40**|Security Lab|Kali, vuln VMs, malware environments|10.ee.ff.0/24|
|**50**|IoT|Smart home / untrusted devices|10.gg.hh.0/24|
|**60**|DMZ (Optional)|Public-facing services behind reverse proxy|10.uu.uu.0/24|

---

# **4. Traffic Rules (Planned Security Boundaries)**

### **Management (10)**
- Full access to all VLANs
- Restricted to admin devices only

### **Trusted (20)**
- Can access Services (30)
- Can access Internet
- Cannot access Lab (40)

### **Services (30)**
- Accessible from Trusted + Mgmt
- Cannot initiate connections to Lab (40)

### **Security Lab (40)**
- Can access Internet (optional)
- Cannot reach Trusted (20) or Mgmt (10)
- Can communicate internally within VLAN

### **IoT (50)**
- Only outbound Internet access
- No access to any other VLAN

---
# **5. Final Hardware Roles (Long-Term Vision)**

## **Firewall Appliance (pfSense / OPNsense)**
**Role**: routing, VLANs, firewall rules, VPN, DHCP, logging  
**Requirements**:
- 2–4 NIC ports
- Runs pfSense/OPNsense  
    **Examples**:
	- Protectli
	- Qotom mini-PC
	- Used Dell/HP small form factor PC + Intel NIC

---

## **Managed Switch (VLAN-capable)**
**Role**: VLAN tagging, trunk ports, segmentation, SPAN/mirroring  
**Requirements:**
- 8–16 ports
- 802.1Q VLAN support  
    **Examples:**
	- TP-Link TL-SG2008 / SG2016
	- Netgear GS308E / GS316E
	- Cisco SG250 series

---

## **Proxmox Hypervisor (Main Compute Node)**
**Role**: Runs all VMs for the lab (AD, SIEM, vuln VMs, etc.)  
**Requirements:**
- 32GB–64GB RAM
- At least 4 cores
- SSD storage
- Intel NIC for VLAN tagging  
    **Examples:**
	- Used workstation (Dell OptiPlex 7040, HP Z-series)
	- Used server (Dell R210/R710, HP DL380)
	- MinisForum/Beelink AMD mini-PCs

---

## **NAS (Storage Server)**
**Role**: central storage for logs, PCAPs, datasets, backups  
**Requirements:**
- RAID/ZFS recommended
- 2–4 drives  
    **Examples:**
	- TrueNAS Core on old PC
	- Synology/QNAP
	- Unraid build

---

# **6. Temporary Layer 0 (Current Implementation)**
Until hardware is acquired:
- ISP router acts as temporary firewall
- Network is flat (no true VLANs)
- Segmentation is simulated using:
    - VirtualBox internal networks
    - Software firewalls
    - Host-only adapters
- Proxmox may run nested on main PC for learning
- NAS → replaced by folders or external SSD

This allows development of VMs and services while designing the final topology.

---
# **7. Definition of Done (Layer 0 Completion Criteria)**
Layer 0 is considered _fully complete_ when:
### **Network**
- VLANs 10/20/30/40/50/60 are created
- Managed switch enforces VLAN tagging
- Firewall handles inter-VLAN routing

### **Services**
- DHCP configured per VLAN
- DNS configured per VLAN
- Gateway separation implemented

### **Security**
- “Lab” VLAN cannot reach Trusted/Mgmt
- “Trusted” VLAN can reach Services
- “Mgmt” VLAN restricted to admin devices

### **Hardware**
- Dedicated firewall appliance in place
- Managed switch deployed
- Proxmox node connected via trunk
- NAS connected to Mgmt VLAN

### **Documentation**
- All diagrams, subnet plans, and traffic rules documented here
- Hardware list and future purchases clearly defined

---
# **8. Notes / Future Additions**
- Add 2nd Proxmox node for clustering
- Add reverse-proxy in DMZ
- Add VPN for remote admin
- Add SPAN port for Zeek/Suricata traffic capture
- **USE Firewall as vpn in order to act as your own tailscale net. Tailscale built on wireguard so you can use wireguard and connect devices from anywhere to the network. Then, use VPN on top to also hide output traffic or to change exit output node. Connect to your home firewall VPN → then route traffic through a commercial VPN provider from inside your lab.**