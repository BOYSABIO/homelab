# Homelab

A deliberately architected homelab built layer by layer around two core pillars: **cybersecurity** and **data science**. Personal and self-hosted services are first-class citizens but never compromise segmentation or learning goals.

For the full architectural overview (all 8 layers, design philosophy, core vs stretch goals), see [Architecture Overview](about.md).

---

## Repo layout

| Folder | What's in it |
|---|---|
| [`layer-0-network/`](layer-0-network/README.md) | Layer 0: physical topology, VLANs, firewall rules, DNS, DHCP, IPv6, Tailscale |
| [`layer-1-compute/`](layer-1-compute/README.md) | Layer 1: Proxmox and the LXC containers |
| [`layer-5-data-science/`](layer-5-data-science/README.md) | Layer 5: the ML services (Huckleberry habitat model) |
| [`services/`](services/README.md) | Personal and self-hosted services that run beside the layers |
| [`devices/`](devices/t440p/build.md) | Client machines and their setup (the ThinkPad T440p) |
| [`incidents/`](incidents/README.md) | One dated file per thing that broke, and what fixed it |
| [`about.md`](about.md) · [`PLANNED-EXPERIMENTS.md`](PLANNED-EXPERIMENTS.md) | The 8-layer architecture · ideas not built yet |

Layers 2–4 and 6–7 get their folder when they're built. Admin URLs and IPs are deliberately not in this repo.

---

## Current Status

**Layer 0 — Physical & Network Foundation** is implemented and stable.
**Layer 1 — Virtualization & Compute** is in progress — Proxmox is running with four LXC containers deployed (webapp, LibreTranslate, Huckleberry habitat model, Discord music bot). k3s, Windows VMs, and snapshot discipline not yet in place.
**Layer 5 — Data Science & ML** has its first component: the Huckleberry habitat suitability model (FastAPI + MLflow) deployed as a Dockerized inference API on its own LXC.
**Personal & Self-Hosted Services** run beside the layers: the MuttMetrics server (a dedicated OptiPlex) is live since 2026-11, monitored from outside by Healthchecks.io.
Layers 2–4 and 6–7 are designed but not yet built.

---

## Layer 0 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [Overview & Blueprint](layer-0-network/README.md) | Target architecture, VLANs, subnets, milestones, hardware roles |
| [Physical Topology](layer-0-network/Physical-Topology.md) | Current hardware, IP ranges, switch port map, end devices |
| [Firewall Rules](layer-0-network/Firewall-Rules.md) | Design principles, per-VLAN intent, key learning moments |
| [VLANs](layer-0-network/VLANs.md) | Concepts, implementation, tagged vs untagged, lessons learned |
| [DNS](layer-0-network/DNS.md) | Unbound recursive resolver, DNSSEC, log cheat sheet |
| [DHCP](layer-0-network/DHCP.md) | Kea DHCP, static mappings, DORA handshake, log cheat sheet |
| [IPv6](layer-0-network/IPv6.md) | Internal ULA design, SLAAC, attribution, security |
| [Tailscale & Remote Access](layer-0-network/Tailscale.md) | MVP setup, ACLs, subnet routing, exit node |

---

## Layer 1 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [Overview & Container Inventory](layer-1-compute/README.md) | Layer 1 scope, status, full container inventory across all conceptual layers |
| [Proxmox](layer-1-compute/Proxmox.md) | Hypervisor install, VLAN-aware bridge fix, switch trunk config — the host itself |
| [Proxmox (Containers)](layer-1-compute/Proxmox-Containers.md) | LXC creation, webapp + LibreTranslate + Huckleberry container setup, the firewall/VLAN bug |

---

## Layer 5 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [Overview & Status](layer-5-data-science/README.md) | Layer 5 scope, what's deployed vs. planned |
| [Huckleberry Habitat Model](layer-5-data-science/Huckleberry-Habitat-Model.md) | ML inference API + MLflow, deployed to a Proxmox LXC — settings, stack, update flow, troubleshooting |

---

## Personal & Self-Hosted Services Documentation

| Document | What It Covers |
| -------- | -------------- |
| [Overview](services/README.md) | Cross-cutting philosophy, full service inventory (incl. webapp/LibreTranslate, documented under Layer 1) |
| [Vintage Story Server](services/Vintage-Story-Server.md) | Dedicated game server admin guide (Ubuntu, Tailscale networking) |
| [MuttMetrics Server](services/MuttMetrics-Server.md) | Always-on host for a small business app: placement, access model, hardening, monitoring, backups |
| [Local AI WebUI](services/Local-AI-WebUI.md) | Ollama + Open WebUI setup for LAN access |
| [OpenClaw](services/OpenClaw.md) | Local OpenClaw learning instance: identity, Telegram, Control UI, memory findings |

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
