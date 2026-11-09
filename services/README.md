# Personal & Self-Hosted Services (Cross-Cutting)

### Purpose

Per the [Architecture Overview](../about.md)'s "Personal & Self-Hosted Services (Cross-Cutting)" section, personal services are first-class citizens in this homelab — but they never compromise security, segmentation, or the core learning goals. This is not a numbered layer; it cuts across whichever layer/host actually runs the service. This folder is the single place to find every personal/self-hosted service, regardless of where its deploy detail technically lives.

### Placement Strategy (per about.md)

- Run in the Services VLAN where possible
- Isolated from the Security Lab
- Managed like production services
- Logged and monitored where appropriate

### Services

| Service | Runs on | Status | Doc |
| ------- | ------- | ------ | --- |
| Cveda (webapp) | Proxmox LXC, CT 100, VLAN 30 (SERVICES) | ✅ Deployed | [Proxmox (Containers)](../layer-1-compute/Proxmox-Containers.md) (Layer 1) § Webapp Container Setup |
| LibreTranslate | Proxmox LXC, CT 101, VLAN 30 (SERVICES) | ✅ Deployed | [Proxmox (Containers)](../layer-1-compute/Proxmox-Containers.md) (Layer 1) § LibreTranslate Container Setup |
| Vintage Story Server | Dedicated machine, VLAN 30 (SERVICES), switch port 4 via an unmanaged switch; Tailscale `tag:vsserver` | ✅ Deployed | [Vintage Story Server](Vintage-Story-Server.md) |
| MuttMetrics server | Dedicated OptiPlex (Debian), VLAN 30 (SERVICES), switch port 4 via an unmanaged switch; Tailscale `tag:muttmetrics` | ✅ Live (muttmetrics#93) | [MuttMetrics Server](MuttMetrics-Server.md) |
| Local AI WebUI (Ollama + Open WebUI) | Workstation, Docker Desktop — not Proxmox | ✅ Deployed | [Local AI WebUI](Local-AI-WebUI.md) |
| OpenClaw | Workstation, local — not yet wired into Layer 0 segmentation | 🔧 Learning / setup phase | [OpenClaw](OpenClaw.md) |
| Discord Music Bot | Proxmox LXC, CT 104, VLAN 30 (SERVICES) | ✅ Deployed | [Discord-Music-Bot](Discord-Music-Bot.md) |

### Notes

webapp and LibreTranslate are documented under [Layer 1](../layer-1-compute/README.md) instead of as standalone files in this folder, since their existing docs are primarily about *how they were built on Proxmox* (LXC creation, settings, Docker install) rather than the service itself. They're listed here anyway so this folder gives the complete cross-cutting picture — every personal service, no matter which layer or host actually runs it.

Vintage Story Server, MuttMetrics Server, Local AI WebUI, and OpenClaw don't run on Proxmox at all — different hosts, different VLANs (or, for OpenClaw, no formal VLAN placement yet) — so they get their own dedicated docs here instead of living inside a Layer folder.

Huckleberry (the habitat suitability model) is **not** listed here even though it's also a Docker workload on an LXC — it's a deployed ML inference API, which makes it a [Layer 5](../layer-5-data-science/README.md) concern, not a personal/self-hosted service in the sense `about.md` means (ad blocking, chatbots, media/game servers, dashboards). See [Huckleberry Habitat Model](../layer-5-data-science/Huckleberry-Habitat-Model.md).
