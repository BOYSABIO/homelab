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

### VLAN Firewall Rules & Design Rationale
> Why the Layer‑0 firewall rules look the way they do, not just _what_ they are. It exists so that future‑me (or anyone reviewing this homelab) can immediately understand the intent, trade‑offs, and reasoning without re‑learning everything from scratch.

This is a **baseline, not an endpoint**. Rules are expected to evolve as services, identity, and workloads are added.

### Core Design Principles (Non‑Negotiable)
These principles guided every decision:
1. **Initiation-based trust**  
    Who is allowed to _start_ a connection matters more than who can respond.
2. **Stateful firewalling**  
    Once a connection is allowed and state is created, return traffic is implicitly allowed.
3. **Default deny everywhere**  
    Anything not explicitly required is blocked.
4. **Infrastructure dependencies are explicit**  
    DNS, NTP, and management access are never assumed.
5. **Temporary exceptions are documented**  
    If something is allowed for convenience or transition (e.g., Services → Internet), it is explicitly labeled as such.
6. **Segmentation before identity overlays**  
    VLANs and firewall rules define trust boundaries first; identity overlays (Tailscale) consume those boundaries later.

### VLAN Intent Overview
#### VLAN10 – MGMT (10.xx.xx.0/24)
**Purpose:** Control plane / administration
**Key characteristics:**
- Hosts _management interfaces_, not workloads
- Firewall UI, switch UI, future hypervisor management (Proxmox)
- Treated like an out‑of‑band management network

**Initiation allowed:**
- MGMT → Firewall (HTTPS, DNS, NTP)

**Initiation blocked:**
- MGMT → Internet (default)

**Notes:**
- Internet access is temporarily enabled only during planned maintenance windows (e.g., firmware downloads), then removed.
- Switch management lives here by design and cannot be accessed from other VLANs due to switch architecture.

#### VLAN20 – TRUSTED (10.yy.yy.0/24)
**Purpose:** User devices (PCs, laptops)
**Key characteristics:**
- Represents human‑operated endpoints
- Convenience balanced with security

**Initiation allowed:**
- TRUSTED → Internet
- TRUSTED → SERVICES (broad for now)
- TRUSTED → Firewall (DNS, NTP, UI as needed)

**Initiation blocked:**
- TRUSTED → MGMT

**Notes:**
- Per‑service port tightening is intentionally deferred until multiple services exist.

#### VLAN30 – SERVICES (10.zz.zz.0/24)
**Purpose:** Servers and hosted services
**Key characteristics:**
- Servers respond; they do not roam
- Designed to minimize blast radius if compromised

**Initiation allowed:**
- SERVICES → Firewall (DNS, NTP)
- SERVICES → Internet (**temporary**, for updates + Tailscale)

**Initiation blocked:**
- SERVICES → TRUSTED
- SERVICES → MGMT
- SERVICES → Firewall UI (HTTPS)

**Notes:**
- Internet access exists only to support updates and Tailscale.
- Planned future refinement:
    - Replace generic internet access with scoped egress (APIs, mirrors, proxies).

#### VLAN40 – CLAB (10.ww.ww.0/24)
**Purpose:** Cyber / attack lab
**Status:** Placeholder
**Policy:**
- Default deny
- No rules until devices exist

#### VLAN50 – GUEST / IOT (10.vv.vv.0/24)
**Purpose:** Untrusted or guest device
**Status:** Placeholde
**Policy:*
- Default deny
- Internet-only when implemented

### Key Learning Moments
#### VLAN firewall rules apply to _traffic entering the firewall_
- Rules on VLAN interfaces govern packets _originating from that VLAN_.
- Direction is almost always **in** for client‑initiated traffic.

#### Connections are bidirectional, initiation is not
- Servers can respond freely once a connection is established.
- Blocking SERVICES → TRUSTED does **not** break game servers or web apps.

#### Block vs Reject
- **Block** = silent drop, stealthy, preferred for sensitive zones
- **Reject** = explicit feedback, used sparingly

#### Management planes are special
- MGMT is not a convenience network.
- Blocking MGMT → Internet does **not** affect firewall or switch operation.
- Maintenance access is intentionally temporary.

#### Identity overlays do not replace segmentation
- Tailscale provides identity‑based access, not trust boundaries.
- Firewall rules remain the ultimate authority.

