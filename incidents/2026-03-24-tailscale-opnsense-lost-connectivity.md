# Tailscale + OPNsense: Troubleshooting, Findings & Configuration Guide
### Symptoms
- Laptop showed as connected to WiFi
- Browser could not load any pages
- Error: "We're having trouble finding that site"
- All internet access completely broken
### Initial Hypothesis
The first assumption was a DNS or general connectivity issue on the laptop itself. Standard diagnostics were run:

```bash
ping -c 4 8.8.8.8        # Failed — confirmed not just DNS

nslookup google.com 8.8.8.8  # Timed out

nmcli dev show | grep DNS    # Showed (home router)

sudo systemd-resolve --flush-caches  # No effect

sudo systemctl restart NetworkManager  # No effect
```

### Immediate Fix

```bash
sudo tailscale down        # Disabled Tailscale temporarily

# Confirmed internet restored

sudo tailscale up --exit-node=  # Cleared exit node setting and internet worked
```

---
## Diagnosis Walkthrough
### Step 1 — Identify Tailscale as the Problem
> "This machine is misconfigured and cannot relay traffic. Review this from the 'Edit route settings...' option in the machine's menu."

Both the **Exit Node** and **Subnets** badges were showing the misconfiguration warning on tailscale UI.

### Step 2 — Identify the Root Cause on OPNsense
Navigating to **VPN → Tailscale → Settings** revealed the **Disable SNAT** checkbox. This had been manually enabled previously in an attempt to see real source IPs in firewall logs. Unchecking it (re-enabling SNAT) immediately restored exit node functionality — confirming SNAT as the root cause of the outage.

---
## Root Cause Analysis
### Why Disabling SNAT Broke Everything

**With SNAT enabled (default, working):**

```
Laptop (100.x.x.x) → Tailscale tunnel → OPNsense

OPNsense rewrites source to WAN IP → Internet

Internet replies to WAN IP → OPNsense

OPNsense has state table entry → forwards back to laptop ✅
```

**With SNAT disabled (broken):**

```
Laptop (100.x.x.x) → Tailscale tunnel → OPNsense

OPNsense forwards with real Tailscale source IP (100.x.x.x) → Internet

Internet tries to reply to 100.x.x.x → no route exists on the internet ❌

Return traffic lost, connection hangs
```

The Tailscale CGNAT range (`100.64.0.0/10`) is **not routable on the public internet**. Without SNAT masquerading the source address, return packets have nowhere to go.

### Why SNAT Was Originally Disabled
When accessing VLAN devices remotely via Tailscale, firewall logs showed traffic like:

```
Source: 10.x.x.1 (VLAN20 gateway) → Destination: 10.x.x.x (device on VLAN20)
```

Instead of the expected:

```
Source: 100.x.x.x (Tailscale client IP) → Destination: 10.x.x.x (device on VLAN20)
```

This made it impossible to audit which Tailscale clients were accessing which VLAN resources. SNAT was disabled in an attempt to expose real source IPs — but this was the wrong solution.

---
## The SNAT Problem — Deep Dive
### The Core Conflict
| Goal                             | Requires          |
| -------------------------------- | ----------------- |
| Exit node internet access        | SNAT **enabled**  |
| Real source IPs in firewall logs | SNAT **disabled** |

These two goals are in direct conflict when using OPNsense's Tailscale plugin's built-in SNAT toggle.

### Attempts
#### Attempt 1: Static Route for CGNAT Range
**Idea:** Add a static route for `100.64.0.0/10` pointing to a Tailscale gateway so OPNsense knows how to route return traffic without SNAT.

