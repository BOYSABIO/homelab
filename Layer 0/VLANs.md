# Layer 0 — VLANs

> Everything about VLANs: what they are, how they work, how they were implemented,
> and the lessons learned along the way.
>
> This document exists to **preserve understanding**, not just configuration.
>
> VLANs are deceptively simple on paper but extremely easy to misconfigure in practice. The goal here is to capture:
>
> - How VLANs *actually* work at Layer 2 and Layer 3
> - The real-world **order of operations**
> - The **failure modes** we encountered
> - The subtle differences between "working", "kind of working", and "correct"
> - A mental model that can be replayed later
>
> If I ever forget how VLANs work, reading this should allow me to *relive the problem-solving process*
> and rebuild confidently.
>
> This reflects a **real-world, enterprise-style VLAN rollout**, not a simplified lab example.

---

## 1. Network Segmentation — Current Status

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

---

## 2. What a VLAN Actually Is

A VLAN is a **logical Layer 2 broadcast domain**.
Each VLAN behaves like a **separate physical switch**, even though all traffic travels over the same cables.

Key properties:
- Devices in different VLANs **cannot communicate at Layer 2**
- Inter-VLAN communication **requires a router or firewall**
- VLANs reduce broadcast traffic and enforce segmentation

A VLAN does **nothing by itself** until:
- A device is assigned to it (via PVID or tagging)
- A Layer 3 interface exists to route traffic

---

## 3. VLAN Design & IP Addressing Scheme

Enterprise-style mapping used in this homelab:

| VLAN ID | Name       | Subnet         | Gateway     |
| ------: | ---------- | -------------- | ----------- |
|      10 | Management | 10.xx.xx.0/24  | 10.xx.xx.1  |
|      20 | Trusted    | 10.yy.yy.0/24  | 10.yy.yy.1  |
|      30 | Servers    | 10.zz.zz.0/24  | 10.zz.zz.1  |
|      40 | CyberLab   | 10.ww.ww.0/24  | 10.ww.ww.1  |
|      50 | Guest      | 10.vv.vv.0/24  | 10.vv.vv.1  |

#### Why this scheme?
- VLAN ID matches subnet (easy to reason about)
- /24 gives enough addresses without being flat
- Gateway always `.1` for consistency
- Aligns with enterprise and certification conventions

---

## 4. Initial Mental Model (Before Hands-On)

Initial understanding:
- VLANs "split a LAN into virtual networks"
- Devices in different VLANs are isolated
- The firewall routes between VLANs
- Managed switches "somehow handle it"

What was **missing**:
- Where VLAN decisions actually happen
- How tagging vs untagging really works
- Why things silently fail instead of erroring
- Why management behaves differently
- How DHCP/DNS interplay with VLANs
- Why order matters more than configuration itself

---

## 5. Where VLANs Live

A VLAN must be configured in **two completely different systems**:
1. **OPNsense** (Layer 3: routing, firewall, services)
2. **Managed Switch** (Layer 2: tagging, forwarding, access)

Both must agree, or traffic silently fails.

### The First Key Realization: VLANs are decided at the switch, not the firewall
This was the most important conceptual unlock.

- The **switch** assigns VLANs to frames
- The **firewall** only routes *between subnets*
- VLAN tagging is **Layer 2**
- IP routing is **Layer 3**

The firewall never "assigns" VLANs — it only:
- Receives tagged frames
- Strips the tag
- Routes based on IP
- Sends frames back *tagged again*

If the switch doesn't tag frames correctly, the firewall never even sees the traffic.

---

## 6. Important Distinction Between Access Ports and Trunk Ports

- Access port → VLAN assigned by switch port
- Trunk port → VLAN preserved via tags
- VLAN-aware device → VLAN assigned in software

VLANs provide Layer 2 separation.
Routing and policy enforcement happen at Layer 3 (firewall).

#### Core Principle
VLANs are assigned when traffic ENTERS a VLAN-aware device,
and preserved across trunk links using 802.1Q tags.

VLANs are NOT inferred from IP addresses or cables.

---

## 7. Tagged vs Untagged Traffic

### Untagged traffic
- Has **no VLAN ID**
- Must be assigned a VLAN by the switch
- Uses the **PVID** (Port VLAN ID)

