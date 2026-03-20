# Layer 0 — Tailscale & Remote Access

> Complete remote access implementation: MVP setup, ACL design, subnet routing, and exit node.
> This is the single source of truth for how remote access was built in Layer 0.

---

## 1. Remote Access Strategy (Implemented vs Planned)

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
> Tailscale is not a replacement for VLANs.
> It is a separate identity-based control plane that can be _selectively bridged_ into VLANs via firewall rules.

> VLAN interfaces are Layer-3 gateways and need IPs.
> Tailscale is a Layer-3 overlay managed by the daemon — OPNsense does not assign its IP.

### Why Tailscale (Not Traditional VPN)
Running a traditional VPN endpoint directly on the firewall introduces significant complexity:
- Requires Dynamic DNS or static public IP
- Exposes a public-facing attack surface
- Per-user authorization is difficult to manage
- Adding multiple users (admin vs family) becomes messy
- Combining inbound VPN access with outbound VPN routing significantly increases complexity
- Operational overhead grows quickly for a home environment

Architectural insight:
> Managing *identity-based access* is often more important than managing tunnels.

---

## 2. Design Principles (Non-Negotiable)
- **Control plane ≠ data plane**: Remote access is for administration, not general LAN access.
- **Identity-first**: Access is tied to a Tailscale identity, not IP location.
- **Least privilege**: Only HTTPS access to the firewall UI.
- **Defense in depth**: Tailscale ACLs + OPNsense firewall rules.
- **Reversible**: Can be fully disabled without touching VLANs.

---

## 3. Conceptual Model

Tailscale creates an **encrypted overlay network (tailnet)** parallel to VLANs.
Remote devices **do not join VLANs**.
Traffic from Tailscale enters OPNsense via a dedicated interface (`tailscale0`).
Firewall rules decide **what tailnet identities may access**.

```
Remote Laptop
   ↓ (Identity + E2EE)
Tailscale Tailnet (100.x.x.x)
   ↓
OPNsense Tailscale Interface
   ↓ (explicit firewall rule)
OPNsense UI ONLY
```

### The Two-Layer Model

| Layer | What It Controls | Where It Lives |
| ----- | ---------------- | -------------- |
| Tailscale ACLs | Which devices/users can communicate within the tailnet | Tailscale admin console |
| OPNsense Firewall Rules | What Tailscale traffic can do once it arrives at OPNsense | OPNsense > Firewall > Rules > Tailscale |

> [!important] Both layers must allow traffic. Think of it as two doors. Tailscale ACLs are the first door — they control who even gets into the building. OPNsense firewall rules are the second door — they control which rooms (VLANs, internet) those people can access once inside. If either door is closed, traffic doesn't flow.

### Single Tailnet Strategy
- OPNsense joins **one tailnet only** (GitHub-linked, paid).
- This tailnet represents the **infrastructure trust domain**.
- Family / friends / services will **not** share this trust domain.

Rationale:
- Tailnets are security boundaries.
- Infrastructure should belong to exactly one identity root.

---

## 4. Remote Access (Tailscale) MVP

Enable **secure, identity-based remote administration** of the homelab firewall (OPNsense) without exposing services to the internet and **without coupling remote access to VLAN trust**.

This remote access acts as an **out-of-band control plane**, allowing Layer 0 work (segmentation, firewall policy) to continue safely from a remote location.

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
> Tailscale interfaces are not Layer-3 gateways; OPNsense does not assign their IPs.

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
- **Direct UDP**: preferred, peer-to-peer, lowest latency.
- **DERP relay**: encrypted relay used when direct UDP fails.

Key insight:
> DERP is **not a security downgrade** — only a performance tradeoff.
> For Layer 0 admin access, DERP is acceptable and expected on restrictive networks.

### Security Guarantees Achieved
- No inbound WAN exposure
- No port forwarding
- No VLAN trust expansion
- Identity-scoped access
- Instant kill-switch via Tailscale admin console

