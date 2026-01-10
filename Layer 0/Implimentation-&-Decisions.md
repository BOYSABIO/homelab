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

### Logs Cheat Sheet
Unbound DNS logs show how name resolution requests are processed by the firewall.
Understanding these entries helps distinguish normal behavior from misconfiguration or attacks.
#### Common Log Fields

| Field       | Meaning                                                               |
| ----------- | --------------------------------------------------------------------- |
| Client IP   | The device that sent the DNS query (e.g. 192.168.0.x)                 |
| Domain name | The fully qualified domain name being queried (often ends with a dot) |
| Record Type | The type of DNS record requested (A, AAAA, PTR, etc.)                 |
| Class (IN)  | The DNS class — almost always `IN` (Internet)                         |

#### DNS Record Types You Will Commonly See

| Record | Name                            | Explanation                                                               |
| ------ | ------------------------------- | ------------------------------------------------------------------------- |
| A      | IPv4 Address Record             | Maps a hostname to an IPv4 address (e.g. www.example.com → 93.184.216.34) |
| AAAA   | IPv6 Address Record             | Maps a hostname to an IPv6 address                                        |
| PTR    | Pointer Record (Reverse Lookup) | Maps an IP address back to a hostname (used for reverse DNS)              |
| NS     | Name Server Record              | Indicates which DNS servers are authoritative for a domain                |
| SOA    | Start of Authority              | Contains administrative information about a DNS zone                      |
| TXT    | Text Record                     | Carries arbitrary text (used for SPF, DKIM, verification, etc.)           |
| SRV    | Service Record                  | Used to locate services (e.g. LDAP, SIP, Kerberos)                        |

#### DNS Class: IN

| Value | Meaning                                       |
| ----- | --------------------------------------------- |
| IN    | Internet (default and standard class for DNS) |

Other classes exist historically but are rarely used today.
In modern networks, DNS logs will almost always show `IN`.

#### Fully Qualified Domain Names (FQDN)
Domains in logs often end with a dot:

Example: www.bing.com.

The trailing dot means:
> “This name is absolute and fully qualified, starting at the DNS root.”

It is normal and expected in DNS logs.

#### What Indicates Normal DNS Recursion
Recursion is working correctly when:
- Clients query the firewall
- The firewall resolves domains it does not host locally
- Responses are returned without SERVFAIL errors

Seeing queries for public domains (bing.com, cloudflare.com, omada, etc.)
indicates recursive resolution is functioning.

#### Red Flags to Watch For

| Pattern                         | Meaning                                |
| ------------------------------- | -------------------------------------- |
| SERVFAIL spikes                 | Upstream resolution problems           |
| Large volumes of random domains | Possible malware or misbehaving client |
| External DNS servers contacted  | Potential DNS bypass or leak           |
| Rebind protection blocks        | Possible DNS rebinding attack          |

---

## 5. DHCP — Current & Planned State

### Current DHCP Service
- **Service:** Kea DHCP
- **Status:** Active 
- **Subnet:** 192.168.x.0/24 (current LAN)
- **Leases:** Dynamic by default with exceptions
- **Interface:** LAN

### Static Mappings
- Static DHCP mapping configured for:
  - Vintage Story server
- Verified lease transition from dynamic → static

### Domain Name
The host name + domain name helps to better identify devices and where they are located in the logs. You can also resolve to these names making it easier when connecting to devices or services. For the domain name, it is not wrong to use random names however, it is currently best practice to use `home.arpa` as it is designed for non-unique use in residential home networks. The domain is not resolvable across the internet and is intended only for use in small networks.

### DNSmasq to Kea
- **Target DHCP Service:** Kea DHCP
- **Reason for Migration:**
  - Better logging
  - Better multi-subnet / VLAN support
  - Cleaner separation of DNS and DHCP roles
  - Future automation friendliness

> Migration to Kea was planned **before VLAN rollout**
> to reduce complexity during segmentation.

### Kea DHCP Log Cheat Sheet (DHCPv4)
Kea DHCP logs describe how IP addresses are assigned, renewed, and released.
Understanding these logs allows you to verify correctness, detect conflicts,
and debug DHCP issues without guessing.

#### Core DHCP Message Types (The DHCP Handshake)

| Message      | Name                    | Explanation                                                |
| ------------ | ----------------------- | ---------------------------------------------------------- |
| DHCPDISCOVER | Client Discovery        | Client broadcasts “Is there any DHCP server?”              |
| DHCPOFFER    | Server Offer            | DHCP server proposes an IP address                         |
| DHCPREQUEST  | Client Request          | Client requests the offered IP                             |
| DHCPACK      | Server Acknowledgment   | Server confirms and assigns the IP                         |
| DHCPRELEASE  | Client Release          | Client voluntarily releases its lease                      |
| DHCPNAK      | Negative Acknowledgment | Server rejects the request (misconfig, wrong subnet, etc.) |
This four-step exchange is known as **DORA**:
DISCOVER → OFFER → REQUEST → ACK

#### Common Log Fields

