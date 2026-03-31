# Proxmox: Setup, Troubleshooting, and Key Learnings
> This document covers the full process of adding a Proxmox hypervisor to the homelab —
> from switch configuration through to a working web UI. It captures not just what was done,
> but why things failed, what the failures revealed, and what mental models were built along the way.

---
## 1. Where Proxmox Lives and Why
Proxmox was placed in **VLAN10 (Management)** with a static IP of **10.xx.xx.3**.

The reasoning follows directly from the Layer 0 firewall design philosophy. The management VLAN exists specifically to host the control planes of infrastructure — things like the firewall UI, the switch UI, and now the hypervisor management interface. Proxmox is not a workload, it is the thing that *runs* workloads. Its management interface has no business sitting on the same network as user devices or servers. Placing it in MGMT keeps the control plane isolated and matches how enterprise environments treat hypervisor management — separately from the data plane entirely.

The IP of 10.x.x.3 was chosen deliberately following the IP pool convention already established:
- 10.x.x.1 → OPNsense (the router/gateway)
- 10.x.x.2 → The TP-Link switch management interface
- 10.x.x.3 → Proxmox

---
## 2. Switch Configuration — Port 7 as a Trunk
### The Decision
Proxmox needs to be a **trunk port**, not an access port. This is because Proxmox is a VLAN-aware device. Unlike a regular PC that just needs to exist on one VLAN, Proxmox needs to be able to spawn virtual machines on *any* VLAN — MGMT, Trusted, Services, CyberLab, or Guest. The only way to carry multiple VLANs over a single cable is with a trunk, where each frame is tagged with its VLAN ID so the receiving device knows which network it belongs to.

Port 7 was chosen as the Proxmox port.

### What Was Configured on the Switch
**802.1Q VLAN page** — Port 7 was added as a **tagged** member of every VLAN. Tagged means the switch will preserve and expect VLAN ID tags on frames going to and from Port 7. This is the opposite of an access port, where the switch strips and adds tags transparently on behalf of a device that doesn't understand VLANs.

Port 7 was also explicitly **excluded from VLAN1** (the legacy system VLAN), for the same reason Port 1 (the OPNsense uplink) is excluded — a trunk port should carry only intentional, named VLANs, not the legacy default.

**Port Config page** — Port 7's **PVID was set to MGMT PVID NUMBER**. The PVID (Port VLAN ID) determines what VLAN any *untagged* frame arriving on this port gets assigned to. Even though Proxmox will eventually send tagged frames for everything, having the PVID set to MGMT is a safety net — any management traffic sent before VLAN tagging is fully configured lands in the right place.

**Ingress checking was enabled** and **Acceptable Frame Types was set to Tagged Only**. This hardens the trunk, ensuring no untagged or incorrectly tagged frames can sneak in and cause ambiguity. It mirrors the same settings applied to Port 1 (the OPNsense trunk), making Port 7 a proper second trunk port.

### Important Observation About VLAN1
The switch has a legacy VLAN1 called "System VLAN" that contains all ports as untagged members by default. This was left in place from the original flat network setup and couldn't be cleanly removed because TP-Link Omada switches treat VLAN1 as a built-in default. Rather than fighting it, the approach was to simply not add Port 7 (or Port 1) to it, keeping it as a harmless legacy artifact that no infrastructure port touches.

---
## 3. OPNsense Configuration — Firewall Rule for Web UI Access
By design, VLAN10 (Management) is not reachable from VLAN20 (Trusted). This is intentional and documented in the Layer 0 firewall rules — management interfaces should not be freely accessible from user devices. However, Proxmox's web UI runs on port 8006 and needs to be reachable from the primary workstation on VLAN20.

The solution was to add a **specific, narrow firewall rule on the VLAN20 interface** in OPNsense:
- **Interface:** VLAN20_TRUST (rules on this interface govern traffic *originating* from VLAN20)
- **Action:** Pass
- **Protocol:** TCP
- **Source:** VLAN20_TRUST net (any device on the trusted network)
- **Destination:** 10.x.x.3 (specifically Proxmox, not all of MGMT)
- **Destination Port:** 8006