### What Is _Not_ Done Yet (Intentionally)
- No subnet routing
- No exit nodes
- No user onboarding
- No service exposure
- No MagicDNS reliance
- No ACL complexity

### Key Takeaway
> **Tailscale provides an identity-based control plane. VLANs protect infrastructure zones. They intersect only by explicit firewall intent.**

---

## 5. Tailscale Remote Access Implementation (ACLs)

### 1. Executive Summary & Why This Matters
Consolidated from **two separate tailnets** (admin + family) into **one single tailnet** with proper ACLs.
- Admin (me) = full control over everything (conservative on MGMT for now: only OPNsense UI).
- Family / member accounts = restricted to **only SERVICES VLAN30** (10.zz.zz.0/24).

This gives:
- Secure remote access to internal services (Vintage Story, future apps)
- Strict VLAN segmentation (Tailscale ACLs + OPNsense firewall on tailscale0)
- Foundation for **full-tunnel privacy** when traveling (exit node – next step)
- All traffic will be logged/inspectable in the homelab (future IDS/IPS, Zeek, SIEM, ML models)

Every choice supports the two pillars: **cybersecurity segmentation** + **high-quality labeled telemetry** for ML.

### 2. Starting State (March 2026)
- OPNsense on Protectli Vault (post-26.1)
- Tailscale plugin (`os-tailscale`) installed and joined to **two separate tailnets** (old admin tailnet + old family tailnet)
- Only one firewall rule existed on Tailscale interface: TCP from TAILNET_V4 alias → This Firewall port 443 (admin UI access)
- No subnet routes advertised
- No ACLs beyond default "allow everything"
- VLAN30 SERVICES already existed with Vintage Story server

**Wall #1 we hit immediately:** Two tailnets = two separate trust domains → messy management, duplicate keys, hard to add family safely.
**How we overcame it:** Decided to consolidate into one paid/personal tailnet under BOYSABIO@github. Re-authenticated OPNsense with a new auth key. Old tailnets were left running temporarily for zero-downtime migration.

### 3. Step-by-Step Journey — How We Actually Built It

#### Step 3.1 — Single Tailnet Consolidation
- Created/joined the single tailnet in Tailscale admin console.
- In OPNsense: VPN → Tailscale → Settings → re-authenticated with new tailnet key.
- All my devices (fedora, iphone-14, pc-sabio) automatically appeared in one list.
- **Replication command (if needed later):** `tailscale up --authkey=tskey-... --reset`

#### Step 3.2 — Designing & Applying ACLs (The Core Security Decision)
We used the **Visual Editor** first (easier for learning), then understood the JSON behind it.

**Key learning moment:**
ACLs control "who is allowed to even attempt a connection". The actual forwarding is still enforced by OPNsense firewall rules on the tailscale0 interface. This is defense-in-depth.

**Exact steps we followed:**
1. Went to Tailscale admin → Access controls → Visual editor.
2. Deleted the default "All users and devices → All users and devices" rule.
3. Added Rule 1 (Admin full access):
   - Source: `autogroup:admin` (or `autogroup:owner`)
   - Destination: All users and devices
   - Port/protocol: All
   - Note: "Admin full access – owner only"
4. Added Rule 2 (Member restricted):
   - Source: `autogroup:member`
   - Destination: `10.zz.zz.0/24`
   - Port/protocol: All
   - Note: "Family/members → SERVICES VLAN30 only"
5. Dragged admin rule above member rule (order matters).
6. Saved → waited 10–15 seconds for propagation.

**Wall #2 we hit:** "Will 10.zz.zz.0/24 in destination actually work?"
**How we overcame it:** Realised Tailscale ACLs only say "you may send packets to these IPs". The actual routing/forwarding still happens on OPNsense (tailscale0 → vlan0.30). This was the exact moment we understood the two-layer model.