### Tagged traffic
- Explicitly carries a VLAN ID (802.1Q)
- The switch must be told whether to:
  - Accept it
  - Preserve it
  - Drop it

The core rule:
> **End devices send untagged traffic.
> Infrastructure devices send tagged traffic.**

---

## 8. Access Ports vs Trunk Ports

### Access Ports (PCs, servers, laptops)
An access port belongs to exactly ONE VLAN.
- End devices (PCs, printers, NAS) are not VLAN-aware
- Frames arrive UNTAGGED
- The switch assigns the VLAN based on the ingress port
- If forwarding to a trunk, the switch ADDS the VLAN tag
- Exactly **one untagged VLAN**
- PVID set to that VLAN
- No tagged VLANs allowed

Access ports:
- Do not carry multiple VLANs
- Do not forward tagged frames

What this means:
- Any frame arriving here gets assigned to the VLAN
- The device never knows VLANs exist
- Clean, predictable behavior

### Trunk Ports (Firewall, Proxmox, APs)
A trunk port carries MULTIPLE VLANs simultaneously.
- Frames are TAGGED with VLAN IDs
- The switch does NOT assign VLANs on trunks
- Tags are preserved end-to-end
- Used between VLAN-aware devices
- **Only tagged VLANs**
- No untagged/native VLAN
- Tagged-only accepted
- Ingress checking enabled

Trunk ports connect:
- Switch ↔ Firewall
- Switch ↔ Proxmox
- Switch ↔ Managed AP
- Switch ↔ Switch

Why:
- The firewall already tags traffic
- Untagged traffic here causes ambiguity
- Native VLANs on trunks are dangerous unless intentional

This distinction explained *most* early failures.

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

### Why PCs Do Not Use Trunks
- PCs are not VLAN-aware by default
- They send and receive untagged frames
- VLAN membership is enforced by the switch port
- This reduces complexity and prevents misconfiguration

---

## 9. PVID — The Activation Switch

A VLAN does **nothing** until a port's PVID is changed.

### What happens when PVID changes:
1. Device sends untagged Ethernet frames
2. Switch assigns those frames to the PVID VLAN
3. Frames are tagged internally
4. Traffic is forwarded only where that VLAN exists
5. Firewall receives tagged traffic and routes it

**This is the moment the VLAN becomes "real."**

Before this:
- VLAN exists conceptually
- No device uses it

After this:
- Device is isolated from all other VLANs
- Routing and firewall rules apply

---

## 10. Conceptual Setup in OPNsense (Nothing Breaks Yet)

### Step 1: Create VLAN Device
- Parent interface: LAN (physical NIC)
- VLAN tag: e.g. 20
- This creates a **virtual interface**, not an active network

**Important:**
At this point, *no traffic flows*. No device is using the VLAN yet.

### Step 2: Assign and Enable the Interface
- Assign VLAN device as an interface (e.g. `VLAN20_TRUSTED`)
- Enable interface
- Assign static IP (e.g. `10.yy.yy.1/24`)

**What this does internally:**
- Creates a Layer 3 endpoint (gateway)
- Allows routing *if* traffic reaches it
- Still does not allow traffic by default (firewall default deny)

### Step 3: Enable Services on the VLAN (Critical Step)
Services must explicitly listen on the VLAN interface:
- DHCP (Kea)
- DNS (Unbound / DNSmasq)

**What we initially missed:**
- DHCP can hand out leases
- But DNS and routing fail if services are not bound to the VLAN
- This leads to "I have an IP but nothing works"

### Step 4: DHCP Configuration
- Subnet: `10.yy.yy.0/24`
- Pool: e.g. `10.yy.yy.100–10.yy.yy.200`
- Interface: VLAN20_TRUSTED

**What DHCP actually does:**
- Assigns IP, mask, gateway, DNS
- Proves Layer 2 VLAN tagging works
- Does NOT guarantee routing or firewall permission
- 100-200 allows space for static IPs

### Step 5: Firewall Rules (Default Deny)
OPNsense blocks all traffic by default on new interfaces.
Initial bootstrap rule:
Allow VLAN20_TRUSTED net → any

This makes the VLAN behave **exactly like the original LAN**.

Security isolation comes later.

---

