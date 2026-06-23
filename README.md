# Homelab

A deliberately architected homelab built layer by layer around two core pillars: **cybersecurity** and **data science**. Personal and self-hosted services are first-class citizens but never compromise segmentation or learning goals.

For the full architectural overview (all 8 layers, design philosophy, core vs stretch goals), see [[PROJECTS/Homelab/homelab/about|Architecture Overview]].

---

## Current Status

**Layer 0 — Physical & Network Foundation** is implemented and stable.
**Layer 1 — Virtualization & Compute** is in progress — Proxmox is running with three LXC containers deployed (webapp, LibreTranslate, Huckleberry habitat model). k3s, Windows VMs, and snapshot discipline not yet in place.
**Layer 5 — Data Science & ML** has its first component: the Huckleberry habitat suitability model (FastAPI + MLflow) deployed as a Dockerized inference API on its own LXC.
Layers 2–4 and 6–7 are designed but not yet built.

---

## Layer 0 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[PROJECTS/Homelab/homelab/Layer 0/README\|Overview & Blueprint]] | Target architecture, VLANs, subnets, milestones, hardware roles |
| [[Physical-Topology\|Physical Topology]] | Current hardware, IP ranges, switch port map, end devices |
| [[Firewall-Rules\|Firewall Rules]] | Design principles, per-VLAN intent, key learning moments |
| [[VLANs\|VLANs]] | Concepts, implementation, tagged vs untagged, lessons learned |
| [[DNS\|DNS]] | Unbound recursive resolver, DNSSEC, log cheat sheet |
| [[DHCP\|DHCP]] | Kea DHCP, static mappings, DORA handshake, log cheat sheet |
| [[IPv6\|IPv6]] | Internal ULA design, SLAAC, attribution, security |
| [[Tailscale\|Tailscale & Remote Access]] | MVP setup, ACLs, subnet routing, exit node |

---

## Layer 1 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[PROJECTS/Homelab/homelab/Layer 1/README\|Overview & Container Inventory]] | Layer 1 scope, status, full container inventory across all conceptual layers |
| [[Proxmox\|Proxmox]] | Hypervisor install, VLAN-aware bridge fix, switch trunk config — the host itself |
| [[Proxmox (Containers)\|Proxmox (Containers)]] | LXC creation, webapp + LibreTranslate + Huckleberry container setup, the firewall/VLAN bug |

---

## Layer 5 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[PROJECTS/Homelab/homelab/Layer 5/README\|Overview & Status]] | Layer 5 scope, what's deployed vs. planned |
| [[Huckleberry-Habitat-Model\|Huckleberry Habitat Model]] | ML inference API + MLflow, deployed to a Proxmox LXC — settings, stack, update flow, troubleshooting |

---

## Personal & Self-Hosted Services Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[PROJECTS/Homelab/homelab/Personal & Self-Hosted Services/README\|Overview]] | Cross-cutting philosophy, full service inventory (incl. webapp/LibreTranslate, documented under Layer 1) |
| [[Vintage-Story-Server\|Vintage Story Server]] | Dedicated game server admin guide (Ubuntu, Tailscale networking) |
| [[Local AI WebUI\|Local AI WebUI]] | Ollama + Open WebUI setup for LAN access |
| [[OpenClaw\|OpenClaw]] | Local OpenClaw learning instance: identity, Telegram, Control UI, memory findings |

---

## Design Philosophy

1. **Security First** — Segmentation before services
2. **Real-World Relevance** — Enterprise-style networking and SOC-style telemetry
3. **Data-Centric Thinking** — Logs, flows, and metrics are first-class citizens
4. **Incremental Growth** — Stable core before stretch features

---

## Repository Security

This repo uses [Gitleaks](https://github.com/gitleaks/gitleaks) to prevent accidental exposure of sensitive information (private IPs, API keys, tokens, etc.).

**Two layers of protection are in place:**

| Layer | Where it runs | What it does |
| ----- | ------------- | ------------ |
| Pre-commit hook | Locally before every commit | Scans staged files and blocks the commit if a leak is detected |
| GitHub Actions CI | On every push / pull request | Scans the full history as a second safety net |

Custom rules are defined in `.gitleaks.toml` to detect private IP address ranges (`192.168.x.x`, `10.x.x.x`, `172.16-31.x.x`, Tailscale CGNAT) in addition to the default secret patterns.

**Useful commands:**

```bash
gitleaks detect --source . --verbose              # scan current files
gitleaks detect --source . --log-opts="--all"      # scan full git history
gitleaks protect --staged --verbose                # scan staged files (what the hook runs)
```
