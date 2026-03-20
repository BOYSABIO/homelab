# Homelab

A deliberately architected homelab built layer by layer around two core pillars: **cybersecurity** and **data science**. Personal and self-hosted services are first-class citizens but never compromise segmentation or learning goals.

For the full architectural overview (all 8 layers, design philosophy, core vs stretch goals), see [[about|Architecture Overview]].

---

## Current Status

**Layer 0 — Physical & Network Foundation** is implemented and stable.
Layers 1–7 are designed but not yet built.

---

## Layer 0 Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[Layer 0/README\|Overview & Blueprint]] | Target architecture, VLANs, subnets, milestones, hardware roles |
| [[Layer 0/Physical-Topology\|Physical Topology]] | Current hardware, IP ranges, switch port map, end devices |
| [[Layer 0/Firewall-Rules\|Firewall Rules]] | Design principles, per-VLAN intent, key learning moments |
| [[Layer 0/VLANs\|VLANs]] | Concepts, implementation, tagged vs untagged, lessons learned |
| [[Layer 0/DNS\|DNS]] | Unbound recursive resolver, DNSSEC, log cheat sheet |
| [[Layer 0/DHCP\|DHCP]] | Kea DHCP, static mappings, DORA handshake, log cheat sheet |
| [[Layer 0/IPv6\|IPv6]] | Internal ULA design, SLAAC, attribution, security |
| [[Layer 0/Tailscale\|Tailscale & Remote Access]] | MVP setup, ACLs, subnet routing, exit node |

---

## Service Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[Vintage-Story-Server\|Vintage Story Server]] | Dedicated game server admin guide (Ubuntu, Tailscale networking) |
| [[Local AI WebUI\|Local AI WebUI]] | Ollama + Open WebUI setup for LAN access |

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