## 11. Switch-Side Configuration (Layer 2)

### VLAN Creation
- Create VLAN ID (e.g. 20)
- VLAN exists logically but has no members yet

### Trunk Port (Firewall Uplink)
The switch port connected to OPNsense is a **trunk**.

Configuration:
- Tagged: VLANs 10, 20, 30, 40, 50
- Untagged: none (or VLAN 1 temporarily)
- Acceptable frames: Tagged only
- Ingress checking: enabled

**Purpose:**
- Carry all VLANs to the firewall
- Preserve tags for routing

### Access Ports (End Devices)
Access ports connect to non-VLAN-aware devices.

Configuration:
- Untagged: exactly one VLAN
- PVID: VLAN ID
- Ingress checking: enabled
- Acceptable frames: Untagged only (or "Admit All" if limited UI)

---

## 12. The "Why Is Nothing Working?" Phase

This phase happened multiple times during implementation and was the most educational part.

### Symptom
- Device plugged in
- No DHCP
- No DNS
- No link detection sometimes
- No errors anywhere

### What was actually wrong (multiple times)
- VLAN existed conceptually but wasn't active
- PVID not changed
- Firewall service not listening on VLAN interface
- DHCP enabled but interface not selected
- DNS enabled but interface not selected
- Switch port in VLAN config but PVID still old
- Firewall rules missing (even for basic connectivity)

Lesson learned
> VLANs don't "turn on" until **PVID is changed**.

Everything can look correct in the UI and still do nothing.

---

## 13. Order of Operations (Critical for Reproducibility)

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

---

## 14. Ingress Checking & Acceptable Frames — When to Harden

### Early phase (migration)
- Ingress checking often disabled
- Accept all frames
- Goal: avoid lockouts

### Stable phase (post-migration)

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

---

## 15. Management VLAN — The Most Confusing (and Educational) Part

### Observation
- Firewall UI accessible from all VLANs (with rules)
- Switch UI accessible **only from VLAN10**

At first this looked like:
- Firewall bug
- Switch bug
- VLAN misconfiguration
- ACL issue

### Actual reason
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

This was a huge "aha" moment.

---

## 16. VLAN1 (Legacy LAN) — The Hidden Trap

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

Most weird behavior early on came from VLAN1 still being "kind of active".

---

## 17. Proxmox VLAN Behavior
- Proxmox NIC is connected via a TRUNK
- Proxmox assigns VLANs to bridges or VM interfaces
- VM traffic is tagged before leaving the host
- Switch preserves tags, no guessing involved

---

## 18. Common Pitfalls Encountered

- VLAN exists but firewall port not tagged
- Services not bound to VLAN interface
- Firewall rules missing (default deny)
- Expecting connectivity before changing PVID
- Confusing "Management VLAN" with "Trunk Port"
- VLAN1 still partially active
- PVID not changed even though VLAN membership was correct
- DHCP enabled but interface not selected
- DNS enabled but interface not selected

Each failure helped clarify the real data path.

---

## 19. What We Learned (The Real Takeaways)

- VLANs fail silently by design
- Most issues are Layer 2, not Layer 3
- The switch is the source of truth for VLAN membership
- Firewalls don't fix broken tagging
- Management planes are special
- Order matters more than settings
- Recovery paths are not optional
- "Working" is not the same as "correct"
- Confusion usually means mixed trust boundaries

---

## 20. Reproducible Mental Model (If I Forget Everything)

If I forget VLANs entirely, remember this:
1. End devices are dumb → access ports
2. Infrastructure is smart → trunk ports
3. VLANs are applied at ingress on the switch
4. PVID decides untagged traffic
5. Firewall routes, it does not assign VLANs
6. Services must explicitly listen on VLAN interfaces
7. Management traffic is special
8. Never remove the recovery path first

If something doesn't work:
- Check PVID
- Check tagging
- Check service bindings
- Check firewall rules
- Check which layer the failure belongs to

### Final Mental Model
- **OPNsense defines networks**
- **Switch assigns devices to networks**
- **PVID determines where untagged traffic lives**
- **Trunk ports carry tags**
- **Firewall rules control inter-VLAN communication**

VLANs are simple once you realize:
> *Nothing happens until the switch assigns a port to a VLAN.*
