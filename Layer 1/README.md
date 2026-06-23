# Layer 1 — Virtualization & Compute

### Purpose

Layer 1 is the compute substrate everything above it runs on: Proxmox as hypervisor, LXC containers as the workload unit, VLAN-attached networking per container. Per the [[PROJECTS/Homelab/homelab/about|Architecture Overview]], this layer turns hardware into a private cloud — a repeatable, controlled platform for everything in Layers 2 and up.

### Status

**In progress.** Proxmox is running with three LXC containers deployed. k3s, Windows VMs, and snapshot discipline are not yet in place.

| Component | Status |
| --- | --- |
| Proxmox hypervisor | ✅ Installed, VLAN-aware bridge configured |
| LXC containers (3) | ✅ CT 100 (webapp), CT 101 (libretranslate), CT 103 (huckleberry) |
| k3s / lightweight Kubernetes | ⬜ Not started |
| Windows VMs | ⬜ Not started |
| Snapshot / backup discipline | ⬜ Not started |

### Container Inventory

This is the substrate-level view — what's actually running on Proxmox, regardless of which conceptual layer or cross-cutting category it serves. Same VLAN (30 / SERVICES), same per-container firewall caveat, three different purposes.

| CT ID | Hostname | Service | Belongs to | Deep-dive doc |
| --- | --- | --- | --- | --- |
| 100 | webapp | Language Learning App (Next.js + PM2) | Personal & Self-Hosted Services | [[Proxmox (Containers)\|Proxmox (Containers)]] § Webapp Container Setup |
| 101 | libretranslate | LibreTranslate (Docker) | Personal & Self-Hosted Services | [[Proxmox (Containers)\|Proxmox (Containers)]] § LibreTranslate Container Setup |
| 103 | huckleberry | Huckleberry habitat model — FastAPI + MLflow | Layer 5 — Data Science & ML | [[PROJECTS/Homelab/homelab/Layer 5/Huckleberry-Habitat-Model\|Huckleberry Habitat Model]] |

CT 100/101 are personal services that happen to be LXCs — see [[PROJECTS/Homelab/homelab/Personal & Self-Hosted Services/README|Personal & Self-Hosted Services]] for the cross-cutting view. CT 103 is Layer 5's first deployed component, but it's still a Layer 1 container like the other two: same LXC settings table conventions, same firewall-disabled rationale, same VLAN.

### Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[Proxmox\|Proxmox]] | Hypervisor install, VLAN-aware bridge fix, switch trunk config — the host itself |
| [[Proxmox (Containers)\|Proxmox (Containers)]] | LXC creation, webapp + LibreTranslate + Huckleberry container setup, the firewall/VLAN bug |
