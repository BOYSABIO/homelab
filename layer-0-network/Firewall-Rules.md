# Layer 0 — Firewall Rules & Design Rationale

> Why the Layer 0 firewall rules look the way they do, not just _what_ they are.
> This exists so that future-me (or anyone reviewing this homelab) can immediately understand
> the intent, trade-offs, and reasoning without re-learning everything from scratch.

This is a **baseline, not an endpoint**. Rules are expected to evolve as services, identity, and workloads are added.

---

## OPNsense — Current State

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

---

## Core Design Principles (Non-Negotiable)

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

---

## Per-VLAN Firewall Intent

### VLAN10 – MGMT (10.xx.xx.0/24)
**Purpose:** Control plane / administration
**Key characteristics:**
- Hosts _management interfaces_, not workloads
- Firewall UI, switch UI, future hypervisor management (Proxmox)
- Treated like an out-of-band management network

**Initiation allowed:**
- MGMT → Firewall (HTTPS, DNS, NTP)

**Initiation blocked:**
- MGMT → Internet (default)

**Notes:**
- Internet access is temporarily enabled only during planned maintenance windows (e.g., firmware downloads), then removed.
- Switch management lives here by design and cannot be accessed from other VLANs due to switch architecture.

---

### VLAN20 – TRUSTED (10.yy.yy.0/24)
**Purpose:** User devices (PCs, laptops)
**Key characteristics:**
- Represents human-operated endpoints
- Convenience balanced with security

**Initiation allowed:**
- TRUSTED → Internet
- TRUSTED → SERVICES (broad for now)
- TRUSTED → Firewall (DNS, NTP, UI as needed)

**Initiation blocked:**
- TRUSTED → MGMT

**Notes:**
- Per-service port tightening is intentionally deferred until multiple services exist.

---

### VLAN30 – SERVICES (10.zz.zz.0/24)
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

---

### VLAN40 – CLAB (10.ww.ww.0/24)
**Purpose:** Cyber / attack lab
**Status:** Placeholder
**Policy:**
- Default deny
- No rules until devices exist

---

### VLAN50 – GUEST / IOT (10.vv.vv.0/24)
**Purpose:** Untrusted or guest devices
**Status:** Placeholder
**Policy:**
- Default deny
- Internet-only when implemented

---

## Key Learning Moments

### VLAN firewall rules apply to _traffic entering the firewall_
- Rules on VLAN interfaces govern packets _originating from that VLAN_.
- Direction is almost always **in** for client-initiated traffic.

### Connections are bidirectional, initiation is not
- Servers can respond freely once a connection is established.
- Blocking SERVICES → TRUSTED does **not** break game servers or web apps.

### Block vs Reject
- **Block** = silent drop, stealthy, preferred for sensitive zones
- **Reject** = explicit feedback, used sparingly

### Management planes are special
- MGMT is not a convenience network.
- Blocking MGMT → Internet does **not** affect firewall or switch operation.
- Maintenance access is intentionally temporary.

### Identity overlays do not replace segmentation
- Tailscale provides identity-based access, not trust boundaries.
- Firewall rules remain the ultimate authority.

---

## Proxmox Design Decision (Forward-Looking)
- Proxmox management **will live in VLAN10 (MGMT)**.
- Proxmox does **not** need general internet access from MGMT.
- VM and container traffic will egress via **VLAN-tagged bridges**, not MGMT.
- Management plane isolation remains intact.

This mirrors enterprise hypervisor design:
> Management is isolated; workloads live elsewhere.