This rule was placed **above** the existing "Block Trusted to MGMT" rule. This ordering is critical. OPNsense processes firewall rules from top to bottom and stops at the first match. If the block rule appeared first, it would catch the traffic before the allow rule ever had a chance to evaluate it. The allow rule must come first so that traffic to Proxmox on port 8006 is permitted before the general MGMT block applies.

This pattern — allowing specific exceptions above a broad block rule — is the correct and intentional way to punch precise holes in a default-deny segmented network. It keeps the MGMT block intact for everything else while allowing exactly one host on one port.

---
## 4. The Proxmox Installation
### Media Preparation
Proxmox VE 9.1 was downloaded from proxmox.com and flashed to a USB drive using Rufus. One important detail: when Rufus prompts for write mode, **DD image mode must be selected**, not ISO mode. Proxmox's installer is structured as a disk image and does not behave correctly when written in ISO mode. This is a common trap.

### Installation Process
The Optiplex was booted from the USB. The **Graphical Terminal UI** option was selected from the Proxmox boot menu — this is the standard path and requires a connected monitor, keyboard, and mouse since it presents an interactive installer.

During the installer, the network was configured manually with a static IP:
- **IP:** 10.x.x.3/24
- **Gateway:** 10.x.x.1
- **DNS:** 10.x.x.1

After installation, the USB was removed and the machine was rebooted.

---
## 5. The Connectivity Problem — And What It Revealed
After installation, Proxmox appeared to be up (the console showed the expected welcome message and the 10.x.x.3:8006 URL), but it was completely unreachable from both OPNsense and from VLAN20. Pinging 10.x.x.3 from OPNsense returned "No route to host." The Proxmox web UI returned nothing. The firewall logs showed traffic from VLAN20 being blocked even though the allow rule was in place and correctly positioned.

This triggered a systematic diagnosis that ultimately revealed two separate issues and taught several important lessons.

### The Real Problem — Untagged Frames on a Tagged-Only Port
With Proxmox connected, `bridge link show` confirmed that nic0 was correctly enslaved to vmbr0 with state `forwarding`. The IP 10.x.x.3 was present on vmbr0. Everything looked correct from Proxmox's perspective. And yet OPNsense still couldn't reach it. The ARP table on OPNsense showed no entry for 10.x.x.3. A packet capture on the LAN interface showed traffic on VLAN20, VLAN30, but absolutely nothing from VLAN10.

This pointed to the frames either not arriving at OPNsense at all, or arriving but being classified incorrectly.

The actual cause was this: **Proxmox's default network configuration sends completely untagged frames.** The default `/etc/network/interfaces` that the Proxmox installer creates sets up a standard Linux bridge (`vmbr0`) where the physical NIC is a bridge port and the bridge has a plain static IP. There is no VLAN tagging anywhere in this configuration. Proxmox just sends raw untagged Ethernet frames onto the wire.

Meanwhile, Port 7 on the switch was configured as a trunk with **Tagged Only** and ingress checking enabled. This means the switch will only accept frames that already carry a VLAN tag. An untagged frame arriving on Port 7 is silently dropped — not rejected with an error, just discarded as if it never existed.

This is why nothing showed up in any packet capture anywhere. The frames were being dropped at the switch before they even reached OPNsense. The PVID setting on Port 7 (which would normally assign untagged traffic to VLAN10) is overridden by the "Tagged Only" acceptable frame type setting — if tagged-only is enforced, untagged frames never get the chance to be assigned to the PVID VLAN.

This is a perfect real-world demonstration of a core concept from the VLANs documentation:
> **End devices send untagged traffic. Infrastructure devices send tagged traffic.**

A regular PC plugged into an access port is an end device. It sends untagged frames and the switch handles all the VLAN assignment transparently. Proxmox is infrastructure. It connects via a trunk port and is expected to tag its own traffic, because it is a VLAN-aware device that understands the 802.1Q standard.