### Proxmox Design Decision (Forward‑Looking)
- Proxmox management **will live in VLAN10 (MGMT)**.
- Proxmox does **not** need general internet access from MGMT.
- VM and container traffic will egress via **VLAN‑tagged bridges**, not MGMT.
- Management plane isolation remains intact.

This mirrors enterprise hypervisor design:
> Management is isolated; workloads live elsewhere.
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

### VLAN Implementation & Mental Model (OPNsense + Managed Switch)

## Purpose
This document captures:
- What VLANs actually do (beyond the textbook definition)
- How VLANs are implemented across OPNsense and a managed switch
- Why VLANs are created conceptually first and only activated later
- Common pitfalls and what we initially missed
- The exact step-by-step process used in this homelab

This reflects a **real-world, enterprise-style VLAN rollout**, not a simplified lab example.

### What a VLAN Actually Is (Practical Definition)
A VLAN is a **logical Layer 2 broadcast domain**.  
Each VLAN behaves like a **separate physical switch**, even though all traffic travels over the same cables.

Key properties:
- Devices in different VLANs **cannot communicate at Layer 2**
- Inter-VLAN communication **requires a router or firewall**
- VLANs reduce broadcast traffic and enforce segmentation

A VLAN does **nothing by itself** until:
- A device is assigned to it (via PVID or tagging)
- A Layer 3 interface exists to route traffic

### The Two Places VLANs Must Exist
A VLAN must be configured in **two completely different systems**:
1. OPNsense (Layer 3: routing, firewall, services)
2. Managed Switch (Layer 2: tagging, forwarding, access)

Both must agree, or traffic silently fails.

### VLAN Design & IP Addressing Scheme
We use a simple, scalable, enterprise-style mapping:

| VLAN ID | Name       | Subnet        | Gateway    |
| ------: | ---------- | ------------- | ---------- |
|      10 | Management | 10.xx.xx.0/24 | 10.xx.xx.1 |
|      20 | Trusted    | 10.yy.yy.0/24 | 10.yy.yy.1 |
|      30 | Servers    | 10.zz.zz.0/24 | 10.zz.zz.1 |
|      40 | CyberLab   | 10.ww.ww.0/24 | 10.ww.ww.1 |
|      50 | Guest      | 10.vv.vv.0/24 | 10.vv.vv.1 |

#### Why this scheme?
- VLAN ID matches subnet (easy to reason about)
- /24 gives enough addresses without being flat
- Gateway always `.1` for consistency
- Aligns with enterprise and certification conventions

### Conceptual Setup in OPNsense (Nothing Breaks Yet)
#### Step 1: Create VLAN Device
- Parent interface: LAN (physical NIC)
- VLAN tag: e.g. 20
- This creates a **virtual interface**, not an active network

**Important:**  
At this point, *no traffic flows*. No device is using the VLAN yet.

#### Step 2: Assign and Enable the Interface
- Assign VLAN device as an interface (e.g. `VLAN20_TRUSTED`)
- Enable interface
- Assign static IP (e.g. `10.yy.yy.1/24`)

**What this does internally:**
- Creates a Layer 3 endpoint (gateway)
- Allows routing *if* traffic reaches it
- Still does not allow traffic by default (firewall default deny)

#### Step 3: Enable Services on the VLAN (Critical Step)
Services must explicitly listen on the VLAN interface:
- DHCP (Kea)
- DNS (Unbound / DNSmasq)

**What we initially missed:**
- DHCP can hand out leases
- But DNS and routing fail if services are not bound to the VLAN
- This leads to “I have an IP but nothing works”

#### Step 4: DHCP Configuration
- Subnet: `10.yy.yy.0/24`
- Pool: e.g. `10.yy.yy.100–10.yy.yy.200`
- Interface: VLAN20_TRUSTED

**What DHCP actually does:**
- Assigns IP, mask, gateway, DNS
- Proves Layer 2 VLAN tagging works
- Does NOT guarantee routing or firewall permission
- 100-200 allows space for static IPs

#### Step 5: Firewall Rules (Default Deny)
OPNsense blocks all traffic by default on new interfaces.
Initial bootstrap rule:
Allow VLAN20_TRUSTED net → any


This makes the VLAN behave **exactly like the original LAN**.

Security isolation comes later.

### Switch-Side Configuration (Layer 2)
#### VLAN Creation
- Create VLAN ID (e.g. 20)
- VLAN exists logically but has no members yet