#### Step 3.3 — Subnet Routing (The Missing Piece)
- In OPNsense: VPN → Tailscale → Advertised Routes tab → added `10.zz.zz.0/24` with description "Remote access to VLAN_30".
- In Tailscale admin console: Machines → opnsense-fw → Subnets → approved the route.
- On remote Linux laptop: `sudo tailscale set --accept-routes` (needed once).

**Wall #3:** Even with ACLs correct, nothing reached VLAN30 until the route was approved.
**Lesson:** Subnet advertisement + approval is a hard prerequisite.

#### Step 3.4 — OPNsense Firewall Forwarding Rule
Added second rule on Tailscale interface:
- Action: Pass
- Interface: Tailscale
- Source: TAILNET_V4 alias
- Destination: 10.zz.zz.0/24
- Protocol/Port: Any
- Log: Enabled

Existing UI rule (port 443) left untouched.

#### Step 3.5 — Testing & Validation
- Remote admin laptop (Linux): ping + SSH to 10.30.30.x → worked.
- Attempt to VLAN20 or MGMT → blocked by ACL (correct).
- Family/test account would only reach VLAN30 (not yet added).

### 4. The Famous Logging Quirk (Wall #4 — Still Open)

**What happened:**
When pinging or SSHing from remote Tailscale device to VLAN30 server:
- Tailscale interface Live View → almost nothing new (only background WireGuard UDP from OPNsense itself).
- VLAN30_SERV interface → shows source = gateway IP 10.zz.zz.1 → server IP.

**Why this happens (deep explanation):**
OPNsense (pf) does stateful forwarding + source IP rewriting when routing from tailscale0 to vlan0.30. Successful flows are handled by the state table and often skip per-packet logging on the inbound tun interface. This is a well-known pf logging optimisation on WireGuard/Tailscale setups.

**How we confirmed it wasn't broken:**
Packet capture on tailscale0 interface showed the **original** remote Tailscale IP (100.x.x.x) arriving correctly.

**Current status:** Parked as "known quirk". Will be revisited with Zeek/Suricata mirroring (Layer 3) for full original 5-tuple visibility. No security impact — traffic is flowing and segmented correctly.

**Replication tip:** Always run a quick packet capture on tailscale0 when debugging forwarded traffic.

### 5. Current Architecture Diagram (text version)
```
Remote Device (Tailscale IP 100.x.x.x) 
↓ (encrypted) OPNsense tailscale0 interface 
↓ (ACL check + firewall rule) Routing decision → vlan0.30 (VLAN30 SERVICES) 
↓ Vintage Story server (10.30.30.x)
```

### 6. Lessons Learned & Key Takeaways (for replication & CV)
- Tailscale ACLs = identity gate, OPNsense firewall = network gate. Never rely on one alone.
- Always approve subnet routes in Tailscale console — easy to forget.
- Logging on tun interfaces is quirky — use packet capture for truth.
- Start conservative (MGMT only UI access) and expand later.
- Subnet routes + ACLs + future exit node = perfect combination for "logged & protected anywhere".

### 7. Next Step (already planned)
Enable **exit node** on OPNsense so I can toggle full-tunnel mode on my phone/laptop when traveling. This will route **all** my traffic through home WAN for centralised logging/IDS protection.

**Signed off** – 2026-03-10
This note is now the single source of truth for how remote access was built in Layer 0.

---

## 6. OPNsense Tailscale Exit Node

### 1. Overview & Purpose

The goal of this setup is to allow trusted devices on the Tailscale network (tailnet) to route their internet traffic through the OPNsense firewall. This provides:

- A single exit point for internet traffic when away from home
- Protection by future IDS/IPS rules on OPNsense for remote users
- Centralized control over who can use the exit node via Tailscale ACLs

### 2. Architecture & How It All Fits Together

#### 2.1 The Two-Layer Model
This setup has two independent layers of access control that **both** need to allow traffic for it to flow:

