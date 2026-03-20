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

| Port | Device | Notes |
| ---- | ------ | ----- |
| 1 | OPNsense firewall | LAN interface (trunk) |
| 2 | Main PC | Trusted endpoint |
| 3 | Docking station | Intermittent laptop |
| 4 | Vintage Story server | Game server |
| 5–8 | Unused | |
| SFP | Unused | |

---

## End Devices (Current)
- Personal PC (wired)
- Laptop docking station (wired)
- Vintage Story game server (wired)
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
- No Proxmox or VMs deployed
- No Wi-Fi behind firewall
- No IDS/IPS
- No egress VPN
- No SIEM ingestion

These are **intentional deferrals**, not oversights.
