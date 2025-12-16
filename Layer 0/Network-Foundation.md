# **Layer 0 — Network & Physical Foundation (Blueprint)**

### 1. Purpose of Layer 0
Layer 0 defines the **physical devices**, **network topology**, and **segmentation model** of the homelab.
It establishes the *non-negotiable foundation* that every other layer builds on.

Layer 0 answers:
- What physical hardware exists?
- How are devices connected?
- How is the network segmented?
- How is traffic allowed or denied?
- How do users access the network securely?

Layer 0 explicitly does **not** define:
- Virtual machines
- Containers
- Applications
- SIEM, ML pipelines, or services

Those begin in later layers.

---
### 2. High-Level Outcome
When Layer 0 is complete, the homelab has:
- A dedicated firewall acting as the network brain
- A managed switch enforcing physical segmentation
- Clearly defined VLANs and trust boundaries
- Secure remote access without exposing the home network
- A foundation safe enough to run vulnerable and hostile systems

---
### 3. Final Network Topology

```
Internet
   |
Modem
   |
[ Firewall (OPNsense) + Tailscale ]
   |
[ Managed Switch (VLANs) ]
   |
-----------------------------------------
| Mgmt | Trusted | Services | Lab | IoT |
-----------------------------------------
```

---
### 4. VLANs & Subnets

| VLAN | Name        | Purpose                                | Subnet        |
| ---- | ----------- | -------------------------------------- | ------------- |
| 10   | Management  | Firewall, Proxmox, Switch, NAS         | 10.xx.xx.0/24 |
| 20   | Trusted     | Personal devices (PC, laptop)          | 10.aa.bb.0/24 |
| 30   | Services    | SIEM, Git, ML, internal services       | 10.cc.dd.0/24 |
| 40   | SecurityLab | Kali, vulnerable systems, malware labs | 10.ee.ff.0/24 |
| 50   | IoT         | Untrusted smart devices                | 10.gg.hh.0/24 |
| 60   | DMZ         | Optional public-facing services        | 10.uu.uu.0/24 |

---
### 5. Traffic Boundaries (Intent)

- Trusted → Services ✅
- Trusted → SecurityLab ❌
- SecurityLab → Trusted/Mgmt ❌
- Management → All VLANs ✅
- IoT → Internet only

These boundaries are enforced at the firewall.

---
### 6. Remote Access Strategy (Key Design Decision)
#### Primary Remote Access: Tailscale
Tailscale runs directly on the firewall and acts as the **secure remote-access layer**.
- Firewall advertises internal subnets via Tailscale
- Users authenticate via identity (not shared VPN keys)
- Access control is handled via Tailscale ACLs
- No public IP exposure
- No port forwarding
- No dynamic DNS required

This allows:
- Full admin access for the owner
- Limited access for family members
- Easy revocation and auditing
- Secure access from anywhere

#### Why Not a Traditional Firewall VPN (Important Learning)
During design, it became clear that running a traditional VPN endpoint directly on the firewall introduces significant complexity:

**Challenges identified:**
- Requires Dynamic DNS or static public IP
- Exposes a public-facing attack surface
- Per-user authorization is difficult to manage
- Adding multiple users (admin vs family) becomes messy
- Combining inbound VPN access with outbound VPN routing (exit nodes, privacy VPNs) significantly increases complexity
- Operational overhead grows quickly for a home environment

This led to an important architectural insight:
> Managing *identity-based access* is often more important than managing tunnels.

#### Deferred Learning Goal
Native WireGuard / IPsec on OPNsense is **not abandoned**, but intentionally deferred:
- It will be explored later as a **learning lab**
- Not used as the primary production access mechanism

This keeps Layer 0 simple, secure, and operationally sane.

---
### 7. Physical Hardware Roles (Layer 0 Scope Only)
#### Firewall (OPNsense)
- Network brain
- VLAN routing
- Firewall rules
- DHCP & DNS
- Traffic logging
- Tailscale subnet router

#### Managed Switch
- VLAN enforcement
- Access and trunk ports
- SPAN/mirror ports for traffic capture

#### Proxmox Host (Existence Only)
- Physical hypervisor
- Connected via trunk port
- Lives in Management VLAN
- Hosts workloads in later layers

#### NAS
- Centralized storage
- Logs, PCAPs, datasets, backups
- Connected to Management VLAN

---

### 8. Milestones for Layer 0
#### Milestone 1 — Logical Design
- VLANs defined
- Subnets defined
- Traffic intent documented

#### Milestone 2 — Hardware in Place
- Firewall installed
- Switch configured
- VLANs enforced physically

#### Milestone 3 — Secure Access
- Tailscale running on firewall
- Subnet routing enabled
- ACLs defined for admin vs family

#### Milestone 4 — Safety Validation
- SecurityLab cannot reach Trusted or Mgmt
- Trusted can reach Services
- Management access restricted

When all milestones are met → **Layer 0 is complete**

---

### 9. Key Learnings from Layer 0
- Physical vs virtual infrastructure separation
- Network segmentation using VLANs
- Firewall policy design and trust boundaries
- Why traditional self-hosted VPN endpoints are operationally complex
- Identity-based access control as a modern alternative
- The value of minimizing exposed attack surface
- Designing for safety before adding vulnerable systems
- Understanding the “network brain” concept

---
### 12. Layer 0 Summary
Layer 0 provides a **secure, segmented, and future-proof foundation**.

It prioritizes:
- Safety
- Clarity
- Operability
- Learning the *right abstractions*

All higher layers build on this without rework.