| Layer | What It Controls | Where It Lives |
| ----- | ---------------- | -------------- |
| Tailscale ACLs | Which devices/users can communicate with each other within the tailnet | Tailscale admin console |
| OPNsense Firewall Rules | What Tailscale traffic can do once it arrives at OPNsense (reach VLANs, reach WAN, etc.) | OPNsense > Firewall > Rules > Tailscale |

> [!important] Both layers must allow traffic. Think of it as two doors. Tailscale ACLs are the first door — they control who even gets into the building. OPNsense firewall rules are the second door — they control which rooms (VLANs, internet) those people can access once inside. If either door is closed, traffic doesn't flow.

#### 2.2 Where Tailscale Sits in the Network
Tailscale is its own separate virtual interface (`tailscale0`) on OPNsense — **it is not a VLAN**. Users who connect to the tailnet are not placed into any VLAN. They live in the Tailscale subnet (`100.x.x.x`) and stay there unless a firewall rule explicitly allows them to reach a specific VLAN or the internet.

The packet journey for exit node traffic looks like this:

```
Device (100.x.x.x)
  --> Tailscale tunnel (encrypted)
    --> OPNsense tailscale0 interface (decrypted)
      --> OPNsense firewall rules (is this allowed?)
        --> NAT (source IP rewritten to WAN IP)
          --> WAN --> Internet
```

#### 2.3 Exit Node Traffic vs Regular Tailscale Access
There is an important distinction between a user accessing a VLAN resource and a user using the exit node:

**Regular VLAN access:** Traffic has a Tailscale source IP and a known internal destination (e.g. `10.30.30.x`). OPNsense can see exactly who is going where, so precise firewall rules can be written.

**Exit node traffic:** Traffic has a Tailscale source IP but the destination is anywhere on the internet. OPNsense cannot distinguish "this is exit node traffic" from "this person is trying to reach my VLANs" — it just sees traffic from a Tailscale IP going somewhere. This is why Tailscale ACLs using `autogroup:internet` are the right tool to control who gets exit node access, not OPNsense firewall rules alone.

### 3. Components & Configuration

#### 3.1 Tailscale Settings
_VPN > Tailscale > Settings_

| Setting              | Value | Notes                                                                    |
| -------------------- | ----- | ------------------------------------------------------------------------ |
| Enabled              | ✅     | Tailscale service must be running                                        |
| Advertise Exit Node  | ✅     | Advertises OPNsense as an available exit node to the tailnet             |
| Accept DNS           | ✅     | Accepts DNS settings from Tailscale                                      |
| Use Exit Node        | None  | OPNsense itself does not use another exit node                           |
| Accept Subnet Routes | ❌     | Not needed — OPNsense is the router, not a client reaching other subnets |
| Disable SNAT         | ❌     | Keep unchecked for normal operation. See Section 5.1 for implications.   |

#### 3.2 Advertised Routes
_VPN > Tailscale > Advertised Routes_
These are the internal subnets that OPNsense advertises to the tailnet, allowing tailnet users to reach internal resources:

| Subnet | Description |
| ------ | ----------- |
| `10.yy.yy.0/24` | Remote access to VLAN20 (TRUST) |
| `10.zz.zz.0/24` | Remote access to VLAN30 (SERV) |
| `10.ww.ww.0/24` | Remote access to VLAN40 |

> [!warning] Routes must be approved in the Tailscale admin console. After advertising routes here, you must also go to the Tailscale admin console (login.tailscale.com/admin/machines), find the OPNsense device, and approve each advertised route. The same applies to the exit node itself — it must be explicitly enabled in the admin console after being advertised here.

#### 3.3 Tailscale ACL Rules
_Tailscale Admin Console > Access Controls_

| Source | Destination | Purpose |
| ------ | ----------- | ------- |
| `autogroup:admin` | `*` | Full access for admins — default rule |
| `autogroup:member` | `10.zz.zz.0/24` | Members can access VLAN30 services (game server etc.) |
| `autogroup:owner` | `10.yy.yy.0/24` | Owner can access VLAN20 trusted network remotely |
| `autogroup:owner` | `autogroup:internet` | Owner can use OPNsense as an exit node |