#### Trunk Port (Firewall Uplink)
The switch port connected to OPNsense is a **trunk**.

Configuration:
- Tagged: VLANs 10, 20, 30, 40, 50
- Untagged: none (or VLAN 1 temporarily)
- Acceptable frames: Tagged only
- Ingress checking: enabled

**Purpose:**
- Carry all VLANs to the firewall
- Preserve tags for routing

#### Access Ports (End Devices)
Access ports connect to non-VLAN-aware devices.

Configuration:
- Untagged: exactly one VLAN
- PVID: VLAN ID
- Ingress checking: enabled
- Acceptable frames: Untagged only (or “Admit All” if limited UI)

### The Critical Concept: PVID Is the Activation Switch
A VLAN does **nothing** until a port’s PVID is changed.
#### What happens when PVID changes:
1. Device sends untagged Ethernet frames
2. Switch assigns those frames to the PVID VLAN
3. Frames are tagged internally
4. Traffic is forwarded only where that VLAN exists
5. Firewall receives tagged traffic and routes it

**This is the moment the VLAN becomes “real”.**

Before this:
- VLAN exists conceptually
- No device uses it

After this:
- Device is isolated from all other VLANs
- Routing and firewall rules apply

### Why Build Everything First, Then Move Devices
Best practice:
1. Create VLANs everywhere
2. Enable services
3. Prepare firewall rules
4. Test on unused ports
5. Migrate devices by changing PVID only

Benefits:
- No downtime
- No lockouts
- Easy rollback
- Matches enterprise change control

### Common Pitfalls Encountered
- VLAN exists but firewall port not tagged
- Services not bound to VLAN interface
- Firewall rules missing (default deny)
- Expecting connectivity before changing PVID
- Confusing “Management VLAN” with “Trunk Port”

Each failure helped clarify the real data path.

### Final Mental Model
- **OPNsense defines networks**
- **Switch assigns devices to networks**
- **PVID determines where untagged traffic lives**
- **Trunk ports carry tags**
- **Firewall rules control inter-VLAN communication**

VLANs are simple once you realize:
> *Nothing happens until the switch assigns a port to a VLAN.*

## VLANs — Understanding, Process, and Lessons Learned
#### Why This Document Exists
This document exists to **preserve understanding**, not just configuration.

VLANs are deceptively simple on paper but extremely easy to misconfigure in practice. The goal here is to capture:

- How VLANs *actually* work at Layer 2 and Layer 3
- The real-world **order of operations**
- The **failure modes** we encountered
- The subtle differences between “working”, “kind of working”, and “correct”
- A mental model that can be replayed later

If I ever forget how VLANs work, reading this should allow me to *relive the problem-solving process* and rebuild confidently.

### Initial Mental Model (Before Hands-On)
Initial understanding:
- VLANs “split a LAN into virtual networks”
- Devices in different VLANs are isolated
- The firewall routes between VLANs
- Managed switches “somehow handle it”

What was **missing**:
- Where VLAN decisions actually happen
- How tagging vs untagging really works
- Why things silently fail instead of erroring
- Why management behaves differently
- How DHCP/DNS interplay with VLANs
- Why order matters more than configuration itself

### The First Key Realization: Where VLANs Live
#### VLANs are decided at the **switch**, not the firewall
This was the most important conceptual unlock.

- The **switch** assigns VLANs to frames
- The **firewall** only routes *between subnets*
- VLAN tagging is **Layer 2**
- IP routing is **Layer 3**

The firewall never “assigns” VLANs — it only:
- Receives tagged frames
- Strips the tag
- Routes based on IP
- Sends frames back *tagged again*

If the switch doesn’t tag frames correctly, the firewall never even sees the traffic.

### Tagged vs Untagged — The Second Big Unlock
#### Untagged traffic
- Has **no VLAN ID**
- Must be assigned a VLAN by the switch
- Uses the **PVID** (Port VLAN ID)
#### Tagged traffic
- Explicitly carries a VLAN ID (802.1Q)
- The switch must be told whether to:
  - Accept it
  - Preserve it
  - Drop it

This leads to the core rule:
> **End devices send untagged traffic.  
> Infrastructure devices send tagged traffic.**

### Trunk Ports vs Access Ports (As Discovered, Not Memorized)
#### Access Port (PCs, servers, laptops)
What finally worked:
- Exactly **one untagged VLAN**
- PVID set to that VLAN
- No tagged VLANs allowed