---
## 6. The Fix — Making Proxmox VLAN-Aware
The solution was to edit `/etc/network/interfaces` on Proxmox to replace the default untagged bridge configuration with a VLAN-aware one.

### What the Default Config Looked Like
```

auto lo

iface lo inet loopback

  

iface nic0 inet manual

  

auto vmbr0

iface vmbr0 inet static

        address 10.x.x.3/24

        gateway 10.x.x.1

        bridge-ports nic0

        bridge-stp off

        bridge-fd 0

```
This creates a simple bridge. nic0 is a member of the bridge, and the bridge itself holds the IP address. All frames leaving the bridge go out through nic0 untagged.

### What the Fixed Config Looks Like
```

auto lo

iface lo inet loopback

  

iface nic0 inet manual

  

auto vmbr0

iface vmbr0 inet manual

        bridge-ports nic0

        bridge-stp off

        bridge-fd 0

        bridge-vlan-aware yes

        bridge-vids 2-4094

  

auto vmbr0.10

iface vmbr0.10 inet static

        address 10.x.x.3/24

        gateway 10.x.x.1

```
There are several important changes here worth understanding in detail.

**`bridge-vlan-aware yes`** tells the Linux kernel bridge driver to operate in VLAN-aware mode. In this mode, the bridge understands 802.1Q tags and can handle traffic from multiple VLANs simultaneously. Without this line, the bridge treats all traffic as belonging to one flat network and strips any tags it encounters.

**`bridge-vids 2-4094`** tells the bridge which VLAN IDs are allowed to pass through it. This is a permissive range that covers essentially all useful VLANs. When a VM is later assigned to VLAN20 or VLAN30, the bridge will already be prepared to carry that traffic.

**`vmbr0.10`** is a VLAN subinterface. The `.10` notation means "VLAN 10 on top of vmbr0." This is where the host's own management IP lives. When Proxmox sends traffic from this interface, it automatically tags it with VLAN ID 10. When the switch receives a frame tagged with VLAN10 on Port 7, it processes it correctly and forwards it to Port 1 (OPNsense), where the VLAN10_MGMT interface picks it up and routes it normally.

The bridge itself no longer holds the IP — it is now `inet manual` because it is a pure Layer 2 switching fabric. Only the VLAN subinterface holds a routable IP address.

After saving the file and rebooting (or running `ifreload -a`), Proxmox began sending properly tagged VLAN10 frames. OPNsense immediately saw them, the ARP table populated with 10.x.x.3, and the web UI became accessible at `https://10.x.x.3:8006`.

---
## 7. Why This Design is Forward-Looking
The VLAN-aware bridge configuration is not just a fix — it is the correct long-term architecture for a Proxmox homelab host that will run VMs across multiple VLANs.

When a VM is created and assigned to VLAN20 (Trusted), Proxmox creates a virtual network interface for that VM and tags its traffic with VLAN ID 20. That tagged traffic passes through vmbr0, exits through nic0 onto the trunk cable, hits Port 7 on the switch (which accepts all tagged VLANs), travels to Port 1 (the OPNsense trunk), and is received by OPNsense on the VLAN20_TRUST interface. The VM behaves exactly as if it were a physical machine plugged into an access port on VLAN20 — it just uses a virtual cable instead of a physical one.

This is the power of a trunk port combined with a VLAN-aware bridge. One physical cable carries all VLANs. The switch config never needs to change as new VMs are added. The firewall policies already in place for each VLAN apply automatically to any VM placed on that VLAN.

---
## 8. Proxmox Web UI Access and the Firewall Rule Ordering Problem
Even after the connectivity problem was resolved, there was a brief period where the VLAN20 → Proxmox firewall rule appeared to be configured correctly but traffic was still being logged as blocked. The rule was visible in OPNsense, it was positioned above the "Block Trusted to MGMT" rule, and it had the correct destination IP and port — but the logs kept showing blocks.