The ACL JSON for the exit node rule:

```json
// Allow owner exit node usage
{
    "src": ["autogroup:owner"],
    "dst": ["autogroup:internet"],
    "ip":  ["*"],
}
```

> [!tip] autogroup:internet is the correct destination for exit node access. Tailscale has a built-in destination called `autogroup:internet` specifically for controlling exit node usage. By using this instead of `*`, you explicitly grant exit node access only to specific users/groups. This is the correct layer to control who can use the exit node, rather than trying to restrict it at the OPNsense firewall level.

#### 3.4 OPNsense Firewall Rules
_Firewall > Rules > Tailscale_

| Action | Version | Source | Destination | Purpose |
| ------ | ------- | ------ | ----------- | ------- |
| Pass | IPv4+IPv6 | `TAILNET_V4` | This Firewall: 443 | Allow Tailscale admin access to OPNsense UI |
| Pass | IPv4 | `TAILNET_V4` | `VLAN30_SERV net` | Allow Tailscale members access to VLAN30 (TCP/UDP) |
| Pass | IPv4 | `TAILNET_V4` | any | Allow exit node traffic to reach the internet |

> [!note] Why "Destination: any" is safe for the exit node rule. The exit node firewall rule uses `destination: any` which sounds broad, but it is safe because Tailscale ACLs are already enforcing who gets exit node permission via `autogroup:internet`. OPNsense needs `any` as the destination because exit node traffic can go to literally any address on the internet. The two layers work together: Tailscale ACLs decide who is allowed, OPNsense firewall decides what they can reach.

#### 3.5 NAT Outbound
_Firewall > NAT > Outbound_

Mode is set to **Automatic**. The automatic rules already include Tailscale networks as a source, meaning OPNsense automatically handles NAT masquerading for Tailscale traffic going out through WAN. No manual rule is needed.

> [!info] What NAT masquerading does and why it is needed. When exit node traffic arrives from a Tailscale device (source IP: `100.x.x.x`), that IP is not routable on the internet. Before OPNsense forwards the packet out through WAN, it rewrites the source IP to its own public WAN IP. When the reply comes back from the internet, OPNsense knows to forward it back through the Tailscale tunnel to the original device. Without this, the internet would have no idea where to send replies.

### 4. Verification & Testing

#### 4.1 Confirming the Exit Node Is Set on a Client

```bash
tailscale status
```

Look for a line near the top that says:

```
# ExitNode: 100.x.x.x
```

Also useful:

```bash
tailscale exit-node list
```

This shows all available exit nodes and which one is currently selected (shown as `selected` in the Status column).

#### 4.2 Testing That Traffic Is Routing Through OPNsense

```bash
curl -4 https://ifconfig.me
```

The `-4` flag forces IPv4, which avoids IPv6 leaking and giving a misleading result. The IP returned should match the WAN IP shown in OPNsense under **Interfaces > Overview**.

> [!warning] Always test from a different network. If your test device is on the same internet connection as OPNsense (same modem/router), the public IP will appear the same whether the exit node is active or not — because both devices share the same WAN IP. Always test from a completely different network such as mobile data to get a meaningful result.

#### 4.3 Understanding the IPv6 Leak
When using the exit node, `ifconfig.me` may return an IPv6 address even when the exit node is active. This happens because:
- The Tailscale exit node on OPNsense handles IPv4 only
- IPv6 traffic bypasses the Tailscale tunnel and goes directly out from the client device
- The client's real IPv6 address is therefore visible to websites

To force an IPv4-only test:

```bash
curl -4 https://ifconfig.me
```

Or visit `https://whatismyipaddress.com` which shows IPv4 and IPv6 separately — the IPv4 address is what matters for confirming the exit node is working.

_Fixing the IPv6 leak is a future improvement — see Section 7._

### 5. Key Concepts & Lessons Learned

