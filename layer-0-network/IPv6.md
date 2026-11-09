# Layer 0 — IPv6 Integration

> Controlled internal IPv6 implementation using ULA addressing.

---

## Objective
Implement controlled IPv6 inside the homelab without introducing:
- Upstream dependency on ISP
- Firewall bypass risk
- Routing ambiguity
- Uncontrolled global exposure

IPv6 was implemented:
- Internal-only (ULA)
- SLAAC-based addressing
- No upstream global routing
- Default deny firewall posture preserved

Result:
Dual-stack internal network with predictable behavior and full firewall enforcement.

---

## Design Decision: Why Internal-Only ULA?

### Constraints
- ISP modem (Fritzbox) is shared
- No control over prefix delegation
- No desire to impact family network
- Avoid global IPv6 misconfiguration risk

### Chosen Strategy
Use ULA prefix:
`fd42:7a9c:3b10::/48`

Assign per VLAN:
`VLAN20 → fd42:7a9c:3b10:20::/64`

This provides:
- Unique internal IPv6 space
- No internet routing
- No NAT66 required
- No ISP dependency

Equivalent to IPv4 private space (10.x.x.x).

---

## How IPv6 Works (Conceptual Summary)

### 1. Link-Local (fe80::/10)
Every IPv6 interface auto-generates:
`fe80::xxxx`

Used for:
- Router discovery
- Neighbor discovery
- Default gateway communication

Not routable beyond local segment.

### 2. Router Advertisements (RA)
OPNsense periodically sends RA packets.
RA tells clients:
- What prefix to use
- Who the gateway is
- Whether to use SLAAC or DHCPv6

RA is the control plane of IPv6.

### 3. SLAAC (Stateless Address Autoconfiguration)
Process:
1. Client receives prefix from RA
2. Client generates interface ID
3. Combines prefix + interface ID
4. Performs Duplicate Address Detection
5. Uses address

Example:
`fd42:7a9c:3b10:20 + :6010:3a75:e103:839d`

No DHCP server required.

### 4. Temporary Addresses (Privacy Extensions)
Windows generates:
- Stable IPv6
- Temporary rotating IPv6

Temporary addresses prevent long-term tracking.

Does NOT break internal attribution.

### 5. DHCPv6 (Stateless Mode)
Used optionally for:
- DNS distribution
- Metadata

Not used for address assignment in this design.

Therefore:
No DHCPv6 leases appear.
This is expected behavior.

---

## Current VLAN20 IPv6 State

### Interface Configuration
VLAN20:
`fd42:7a9c:3b10:20::1/64`
Router Advertisements:
- Mode: Assisted
- SLAAC enabled
- DHCPv6 optional

Clients receive:
- ULA IPv6
- Temporary IPv6
- Link-local
- IPv4 (via Kea DHCPv4)

---

## Traffic Behavior

### Internal IPv6
VLAN20 → VLAN30 (if enabled later)
Allowed (firewall permitting)

### External IPv6
Client tries:
`fd42:... → 2620:xxxx`

Firewall blocks:
`Default deny / state violation`

Correct behavior.
No upstream IPv6 routing exists.

---

## Attribution Procedure (SOC Runbook)
If suspicious IPv6 appears in logs:

Example log:
`fd42:7a9c:3b10:20:6010:3a75:e103:839d → 2620:1ec:33::11`

#### Step 1 – IPv6 → MAC
OPNsense:
Interfaces → Diagnostics → NDP Table

Find:
`fd42:...839d → 18:C0:4D:05:83:E6`

#### Step 2 – MAC → Switch Port
TP-Link:
MAC Address Table

Find:
`18:C0:4D:05:83:E6 → Port 2`

#### Step 3 – Port → Device
From authoritative map:
Port 2 → Primary Windows PC
Attribution complete.
No DHCPv6 required.

---

## Security Implications

### IPv6 Is Not Less Secure
Firewall still:
- Default deny
- Stateful
- Interface-bound
- Logs per protocol

IPv6 does not bypass IPv4 rules.
Rules are per protocol.

### Privacy Extensions
Temporary IPv6:
- Rotates outbound identity
- Does not prevent internal correlation
- Increases privacy externally

Enterprise networks use similar setups.

---

## Performance Impact
Negligible.

RA:
- Very small periodic packets

SLAAC:
- Client-side operation

No upstream IPv6:
- No NAT66
- No additional routing complexity

No measurable load increase on OPNsense.

---

## Design Philosophy Alignment
This implementation follows core principles:
- Initiation-based trust
- Default deny everywhere
- Infrastructure dependencies explicit
- Segmentation before overlays
- No uncontrolled external exposure

---

## Layer 0b Status
IPv6 implementation is:
- Intentional
- Controlled
- Documented
- Not dependent on ISP
- Not impacting family network
- Not exposing global routes

Layer 0 foundation remains stable.
