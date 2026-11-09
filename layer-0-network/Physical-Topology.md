# Layer 0 — Physical Topology

> Current physical state of the homelab network.
> This is a living document — updated as hardware changes.

---

## Current Phase

**Phase:** Layer 0 — Physical & Network Implementation
**Status:** In progress (core services established, segmentation pending)

Primary focus at this stage:
- Turn architectural design into physical and logical reality
- Establish stable, observable network primitives
- Avoid premature Layer 1 (compute / VMs) until segmentation exists

---

## Internet Edge
- **Modem:** _(ISP-provided)_
- **Public IP:** Dynamic (behind ISP)

## Firewall
- **Hardware:** Protectli Vault V1410
- **Role:** Network brain
- **OS:** OPNsense
- **Position:** Modem → Firewall → Managed Switch

## Managed Switch
- **Model:** TP-Link Omada SG2210P
- **Current State:**
  - Physically connected
  - VLANs configured and enforced
  - Acting as VLAN-aware L2 switch

---

## IP Addressing

In enterprise environments, utilizing 10.x.x.x is more common due to the larger address space, easier segmentation (VLANs) for example (10.10.x.0/24) and overall clearer mental mapping.

### Understanding Subnets
Start with an IP range. For simplicity, 192.168.x.0/24. The /24 is the subnet and determines how the predefined number of IPs are split. It is about size and not position. IPv4 has four octets that represent the IP address which under the hood are in binary and are 32 bits total. When you say /24 (which is a standard), you are saying that 24 bits are fixed and that the remaining 8 can be changed. So this defines the size. 8 bits represents one of the octets so in this case you would be changing 192.168.1.**x**

However, when you change /24 by one number, this mental idea starts to change because when you add one bit by saying /25, you end up creating two subnets which are seen as two separate networks. This is because your range is still 0-255 IPs however /25 means that 25 of the bits are fixed meaning you half your IPs into two networks: 0-127 and 128-255. Both networks have to have their own router at .0 and in the other network at .128 and they both need to have their own IP dedicated to broadcast as well. They are NOT on the same network.

### IP Pool Convention

| IP Pool    | Enterprise Standard Use                     |
| ---------- | ------------------------------------------- |
| .1         | Router                                      |
| .2 - .20   | **Infrastructure** (firewall, NAS, Servers) |
| .21 - .40  | Reserved / future **statics**               |
| .41 - .245 | DHCP Pool (Dynamic)                         |

---

## Switch Port Map

Switch: TP-Link Omada SG2210P

Checked against the switch's 802.1Q *VLAN Config* and *Port Config* pages on 2026-11-05.

| Port | VLANs | PVID | Ingress check | Frames | Device | Notes |
| ---- | ----- | ---- | ------------- | ------ | ------ | ----- |
| 1 | Tagged 10–50 | 10 | on | Tagged only | OPNsense firewall | Trunk uplink, not in VLAN1 |
| 2 | 20 (+ VLAN1) | 20 | on | Admit all | Main PC | Workstation |
| 3 | 20 (+ VLAN1) | 20 | on | Admit all | Docking station | Intermittent laptop |
| 4 | 30 (+ VLAN1) | 30 | on | Admit all | **Unmanaged switch** → Vintage Story server + MuttMetrics server | Both boxes share one L2 segment (see below) |
| 5 | 40 (+ VLAN1) | 40 | on | Admit all | — | Prepared access port for the CyberLab |
| 6 | 50 (+ VLAN1) | 50 | on | Admit all | — | Prepared access port for Guest |
| 7 | Tagged 10–50 | 10 | on | Tagged only | Proxmox host | Trunk, not in VLAN1 ([Proxmox](../layer-1-compute/Proxmox.md)) |
| 8 | 10 (+ VLAN1) | 10 | **off** | Admit all | — | Management access port: turn ingress checking on |
| SFP 9–10 | VLAN1 | 1 | on | Admit all | Unused | |

### VLAN1 is not as harmless as it looks (open, 2026-11-05)

VLAN1 ("System-VLAN") still has ports 2–6 and 8–10 as members. No trunk carries it, so it never reaches OPNsense, which is why it looked harmless ([Proxmox](../layer-1-compute/Proxmox.md) §VLAN1). The catch is that the access ports **Admit All** frames, and ingress checking only drops VLANs a port *isn't* a member of. So a device on port 4 can send frames **tagged VLAN 1** and they're accepted. The switch then delivers them, untagged, out of every other VLAN1 port, including the main PC on port 2. That's a path from SERVICES to TRUSTED that **never touches the firewall**.

Fix, one port at a time, port 4 first:
1. Remove the access ports (2–6, 8, and the unused SFPs) from VLAN1's membership, as was already done for ports 1 and 7. Their PVIDs aren't 1, and switch management lives on VLAN 10, so nothing should drop. Keep port 8 as the recovery path while doing it.
2. Turn ingress checking on for port 8.
3. Test before and after: from a box on port 4, create a VLAN-1 subinterface (`ip link add link <nic> name <nic>.1 type vlan id 1`), give it an address in the TRUSTED range, and try to reach the main PC. Before the fix it should answer; after, it shouldn't.

Untagged frames on an access port always land in the PVID's VLAN. To know where a device ends up, read the *Port Config* page; the membership list doesn't tell you.

### Port 4: two servers behind one unmanaged switch (2026-11)

An unmanaged switch on port 4 lets the Vintage Story server and the MuttMetrics server share one cable run. Both end up in VLAN 30, because the unmanaged switch passes untagged frames and port 4 assigns them to VLAN 30.

**What this means:** traffic between those two boxes is switched inside the unmanaged switch and **never reaches OPNsense**, so no firewall rule can separate them. Isolation between them has to come from each host's own firewall. The MuttMetrics server runs nftables for this, because it holds client data ([MuttMetrics-Server](../services/MuttMetrics-Server.md)). If that ever stops being enough, the fix is to give one of the boxes its own managed-switch port (or its own VLAN).

---

## End Devices (Current)
- Personal PC (wired, TRUSTED)
- Laptop docking station (wired, TRUSTED)
- Proxmox host, an OptiPlex (trunk, management on MGMT, containers on SERVICES)
- Vintage Story game server (wired, SERVICES, via the port 4 unmanaged switch)
- MuttMetrics server, a refurbished OptiPlex running Debian (wired, SERVICES, via the port 4 unmanaged switch)
- No Wi-Fi AP behind firewall yet (intentionally deferred)

> Note: There are currently **two separate networks** in the house:
> - Household Wi-Fi
> - Personal wired homelab network

---

## Security Baseline

### Inbound
- Default deny (no exposed services)

### Outbound
- Default allow (acknowledged risk)
- Logging available
- Future tightening planned after segmentation

### Awareness
- Outbound beaconing risk understood
- DNS telemetry viewed as early detection signal
- IDS/IPS intentionally deferred to later layer

---

## Known Limitations (Intentional)
- No Wi-Fi behind firewall
- No IDS/IPS
- No egress VPN
- No SIEM ingestion

These are **intentional deferrals**, not oversights.