#### 5.1 SNAT and Traffic Attribution

By default, when Tailscale traffic reaches OPNsense and is forwarded to an internal VLAN, OPNsense rewrites the source IP to the VLAN gateway IP (e.g. `10.zz.zz.1`). This is called **SNAT (Source NAT)**.

The result is that in firewall logs, all Tailscale users accessing VLAN30 appear to come from `10.zz.zz.1` rather than their actual `100.x.x.x` Tailscale IP. This makes it impossible to tie traffic to a specific user in the logs — everyone is effectively anonymous.

**To fix this:** Enable **Disable SNAT** in VPN > Tailscale > Settings. This preserves the original Tailscale source IP all the way through to the destination, making logs attributable to specific users. The tradeoff is minimal since OPNsense handles return routing for `100.x.x.x` addresses automatically.

#### 5.2 IDS/IPS Placement for Exit Node Traffic
When IDS/IPS (Suricata) and traffic analysis tools (Zeek) are configured in the future, interface placement matters for what you can see and do:

| Interface | What You See | Best For |
| --------- | ------------ | -------- |
| Tailscale | Decrypted traffic with original `100.x.x.x` source IPs, before NAT | User attribution — knowing which user triggered something |
| WAN | All traffic leaving to the internet, source is OPNsense WAN IP | Threat protection — blocking malicious responses from the internet |

For full protection of exit node users, IDS/IPS should ideally be placed on **both** interfaces: Tailscale for attribution, WAN for threat detection on traffic coming back from the internet.

#### 5.3 Why Firewall Logs Show the VLAN Interface Instead of Tailscale
When a Tailscale user accesses a resource on VLAN30, the firewall log entry shows the VLAN30 interface with source IP `10.zz.zz.1`, not the Tailscale interface with a `100.x.x.x` IP. This is normal and expected — it is a consequence of SNAT (see Section 5.1). Disabling SNAT resolves this.

#### 5.4 Where Exit Node Traffic Appears in Firewall Logs
Exit node traffic does not show up on the Tailscale interface in live view logs the way you might expect. By the time OPNsense processes and forwards it out, the traffic appears on the **WAN interface**. To monitor exit node traffic in live view, filter by **Interface: WAN** rather than Interface: Tailscale.

### 6. Troubleshooting Reference

| Symptom | Likely Cause | Fix |
| ------- | ------------ | --- |
| `curl -4` shows same IP with exit node on and off | Testing from same network as OPNsense — both share the same WAN IP | Test from mobile data or a completely different internet connection |
| `ifconfig.me` shows IPv6 even with exit node active | IPv6 traffic bypasses Tailscale tunnel | Use `curl -4` to force IPv4 test. Full IPv6 tunneling is a future improvement. |
| Exit node selected but traffic not routing through OPNsense | Missing firewall rule allowing Tailscale traffic to WAN | Add Pass rule on Tailscale interface, source `TAILNET_V4`, destination `any` |
| Exit node not selectable on client | Exit node not approved in Tailscale admin console | Go to admin console, find OPNsense device, enable as exit node |
| VLAN30 logs show gateway IP instead of Tailscale IP | SNAT is rewriting source IP to gateway | Enable Disable SNAT in VPN > Tailscale > Settings |
| No Tailscale traffic visible in live view | Filtering by Tailscale interface — exit node traffic appears on WAN | Change live view filter to WAN interface |

### 7. Future Improvements

- [x] Enable **Disable SNAT** in Tailscale settings to improve user attribution in firewall logs
- [ ] Configure **Suricata (IDS/IPS)** on both the Tailscale and WAN interfaces for full exit node user protection
- [ ] Configure **Zeek** on the Tailscale interface for detailed traffic analysis with user attribution
- [ ] Investigate tunneling **IPv6 through the exit node** to prevent IPv6 leaks
- [ ] Consider adding exit node access for `autogroup:admin` in Tailscale ACLs if other admins need it