| Field       | Meaning                                    |
| ----------- | ------------------------------------------ |
| hwtype      | Hardware type (1 = Ethernet)               |
| MAC address | Physical address of the client             |
| cid         | Client Identifier (may differ from MAC)    |
| tid         | Transaction ID (unique per DHCP exchange)  |
| interface   | Interface handling the request (e.g. igc1) |
| lease       | IP address being offered or assigned       |

#### Lease Lifecycle Events

| Log Entry         | Meaning                        |
| ----------------- | ------------------------------ |
| DHCP4_LEASE_OFFER | IP address proposed to client  |
| DHCP4_LEASE_ALLOC | IP address officially assigned |
| DHCPRELEASE       | Client gave back its IP        |
| valid-lifetime    | Lease duration (in seconds)    |

#### Static Reservations (Very Important)
When a client has a reservation, Kea logs will show:
- The reserved IP being offered
- Assignment **outside** the dynamic pool
- The reservation overriding pool logic

This proves the reservation system is working.

#### Example: Dynamic Client (Pool Assignment)
```
DHCPDISCOVER  
DHCPOFFER lease 192.168.0.x
DHCPREQUEST  
DHCPACK lease allocated
```
Meaning:
- Client had no reservation
- IP came from the pool
- Lease assigned normally

#### Example: Reserved Host (Static Mapping)
```
DHCPDISCOVER  
DHCPOFFER lease 192.168.x.20  
DHCPREQUEST  
DHCPACK lease allocated
```
Meaning:
- MAC matched a reservation
- IP assigned outside pool
- Reservation took priority

#### DHCPRELEASE Explained
```
DHCPRELEASE received from 192.168.x.20
```
Meaning:
- Client shut down cleanly
- Lease returned early
- IP becomes reusable (unless reserved)

This is normal behavior for servers and VMs.

#### What Normal Looks Like
Normal DHCP noise includes:
- Frequent DISCOVER / REQUEST messages
- Clients renewing leases
- Infrastructure devices asking occasionally
- DHCPRELEASE on shutdown or reboot

A quiet DHCP log is **not** expected.

#### Red Flags to Watch For

| Pattern                         | Meaning                                     |
| ------------------------------- | ------------------------------------------- |
| Repeated DISCOVER with no OFFER | DHCP server unreachable                     |
| Frequent DHCPNAK                | Wrong subnet or conflicting servers         |
| Same IP offered repeatedly      | Pool exhaustion or conflict                 |
| No DHCPRELEASE ever             | Client crashes or power loss (not critical) |

---

## 6. Network Segmentation — Current Status

### VLANs
- **Status:** Designed but not implemented
- **Reason:** Waiting for:
  - Switch configuration
  - Trunk/access port setup
  - Kea DHCP migration

### Physical Switch Port Map (Current)
Switch: TP-Link Omada SG2210P

Port 1 → OPNsense firewall (LAN interface)
Port 2 → Main PC (Trusted endpoint)
Port 3 → Docking station (intermittent laptop)
Port 4 → Vintage Story server
Port 5–8 → Unused
SFP ports → Unused

### Current Reality
- Network is flat
- Segmentation enforced only by physical separation
- Firewall rules currently apply to single LAN interface

### Important Distinction Between Access Ports and Trunk Ports
- Access port → VLAN assigned by switch port
- Trunk port → VLAN preserved via tags
- VLAN-aware device → VLAN assigned in software

VLANs provide Layer 2 separation.
Routing and policy enforcement happen at Layer 3 (firewall).
#### Core Principle
VLANs are assigned when traffic ENTERS a VLAN-aware device,  
and preserved across trunk links using 802.1Q tags.

VLANs are NOT inferred from IP addresses or cables.

### Access Ports
An access port belongs to exactly ONE VLAN.
- End devices (PCs, printers, NAS) are not VLAN-aware
- Frames arrive UNTAGGED
- The switch assigns the VLAN based on the ingress port
- If forwarding to a trunk, the switch ADDS the VLAN tag

Access ports:
- Do not carry multiple VLANs
- Do not forward tagged frames

### Trunk Ports
A trunk port carries MULTIPLE VLANs simultaneously.
- Frames are TAGGED with VLAN IDs
- The switch does NOT assign VLANs on trunks
- Tags are preserved end-to-end
- Used between VLAN-aware devices

Trunk ports connect:
- Switch ↔ Firewall
- Switch ↔ Proxmox
- Switch ↔ Managed AP
- Switch ↔ Switch

### VLAN-Aware Devices
Some devices understand VLAN tags and assign VLANs in software.
Examples:
- Firewalls (OPNsense)
- Hypervisors (Proxmox)
- Managed Access Points

These devices:
- Receive tagged frames
- Strip tags internally
- Assign traffic to logical interfaces or bridges
- Re-tag traffic when sending it back out

### Proxmox VLAN Behavior
- Proxmox NIC is connected via a TRUNK
- Proxmox assigns VLANs to bridges or VM interfaces
- VM traffic is tagged before leaving the host
- Switch preserves tags, no guessing involved

### Why PCs Do Not Use Trunks
- PCs are not VLAN-aware by default
- They send and receive untagged frames
- VLAN membership is enforced by the switch port
- This reduces complexity and prevents misconfiguration

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

