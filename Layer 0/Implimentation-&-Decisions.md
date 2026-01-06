# Layer 0 — Implementation & Decisions

> This document captures the **current implemented state** of Layer 0 of the homelab,
> including concrete configurations, decisions made, and rationale.
>
> It is a living document and represents **what actually exists today**, not just intent.
>  
> The companion document **“Layer 0 — Network & Physical Foundation (Blueprint)”**
> describes the target architecture and long-term vision.

---

## 1. Current Phase

**Phase:** Layer 0 — Physical & Network Implementation  
**Status:** In progress (core services established, segmentation pending)

Primary focus at this stage:
- Turn architectural design into physical and logical reality
- Establish stable, observable network primitives
- Avoid premature Layer 1 (compute / VMs) until segmentation exists

---

## 2. Physical Topology (As Implemented)

### Internet Edge
- **Modem:** _(ISP-provided)_
- **Public IP:** Dynamic (behind ISP)

### Firewall
- **Hardware:** Protectli Vault V1410
- **Role:** Network brain
- **OS:** OPNsense
- **Position:** Modem → Firewall → Managed Switch

### IP Ranges
In enterprise environments, utilizing 10.x.x.x is more common due to the larger address space, easier segmentation (VLANs) for example (10.10.x.0/24) and overall clearer mental mapping. This would be something to incorporate in the future of the lab with segmentation. 

Important understanding (subnets). Start with an IP range. For simplicity, 192.168.x.0/24. The /24 is the subnet and determines how the predefined number of IP's are split. It is about size and not position. IPv4 has four octets that represent the IP address which under the hood are in binary and are 32 bits total. When you say /24 (which is a standard), you are saying that 24 bits are fixed and that the remain 8 can be changed. So this defines the size. 8 bits represents one of the octets so in this case you would be changing 192.168.1.**x** However, when you change /24 by one number, this mental idea starts to change because when you add one bit by saying /25, you end up creating two subnets which are seen as two separate networks. This is because your range is still 0-255 IPs however /25 means that 25 of the bits are fixed meaning you half your IPs into two networks: 0-127 and 128-255. Both networks have to have their own router at .0 and in the other network at .128 and they both need to have their own IP dedicated to broadcast as well. They are NOT on the same network. 

| IP Pool    | Enterprise Standard Use                     |
| ---------- | ------------------------------------------- |
| .1         | Router                                      |
| .2 - .20   | **Infrastructure** (firewall, NAS, Servers) |
| .21 - .40  | Reserved / future **statics**               |
| .41 - .245 | DHCP Pool (Dynamic)                         |

### Switching
- **Managed Switch:** _(model to be confirmed / filled in)_
- **Current State:**
  - Physically connected
  - VLANs not yet enforced
  - Acting as a flat L2 switch for now

### End Devices (Current)
- Personal PC (wired)
- Laptop docking station (wired)
- Vintage Story game server (wired)
- No Wi-Fi AP behind firewall yet (intentionally deferred)

> Note: There are currently **two separate networks** in the house:
> - Household Wi-Fi
> - Personal wired homelab network

---

## 3. Firewall (OPNsense) — Current State

### Core Responsibilities
- WAN ↔ LAN routing
- DHCP
- DNS
- Firewall policy enforcement
- Traffic logging
- Future: VLAN routing, IDS/IPS, VPN egress

### Interfaces
- **WAN:** Connected to modem
- **LAN:** Connected to switch (future VLAN trunk)

(No VLAN interfaces created yet)

---

## 4. DNS — Implementation Details

### Service
- **DNS Resolver:** Unbound
- **Mode:** Recursive
- **DNSSEC:** Enabled
- **Query Logging:** Enabled

### Behavior
- Clients use OPNsense as DNS server
- Unbound performs full recursive resolution:
  - Root → TLD → Authoritative
  - Recursion essentially means OPNsense will not only resolve DNS lookups locally but will also store all the logs to maintain control and future data that can be used for ML / AI analysis. **DNS is a crucial area for SOC**
- No third-party DNS forwarders configured

### Rationale
- Maximum privacy (no central DNS observer)
- Full visibility into DNS telemetry
- Better foundation for future detection & ML work
- Avoids trust dependency on public resolvers

### Observability
- Queries visible in Unbound logs
- Client IPs visible
- Domain patterns observable

---

## 5. DHCP — Current & Planned State

### Current DHCP Service
- **Service:** dnsmasq
- **Status:** Active (temporary)
- **Subnet:** 192.168.x.0/24 (current LAN)
- **Leases:** Dynamic by default
- **Interface:** LAN

### Static Mappings
- Static DHCP mapping configured for:
  - Vintage Story server
- Verified lease transition from dynamic → static

### Domain Name
The host name + domain name helps to better identify devices and where they are located in the logs. You can also resolve to these names making it easier when connecting to devices or services. For the domain name, it is not wrong to use random names however, it is currently best practice to use `home.arpa` as it is designed for non-unique use in residential home networks. The domain is not resolvable across the internet and is intended only for use in small networks.

### Planned Change
- **Target DHCP Service:** Kea DHCP
- **Reason for Migration:**
  - Better logging
  - Better multi-subnet / VLAN support
  - Cleaner separation of DNS and DHCP roles
  - Future automation friendliness

> Migration to Kea is planned **before VLAN rollout**
> to reduce complexity during segmentation.

---

## 6. Network Segmentation — Current Status

### VLANs
- **Status:** Designed but not implemented
- **Reason:** Waiting for:
  - Switch configuration
  - Trunk/access port setup
  - Kea DHCP migration

### Current Reality
- Network is flat
- Segmentation enforced only by physical separation
- Firewall rules currently apply to single LAN interface

---

## 7. Remote Access Strategy (Implemented vs Planned)

### Current
- No remote access configured yet

### Planned Primary Method
- **Tailscale on OPNsense**
- Firewall acts as subnet router
- Identity-based access (ACLs)
- No exposed ports
- No Dynamic DNS
- No public VPN endpoint

### Deferred Learning
- Native WireGuard / IPsec on OPNsense
- To be explored later as a lab exercise

---

## 8. Security Baseline (Current)

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

## 9. Known Limitations (Intentional)

- No VLAN enforcement yet
- No Proxmox or VMs deployed
- No Wi-Fi behind firewall
- No IDS/IPS
- No egress VPN
- No SIEM ingestion

These are **intentional deferrals**, not oversights.

---

## 10. Immediate Next Steps (Layer 0 Continuation)

1. Finalize this implementation document
2. Migrate DHCP from dnsmasq → Kea
3. Identify and document managed switch model & capabilities
4. Map physical ports
5. Create VLAN interfaces in OPNsense
6. Implement switch VLAN configuration
7. Apply minimal firewall policy per VLAN
8. Validate segmentation

Layer 1 (compute & virtualization) begins **only after step 6 is complete**.

---

## 11. Key Learnings So Far

- Firewalls are network operating systems, not just packet filters
- DNS recursion vs forwarding has deep privacy implications
- DHCP is identity assignment, not just convenience
- Separating services increases control, observability, and safety
- Layering infrastructure prevents rework
- Understanding precedes implementation

---

## 12. Layer 0 Status Summary

**Design:** Complete  
**Core Services:** Implemented  
**Segmentation:** Pending  
**Safety:** Baseline established  
**Readiness for Layer 1:** Not yet (by design)

Layer 0 remains the active focus.