What this means:
- Any frame arriving here gets assigned to the VLAN
- The device never knows VLANs exist
- Clean, predictable behavior

#### Trunk Port (Firewall, Proxmox, APs)
What finally worked:
- **Only tagged VLANs**
- No untagged/native VLAN
- Tagged-only accepted
- Ingress checking enabled

Why:
- The firewall already tags traffic
- Untagged traffic here causes ambiguity
- Native VLANs on trunks are dangerous unless intentional

This distinction explained *most* early failures.

### The “Why Is Nothing Working?” Phase
#### Symptom
- Device plugged in
- No DHCP
- No DNS
- No link detection sometimes
- No errors anywhere
#### What was actually wrong (multiple times)
- VLAN existed conceptually but wasn’t active
- PVID not changed
- Firewall service not listening on VLAN interface
- DHCP enabled but interface not selected
- DNS enabled but interface not selected
- Switch port in VLAN config but PVID still old
- Firewall rules missing (even for basic connectivity)
Lesson learned
> VLANs don’t “turn on” until **PVID is changed**.

Everything can look correct in the UI and still do nothing.

### Order of Operations (Critical for Reproducibility)
The **exact sequence** that worked reliably:
1. Create VLAN interface in OPNsense
2. Assign IP and subnet
3. Enable DHCP on that VLAN
4. Enable DNS (Unbound / Kea / dnsmasq) on that VLAN
5. Add a **temporary allow-any firewall rule**
6. Create VLAN in switch (do not move devices yet)
7. Configure VLAN membership (tagged/untagged)
8. Change PVID on the access port
9. Renew DHCP lease
10. Validate:
   - IP
   - Gateway
   - DNS
   - Internet
11. Only then move the next device

Skipping steps caused silent failure every time.

### Management VLAN — The Most Confusing (and Educational) Part
#### Observation
- Firewall UI accessible from all VLANs (with rules)
- Switch UI accessible **only from VLAN10**

At first this looked like:
- Firewall bug
- Switch bug
- VLAN misconfiguration
- ACL issue

#### Actual reason
The switch has a **management plane** that:
- Lives in exactly one VLAN
- Only responds to management traffic from that VLAN
- Is separate from normal frame forwarding

This behavior:
- Is not ingress checking
- Is not firewall related
- Is intentional and secure by default

This explained:
- Why VLAN20 could access servers in VLAN30
- But could *not* access the switch UI in VLAN10

This was a huge “aha” moment.

### VLAN1 (Legacy LAN) — The Hidden Trap
Key discovery:
- Leaving VLAN1 partially configured is dangerous
- Especially leaving it untagged on trunk ports

Final understanding:
- VLAN1 should not be part of the trunk
- VLAN1 should not be untagged on the firewall port
- VLAN1 should exist only as:
  - Transitional
  - Recovery
  - Or fully retired

Most weird behavior early on came from VLAN1 still being “kind of active”.

### Ingress Checking & Acceptable Frames — When to Harden
#### Early phase (migration)
- Ingress checking often disabled
- Accept all frames
- Goal: avoid lockouts

#### Stable phase (post-migration)
**Access ports**
- Ingress checking: enabled
- Acceptable frames: untagged only

**Trunk ports**
- Ingress checking: enabled
- Acceptable frames: tagged only

This enforces:
- No VLAN hopping
- No accidental tagging
- No ambiguity

Hardening only worked **after** understanding was solid.

### What We Learned (The Real Takeaways)
- VLANs fail silently by design
- Most issues are Layer 2, not Layer 3
- The switch is the source of truth for VLAN membership
- Firewalls don’t fix broken tagging
- Management planes are special
- Order matters more than settings
- Recovery paths are not optional
- “Working” is not the same as “correct”
- Confusion usually means mixed trust boundaries

### Reproducible Mental Model (If I Forget Everything)
If I forget VLANs entirely, remember this:
1. End devices are dumb → access ports
2. Infrastructure is smart → trunk ports
3. VLANs are applied at ingress on the switch
4. PVID decides untagged traffic
5. Firewall routes, it does not assign VLANs
6. Services must explicitly listen on VLAN interfaces
7. Management traffic is special
8. Never remove the recovery path first

If something doesn’t work:
- Check PVID
- Check tagging
- Check service bindings
- Check firewall rules
- Check which layer the failure belongs to

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

