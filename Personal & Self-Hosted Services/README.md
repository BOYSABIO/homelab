# Personal & Self-Hosted Services (Cross-Cutting)

### Purpose

Per the [[PROJECTS/Homelab/homelab/about|Architecture Overview]]'s "Personal & Self-Hosted Services (Cross-Cutting)" section, personal services are first-class citizens in this homelab — but they never compromise security, segmentation, or the core learning goals. This is not a numbered layer; it cuts across whichever layer/host actually runs the service. This folder is the single place to find every personal/self-hosted service, regardless of where its deploy detail technically lives.

### Placement Strategy (per about.md)

- Run in the Services VLAN where possible
- Isolated from the Security Lab
- Managed like production services
- Logged and monitored where appropriate

### Services

| Service | Runs on | Status | Doc |
| ------- | ------- | ------ | --- |
| Language Learning App (webapp) | Proxmox LXC, CT 100, VLAN 30 (SERVICES) | ✅ Deployed | [[Proxmox (Containers)\|Proxmox (Containers)]] (Layer 1) § Webapp Container Setup |
| LibreTranslate | Proxmox LXC, CT 101, VLAN 30 (SERVICES) | ✅ Deployed | [[Proxmox (Containers)\|Proxmox (Containers)]] (Layer 1) § LibreTranslate Container Setup |
| Vintage Story Server | Dedicated machine, TRUSTED VLAN (routed to SERVICES via Tailscale/OPNsense for remote play) | ✅ Deployed | [[Vintage-Story-Server\|Vintage Story Server]] |
| Local AI WebUI (Ollama + Open WebUI) | Workstation, Docker Desktop — not Proxmox | ✅ Deployed | [[Local AI WebUI\|Local AI WebUI]] |
| OpenClaw | Workstation, local — not yet wired into Layer 0 segmentation | 🔧 Learning / setup phase | [[OpenClaw\|OpenClaw]] |

### Notes

webapp and LibreTranslate are documented under [[PROJECTS/Homelab/homelab/Layer 1/README|Layer 1]] instead of as standalone files in this folder, since their existing docs are primarily about *how they were built on Proxmox* (LXC creation, settings, Docker install) rather than the service itself. They're listed here anyway so this folder gives the complete cross-cutting picture — every personal service, no matter which layer or host actually runs it.

Vintage Story Server, Local AI WebUI, and OpenClaw don't run on Proxmox at all — different hosts, different VLANs (or, for OpenClaw, no formal VLAN placement yet) — so they get their own dedicated docs here instead of living inside a Layer folder.

Huckleberry (the habitat suitability model) is **not** listed here even though it's also a Docker workload on an LXC — it's a deployed ML inference API, which makes it a [[PROJECTS/Homelab/homelab/Layer 5/README|Layer 5]] concern, not a personal/self-hosted service in the sense `about.md` means (ad blocking, chatbots, media/game servers, dashboards). See [[PROJECTS/Homelab/homelab/Layer 5/Huckleberry-Habitat-Model|Huckleberry Habitat Model]].
