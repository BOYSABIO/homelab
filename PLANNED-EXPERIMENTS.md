# Planned Experiments & Next Builds

This document captures concrete next-step projects for the homelab — ideas that go beyond the architectural sketch in `about.md` and are close enough to act on. Each entry maps to its layer, lists what's needed, and describes what you'd actually learn or build.

---

## 1. Honeypot with Live Visual Display
**Layer:** 3 (Security Stack — Blue Team)  
**Status:** Planned

### What it is
A medium-interaction honeypot that emulates SSH, Telnet, and a handful of common services. Any connection attempt gets logged, the source IP gets flagged, and a physical display shows a live visualization of what's hitting it.

### Hardware
- Raspberry Pi (any model with enough RAM for the daemon + display script)
- Small screen: either a dedicated Pi HAT display (e.g. Waveshare 3.5") or a spare monitor via HDMI
- Alternatively: a spare Optiplex running the honeypot + a screen attached

### Software Stack
- **[Cowrie](https://github.com/cowrie/cowrie)** — SSH/Telnet honeypot, logs all commands typed by attackers, captures credentials used
- **[OpenCanary](https://github.com/thinkst/opencanary)** — lightweight multi-service honeypot (FTP, HTTP, RDP, SMB stubs) as a complement
- **Display script** — Python script running on the Pi that pulls from Cowrie's JSON log in real time and renders: live connection attempts, source IPs, commands typed, maybe a world map ping animation using `curses` or a simple Pygame overlay

### Auto-Response Logic
1. Cowrie logs a connection → script parses log
2. If IP hits threshold (e.g. 3+ attempts) → call OPNsense API to add IP to a blocklist alias
3. Trigger a notification (Telegram bot / ntfy.sh) with the IP, time, and commands attempted

### What it feeds
- SIEM (when Layer 3 is up) — honeypot logs are high-signal, low-noise indicators
- ML layer (Layer 5) — clean ground truth for malicious IP behavior
- The display is both functional and a good visual piece

### Prerequisites
- Layer 0 stable ✓
- Honeypot should sit in the **Security Lab VLAN** — isolated, reachable from outside only via specific forwarded ports or a DMZ rule

---

## 2. Malware Analysis & Behavior Lab
**Layer:** 4 (Offensive & Adversary Simulation)  
**Status:** Planned — after Layer 1 (Proxmox) is up

### What it is
A fully isolated sandbox environment for studying how malware works: how it persists, evades, communicates with C2, and what telemetry it generates. The goal is to understand malware behavior from first principles — not just run scanners against it.

### Architecture
- **Isolated VLAN** — Security Lab VLAN already planned in Layer 0. No route to trusted devices or internet (or only through a controlled egress with full capture)
- **Windows VM** — Primary malware target (most malware targets Windows). Use a snapshot baseline so you can restore instantly
- **Linux Analysis VM** — REMnux or Flare-VM (or both): static analysis, dynamic tracing, network capture
- **Sandbox orchestrator** — [CAPE Sandbox](https://github.com/kevoreilly/CAPEv2) or [Cuckoo3](https://github.com/cert-ee/cuckoo3) for automated behavioral analysis with reports

### What you'd study / build
- Persistence mechanisms: registry run keys, scheduled tasks, DLL hijacking, service installation
- Evasion: sandbox detection (sleep, environment checks), AMSI bypass, process hollowing
- C2 communication: HTTP beaconing, DNS tunneling, encrypted channels
- Lateral movement: credential dumping, pass-the-hash, SMB propagation (once AD is up in Layer 2)
- Write simple proof-of-concept versions of each technique in a contained environment to understand the mechanics

### What it feeds
- Layer 3 — your honeypot and SIEM now have *real* malware telemetry, not just port scans
- Layer 5 — labeled behavioral data for ML models (e.g. "these network flows are C2 beaconing")
- Career signal — hands-on malware analysis is a direct skill for SOC/threat intel/red team roles

### Prerequisites
- Layer 1 (Proxmox) running with snapshot capability
- Security Lab VLAN confirmed isolated at Layer 0
- Strong habit of snapshot-before-execute

---

## 3. Full Red/Blue Team Security Exercise
**Layer:** 3 + 4 (combined — the first real integration test)  
**Status:** Later phase — do this after ML network monitoring and SOC triage agent are deployed

### What it is
A structured attack-and-defend exercise run entirely within the homelab. You attack from Kali, your SIEM + ML models try to catch you, and you iteratively tune detection until you can reliably alert on your own attacks.

### Why "later"
This exercise only has value once the detection stack exists. Running it now (no SIEM, no ML model) means you learn attack techniques but get no feedback on detection. The goal is to run both sides simultaneously — attack, then look at what the defender saw.

### Exercise structure
**Phase A — External Recon & Initial Access**
- Port scan the homelab from Kali (external or Security Lab VLAN)
- Attempt to exploit exposed services (if any intentional vulnerable VMs are up)
- Try credential attacks against the honeypot
- Observe: does Suricata alert? Does the ML model flag anomalous flows?

**Phase B — Internal Lateral Movement (requires Layer 2 — AD)**
- Once inside a system, attempt to enumerate AD, Kerberoast service accounts, pass-the-hash
- Observe: does Wazuh/endpoint agent catch the credential access?

**Phase C — Detection Tuning**
- For every attack that wasn't caught: trace why, tune the rule or model
- For every false positive: reduce noise without losing signal
- Document the gap and the fix

### What to track
A simple markdown table per exercise run: attack technique → detected (Y/N) → alert latency → false positive rate → action taken

### Prerequisites
- Layer 2 (AD/FreeIPA) deployed
- Layer 3 (Suricata, SIEM, Zeek) operational
- ML network monitoring model deployed
- SOC triage agent deployed
- Honeypot active

---

## 4. NAS — Network Attached Storage
**Layer:** Cross-cutting (Layer 0/1 infrastructure + Layer 5 AI/data integration)  
**Status:** Planned — can start early, useful immediately

### What it is
A dedicated NAS built from one of the spare Optiplex PCs. Provides shared storage for the entire homelab — backups, datasets, model weights, media — and has a specific role in the local AI stack.

### Hardware
- Dell Optiplex (any generation) with as much RAM as you can get in it
- Add drives: even a couple of used 2TB SATA drives work fine for a start
- Optional: HBA card if you want more drive bays later

### OS Options
- **TrueNAS SCALE** (recommended) — ZFS, Docker app support, built-in Samba/NFS/iSCSI, good UI. ZFS gives you data integrity, snapshots, and easy expansion.
- **OpenMediaVault** — lighter weight, also solid, easier on low-RAM hardware

### Storage Layout (suggested)
| Dataset | Purpose |
|---------|---------|
| `backups/` | VM snapshots from Proxmox, config backups from OPNsense |
| `datasets/` | ML training data, Zeek/SIEM logs for offline analysis |
| `models/` | Ollama model weights (avoids re-downloading on every machine) |
| `media/` | Personal media if running Jellyfin/Plex |
| `scratch/` | Temporary working space for experiments |

### AI & Local Model Integration
The NAS changes the local AI stack meaningfully:

- **Shared model weights** — Ollama on multiple hosts can pull from a single NFS/SMB share instead of each machine storing its own copy. Saves significant disk space.
- **RAG database storage** — Vector databases (Chroma, Qdrant, pgvector) can use the NAS for persistent storage. Run the DB on Proxmox, data lives on NAS.
- **Dataset versioning** — Store training datasets and experiment artifacts on NAS, track with DVC (Data Version Control) pointing at the NAS share as remote.
- **Log archival** — Ship aged-out SIEM logs to NAS for long-term retention without filling up Proxmox storage.

### MCP Server Angle
This is the most interesting idea. A dedicated MCP (Model Context Protocol) server running on or alongside the NAS that gives local AI models structured access to your data:

- **Filesystem MCP** — lets Claude/local models browse and read files on the NAS (personal docs, notes, project files)
- **RAG MCP** — exposes a vector search interface over indexed documents on the NAS; models can query your own knowledge base
- **Homelab context MCP** — indexes your homelab docs, incident logs, config files; lets an AI assistant answer "what are my current firewall rules?" by actually reading the docs

The MCP server itself is lightweight — a small Python or Node service running in a Proxmox container, pointing at NAS shares. Could be its own dedicated container or co-hosted on the NAS if running TrueNAS SCALE (which supports Docker apps natively).

### Integration with Homelab
- NAS sits on the **Services VLAN** (or Management VLAN if you want strict separation)
- Proxmox mounts NAS shares for VM storage/backups
- ML VMs mount `datasets/` and `models/` via NFS
- Ollama instances point at shared model directory

### Prerequisites
- Spare Optiplex + drives
- TrueNAS SCALE ISO
- Layer 0 VLAN already configured — just add a new DHCP static mapping and firewall rule for the NAS IP