### Important Distinctions
>Tailscale is not a replacement for VLANs.  
>It is a separate identity-based control plane that can be _selectively bridged_ into VLANs via firewall rules.

>VLAN interfaces are Layer-3 gateways and need IPs.  
>Tailscale is a Layer-3 overlay managed by the daemon — OPNsense does not assign its IP.

### Remote Access (Tailscale) MVP
Enable **secure, identity-based remote administration** of the homelab firewall (OPNsense) without exposing services to the internet and **without coupling remote access to VLAN trust**.

This remote access acts as an **out-of-band control plane**, allowing Layer 0 work (segmentation, firewall policy) to continue safely from a remote location.

### Design Principles (Non‑Negotiable)
- **Control plane ≠ data plane**: Remote access is for administration, not general LAN access.
- **Identity-first**: Access is tied to a Tailscale identity, not IP location.
- **Least privilege**: Only HTTPS access to the firewall UI.
- **Defense in depth**: Tailscale ACLs + OPNsense firewall rules.
- **Reversible**: Can be fully disabled without touching VLANs.

### Conceptual Model
- Tailscale creates an **encrypted overlay network (tailnet)** parallel to VLANs.
- Remote devices **do not join VLANs**.
- Traffic from Tailscale enters OPNsense via a dedicated interface (`tailscale0`).
- Firewall rules decide **what tailnet identities may access**.
```
Remote Laptop
   ↓ (Identity + E2EE)
Tailscale Tailnet (100.x.x.x)
   ↓
OPNsense Tailscale Interface
   ↓ (explicit firewall rule)
OPNsense UI ONLY
```
### Single Tailnet Strategy
- OPNsense joins **one tailnet only** (GitHub-linked, paid).
- This tailnet represents the **infrastructure trust domain**.
- Family / friends / services will **not** share this trust domain.

Rationale:
- Tailnets are security boundaries.
- Infrastructure should belong to exactly one identity root.

### Implementation Summary
#### 1. Tailscale Plugin
- Installed `os-tailscale` plugin on OPNsense.
- Enabled service using a **pre-authentication key**.

Pre-auth key settings:
- Reusable: ✅ (for redeploy safety)
- Expiration: 7 days
- Ephemeral: ❌
- Tags: ❌

#### 2. Interface Assignment
- Assigned `tailscale0` as an OPNsense interface (`Tailscale`).
- IPv4 / IPv6 configuration: **None** (IP is managed by Tailscale daemon).

Key nuance:
> Tailscale interfaces are not Layer‑3 gateways; OPNsense does not assign their IPs.

#### 3. Firewall Rule (MVP)
**Goal:** Allow admin-only HTTPS access to the firewall itself.

Final rule (conceptual):
- Interface: `Tailscale`
- Direction: `in`
- Protocol: TCP
- Source: Explicit tailnet IPv4 range
- Destination: Firewall Tailscale IP
- Port: 443 (HTTPS)

Important nuance:
- Built-in `Tailscale net` alias did **not** reliably match traffic.
- Solution: define an explicit alias for the tailnet range.

Alias:
- Name: `TAILNET_V4`
- Type: Network(s)
- Value: `100.64.0.0/10`

This restored deterministic firewall matching.

#### 4. Validation
Positive tests:
- Remote access to `https://<OPNsense_Tailscale_IP>` works.

Negative tests:
- Cannot ping or access VLAN gateways.
- No access to switch or LAN devices.

Logs:
- Firewall logs show **pass** on Tailscale interface for TCP/443.
- No VLAN interfaces involved.

### DERP vs Direct UDP (Understanding)
- **Direct UDP**: preferred, peer‑to‑peer, lowest latency.
- **DERP relay**: encrypted relay used when direct UDP fails.

Key insight:
> DERP is **not a security downgrade** — only a performance tradeoff.
> For Layer 0 admin access, DERP is acceptable and expected on restrictive networks.

### Security Guarantees Achieved
- No inbound WAN exposure
- No port forwarding
- No VLAN trust expansion
- Identity‑scoped access
- Instant kill‑switch via Tailscale admin console

### What Is _Not_ Done Yet (Intentionally)
- No subnet routing
- No exit nodes
- No user onboarding
- No service exposure
- No MagicDNS reliance
- No ACL complexity

### Key Takeaway
> **Tailscale provides an identity‑based control plane. VLANs protect infrastructure zones. They intersect only by explicit firewall intent.**
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