**Steps taken:**
1. Created gateway `TAILSCALE_GW` under **System → Gateways**:
   - Interface: Tailscale (opt6)
   - IP: `100.x.x.x` (OPNsense's own Tailscale IP)
   - Far Gateway: ✅ (required — Tailscale IPs aren't on a local subnet)
   - Disable Gateway Monitoring: ✅ (prevents false down alerts)
1. Added static route: `100.64.0.0/10` → `TAILSCALE_GW`

**Result:** This helped with **subnet routing** (VLAN access) but did NOT fix exit node internet access. Return traffic from the internet still had no way back to the Tailscale client.

**Why it partially works:** For subnet routing, traffic stays within the network and the static route gives OPNsense a valid path to return traffic back through the Tailscale interface. For exit node traffic, return packets come from the public internet and have no knowledge of how to reach `100.64.x.x` addresses.

#### Attempt 2: IPv6 Firewall Rule
**Discovery via live logs:** With SNAT disabled, the firewall logs showed two distinct traffic patterns:
- IPv4 Tailscale traffic (`100.x.x.x`) → **passing** the exit node rule ✅
- IPv6 Tailscale traffic (`fd7a:xxxx:xxxx::xxxx:xxxx`) → **blocked** by state violation ❌

**Root cause:** The "Allow access to internet for exit node" firewall rule was IPv4 only. Updated to IPv4+IPv6.

**Result:** More traffic passing, but browsing still broken.

#### Attempt 3: Sloppy State Tracking
**Discovery:** State violation errors indicated asymmetric routing — return traffic arriving on unexpected interfaces.

**Fix:** Changed the exit node firewall rule's state type from `keep state` (strict) to `sloppy state`.
- **keep state:** Return traffic must arrive on the same interface it left on
- **sloppy state:** Return traffic can arrive on any interface, as long as it matches an existing connection in the state table

**Result:** Significantly more green (passing) entries in logs, but browsing still not loading.

**Security note:** Sloppy state only relaxes the interface check — connections still must match IP, port, and sequence number. Appropriate for VPN interfaces with asymmetric traffic paths. Not recommended for WAN or VLAN interfaces.

#### Attempt 4: IPv6 Reject Rule
**Discovery:** Even with sloppy state and IPv6 rule fix, the browser was still failing because:
1. Browser tries IPv6 first (modern browsers prefer IPv6)
2. IPv6 traffic gets silently dropped (state violation)
3. Browser keeps retrying IPv6, never falls back to IPv4
4. Page never loads even though IPv4 would work

**Fix attempted:** Added a **Reject** rule (not Block) for IPv6 at the top of Tailscale firewall rules.

**Block vs Reject — critical distinction:**
- **Block:** Silently drops packets. Client keeps retrying, never falls back.
- **Reject:** Sends explicit ICMP "unreachable" response. Client immediately knows to try IPv4.

**Result:** Rule was not firing as expected (ordering issue), and even with IPv4 passing through to WAN, curl was hanging — confirming return traffic asymmetry as the real remaining problem.

#### Final Diagnosis
Running `curl -4 --verbose https://google.com` confirmed:
- Traffic leaving the laptop ✅
- Passing through OPNsense Tailscale interface ✅  
- Reaching WAN and going to correct destination IP ✅
- **Return traffic not making it back to the laptop ❌**

This is the fundamental asymmetric routing problem that SNAT solves. The Tailscale plugin's SNAT implementation does more than just masquerade — it also manages the connection state necessary for return traffic to find its way back through the Tailscale tunnel.

### Conclusion on SNAT
**Re-enabling SNAT is the correct solution for exit node traffic.** The static route added handles the subnet routing case and does improve return routing for VLAN access. But for exit node internet traffic, SNAT is fundamentally necessary.

---
## 6. What was Fixed and How
### Fix 1 — Restore Exit Node Functionality
**Action:** Uncheck "Disable SNAT" in **VPN → Tailscale → Settings** → Apply
**Result:** Exit node immediately restored, internet access working on all clients.

### Fix 2 — Add Tailscale Gateway
**Location:** System → Gateways → Configuration → Add

| Field                      | Value                 |
| -------------------------- | --------------------- |
| Name                       | TAILSCALE_GW          |
| Interface                  | Tailscale (opt6)      |
| Address Family             | IPv4                  |
| IP Address                 | opnsense tailscale ip |
| Far Gateway                | ✅ Checked             |
| Disable Gateway Monitoring | ✅ Checked             |

**Why Far Gateway:** The Tailscale interface doesn't have a traditional local subnet, so OPNsense needs to be told the gateway is "far" (not directly on the interface's subnet).
**Why Disable Monitoring:** OPNsense's gateway monitoring works by pinging the gateway IP. Tailscale doesn't respond to monitoring pings in the expected way, which would cause OPNsense to mark the gateway as down and disable the route.

### Fix 3 — Add Static Route for Tailscale CGNAT
**Location:** System → Routes → Configuration → Add

| Field           | Value                 |
| --------------- | --------------------- |
| Network Address | 100.64.0.0/10         |
| Gateway         | TAILSCALE_GW          |
| Description     | Tailscale CGNAT route |

**What this does:** Tells OPNsense that the entire Tailscale address space (`100.64.0.0/10`) is reachable via the Tailscale interface. This improves return routing for subnet access traffic and means real Tailscale source IPs appear in firewall logs for VLAN access.

### Fix 4 — Firewall Rule Updates (Tailscale Interface)
**Updated:** "Allow access to internet for exit node" rule  
- Changed IP version from IPv4 to **IPv4+IPv6**
- Ensures IPv6 exit node traffic is permitted

**Reverted:** Sloppy state change  
- Changed back to `keep state` (default)
- With SNAT re-enabled, state tracking works correctly and sloppy state is not needed

---

## 7. Final Configuration State
### VPN → Tailscale → Settings

| Setting              | Value | Notes                                         |
| -------------------- | ----- | --------------------------------------------- |
| Enabled              | ✅     |                                               |
| Advertise Exit Node  | ✅     | OPNsense acts as exit node                    |
| Use Exit Node        | None  | OPNsense itself doesn't use another exit node |
| Accept Subnet Routes | ☐     | Not needed in this setup                      |
| Disable SNAT         | ☐     | **Must remain unchecked** — SNAT enabled      |

### System → Gateways
- `TAILSCALE_GW` pointing to `100.x.x.x` (opnsense) on Tailscale interface

### System → Routes
- `100.64.0.0/10` → `TAILSCALE_GW`

### Firewall → Rules → Tailscale

| Order | Action | Version   | Source     | Destination       | Description                              |
| ----- | ------ | --------- | ---------- | ----------------- | ---------------------------------------- |
| 1     | Pass   | IPv4+IPv6 | TAILNET_V4 | This Firewall:443 | Allow Tailscale admin to OPNsense UI     |
| 2     | Pass   | IPv4      | TAILNET_V4 | VLAN30_SERV net   | Allow Tailscale members access to VLAN30 |
| 3     | Pass   | IPv4+IPv6 | TAILNET_V4 | *                 | Allow access to internet for exit node   |

### Firewall → NAT → Outbound
- Mode: **Automatic**
- Auto rule includes Tailscale networks as source, NATted out WAN ✅

---
## 8. What was Learned
### Lesson 1 — Always Check Live Firewall Logs First
The live log view at **Firewall → Log Files → Live View** was the single most valuable diagnostic tool in this session. It revealed:
- Which traffic was being blocked vs allowed
- The exact source IPs (exposing the SNAT masquerading)
- The IPv6 vs IPv4 split in traffic
- State violation errors pointing to asymmetric routing

**Takeaway:** When network issues are mysterious, open the live logs and reproduce the problem while watching in real time.

### Lesson 2 — SNAT Serves Multiple Purposes
SNAT on the Tailscale interface isn't just about IP masquerading for log readability. It also:
- Ensures return traffic from the internet has a valid destination
- Maintains correct connection state in OPNsense's state table
- Enables proper NAT for exit node traffic through the WAN

Disabling it affects all three of these functions simultaneously, not just the IP masquerading.

### Lesson 3 — Block vs Reject Matters for Fallback Behavior
When a firewall **blocks** traffic silently, the client has no way to know it was rejected and will keep retrying until timeout. This causes:
- Slow fallback (if it happens at all)
- Confusing user experience (spinning, then eventual failure)
- Hard to diagnose (no explicit failure, just silence)

When a firewall **rejects** traffic, the client receives an immediate ICMP response and can fall back to an alternative (e.g., IPv4 when IPv6 is rejected) almost instantly.

**Rule of thumb:** Use Reject (not Block) when you want clients to fall back gracefully. Use Block when you want to silently discard traffic without giving the sender any information.

### Lesson 4 — IPv6 is Invisible Until It Breaks Things
Modern browsers and operating systems prefer IPv6 when available. Tailscale assigns both IPv4 (`100.x.x.x`) and IPv6 (`fd7a:...`) addresses. If IPv6 path is broken but IPv4 works, the browser will appear completely broken because it keeps trying IPv6 and never successfully falls back in time.

**Always check both IPv4 and IPv6 when diagnosing connectivity issues.**

### Lesson 5 — Asymmetric Routing and Stateful Firewalls Don't Mix Well
OPNsense (like most firewalls) is stateful — it tracks which interface a connection came in on and expects return traffic on the same interface. VPN tunnels like Tailscale often create asymmetric traffic paths where this assumption breaks down.

Solutions (in order of preference):
1. SNAT — makes OPNsense the apparent source, so return traffic always comes back correctly
2. Sloppy state — relaxes the interface check for return traffic matching
3. Static routes — ensures the kernel knows where to forward traffic, but doesn't fix stateful tracking

### Lesson 6 — Far Gateway is Required for Non-Local Interfaces
When creating a gateway for an interface that doesn't have a traditional subnet (like Tailscale, WireGuard, or other VPN interfaces), **Far Gateway must be checked**. Without it, OPNsense will refuse to use the gateway because it can't verify the gateway is on the interface's local subnet.

### Lesson 7 — Disable Gateway Monitoring for VPN Interfaces
OPNsense monitors gateways by pinging them. VPN interfaces often don't respond to ICMP pings in the expected way, causing the gateway to be incorrectly marked as down and routes to be pulled. Always disable gateway monitoring for Tailscale, WireGuard, and similar VPN gateways.

---
## 9. Key Concepts Reference
### Tailscale CGNAT Range
Tailscale uses the CGNAT range `100.64.0.0/10` for all device IPs. This range is:
- Reserved for carrier-grade NAT (RFC 6598)
- **Not routable on the public internet**
- Used by Tailscale across all devices on a tailnet
- Why exit node traffic must be SNATted before leaving to the internet

### SNAT (Source Network Address Translation)
Rewrites the source IP of outgoing packets to the router's own IP (typically the WAN IP). The router maintains a state table to map return traffic back to the original internal source. Essential for any traffic that needs to traverse the public internet from a non-routable source address.

### Stateful Firewall
A firewall that tracks the state of network connections. It knows which connections are established and only allows return traffic that matches an existing state table entry. The state table includes: source IP, destination IP, source port, destination port, interface.

### Asymmetric Routing
A situation where traffic between two endpoints takes different paths in each direction. Common with VPNs. Problematic with stateful firewalls because return traffic may arrive on a different interface than expected, causing state violations.

### Sloppy State
A less strict form of stateful tracking in OPNsense/pf that relaxes the interface check. Return traffic can arrive on any interface as long as it matches an existing connection by IP, port, and sequence number. Appropriate for VPN interfaces; not recommended for perimeter interfaces.

### Exit Node (Tailscale)
A Tailscale node that routes all internet traffic for other nodes on the tailnet. When a device sets an exit node, all its internet traffic is tunneled through Tailscale to the exit node, which then forwards it to the internet. The exit node's public IP appears as the source for all that traffic.

### Subnet Router (Tailscale)
A Tailscale node that advertises local subnets to the tailnet, making those subnets accessible from anywhere on the tailnet without installing Tailscale on every device. OPNsense in this setup advertises all VLAN subnets.