The eventual resolution was simply ensuring **Apply Changes** had been clicked after saving the rule. In OPNsense, creating or editing a rule writes it to the pending configuration but does not activate it until changes are explicitly applied. This is a deliberate design — it allows multiple rule changes to be staged and applied atomically — but it means that a rule can appear correct in the UI while not actually being enforced yet.

This is worth remembering: **in OPNsense, saving is not the same as applying.** Always look for the Apply Changes prompt after modifying firewall rules.

---
## 9. Remaining Steps (To Complete Layer 1 Baseline)
The switch is configured, Proxmox is reachable, and the web UI is accessible. Before moving on to spinning up VMs, the following should be completed:

**Run system updates** — A temporary rule allowing MGMT → Internet should be added to OPNsense, updates run from the Proxmox shell (`apt update && apt dist-upgrade -y`), and the rule removed immediately after. The "no valid subscription" warning from Proxmox is expected behavior for the free community edition and can be dismissed.

**Add a DHCP reservation or document the static** — Proxmox's IP is statically configured inside the machine itself so Kea DHCP will never show a lease for it. This is correct behavior. However, it is worth adding a note in the physical topology documentation so the .3 address is never accidentally reused or questioned.

**Disable the Proxmox enterprise repository** — By default Proxmox points at a paid enterprise repository for updates. Since this is a homelab using the community edition, the enterprise repo should be disabled and the free community repo added instead. Without doing this, `apt update` will throw authentication errors.
```bash

# Disable enterprise repo

echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list

  

# Add community repo

echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \

  > /etc/apt/sources.list.d/pve-community.list

  

apt update && apt dist-upgrade -y

```

---

## 10. Key Learnings from This Process
**Infrastructure devices tag their own traffic.** End devices like PCs rely on the switch to handle VLAN assignment transparently via access ports. Infrastructure devices like firewalls, hypervisors, and access points connect via trunks and are expected to send and receive tagged frames themselves. Proxmox defaulting to an untagged bridge is a deliberate choice for simple environments — but this homelab is not a simple environment.

**Tagged-only + ingress checking is a silent killer.** When a port is configured to accept only tagged frames and a device sends untagged frames, those frames are dropped without any log, any error, or any indication that anything is wrong. The device thinks it sent the packet. The switch says nothing. The destination sees nothing. This makes it one of the hardest failure modes to diagnose because everything looks correct at every layer — until you understand that the frames never survived the ingress check.

**PVID does not rescue you from Tagged Only.** It would be reasonable to assume that since Port 7's PVID was set to 10, any untagged frame arriving there would be assigned to VLAN10. This assumption is wrong when "Tagged Only" is enabled. The PVID only applies to untagged frames that are *accepted*. If the port is set to reject untagged frames, they never reach the PVID assignment logic.

**No route to host from OPNsense to its own subnet means something is wrong at Layer 2.** OPNsense is the gateway for VLAN10. If it returns "no route to host" when pinging an address on its own subnet, the problem is not routing — routing is fine. The problem is that OPNsense cannot see the target device at all, which means either the device is not on the network or its frames are not reaching OPNsense. This is a Layer 2 problem and should be diagnosed at Layer 2.

**An empty ARP table is the clearest signal.** If a device has a correct IP, a correct gateway, an active bridge, and a connected cable — but OPNsense's ARP table has no entry for it — then traffic is not flowing at all. The ARP table is populated when a device first communicates on the network. No ARP entry means no communication, which means the problem is before the point where any IP-level diagnosis can help.

**The bridge-vlan-aware architecture is the correct Proxmox design for multi-VLAN homelabs.** The Proxmox default configuration is designed for environments with a flat untagged network. For any environment with VLANs — which is any environment serious about network segmentation — the bridge must be made VLAN-aware and a subinterface must be used for the host's own management IP. This is not a workaround, it is the intended architecture.

**Order of operations in OPNsense matters.** Rules are evaluated top to bottom, first match wins, and changes must be applied before they are active. These three facts explain most OPNsense firewall debugging sessions.