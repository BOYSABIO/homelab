# Ultimate Homelab — Architecture Overview
### Purpose of This Document
This document provides a **high-level architectural overview** of the entire homelab.

It explains:
- The layered structure of the homelab
- The purpose of each layer
- How cybersecurity and data science are the core focus
- How personal/self-hosted services fit in safely
- What is considered *core* vs *stretch* functionality

This document intentionally avoids implementation details.
Each layer has (or will have) its own dedicated documentation.

---
### Design Philosophy
This homelab is designed around four core principles:

1. **Security First**
   - Segmentation before services
   - Safe-by-default design
   - Vulnerable systems isolated by architecture, not discipline

2. **Real-World Relevance**
   - Enterprise-style networking
   - SOC-style telemetry
   - Production-style ML pipelines
   - Skills that transfer directly to professional roles

3. **Data-Centric Thinking**
   - Everything generates data
   - Logs, flows, events, and metrics are first-class citizens
   - ML models operate on real telemetry, not toy datasets

4. **Incremental Growth**
   - A stable core comes first
   - Stretch features are added only after foundations are solid
   - No architectural rework as complexity grows

---
### Layered Architecture Overview
The homelab is organized into **eight conceptual layers**.

Each layer builds on the ones below it.
Higher layers never bypass lower layers.

---
### [[Layer 0/README|Layer 0 — Physical & Network Foundation]]
#### Purpose
Defines the **physical hardware**, **network topology**, and **trust boundaries**.
This is the most important layer — everything else depends on it.
#### Responsibilities
- Firewall and routing
- VLAN segmentation
- Physical separation of trust zones
- Secure remote access
- Safety guarantees for running hostile systems
#### Core Outcomes
- Dedicated firewall (OPNsense)
- Managed switch with VLANs
- Clear separation between:
  - Management
  - Trusted devices
  - Services
  - Security Lab
  - IoT / untrusted devices
- Secure remote access via identity-based networking (Tailscale)
#### Why It Matters
Layer 0 ensures the homelab is:
- Safe
- Scalable
- Resistant to mistakes
- Realistic from a security perspective

---
### Layer 1 — Virtualization & Compute Platform

#### Purpose
Provides the **compute substrate** on which all workloads run.
#### Responsibilities
- Hypervisor (Proxmox)
- VM lifecycle management
- Network attachment to VLANs
- Resource allocation (CPU, RAM, storage)
- Container hosts
#### Core Outcomes
- Proxmox as the primary hypervisor
- Linux and Windows VMs
- Docker / Podman environments
- Optional lightweight Kubernetes (k3s) for modern workloads
#### Why It Matters
This layer turns hardware into:
- A private cloud
- A repeatable lab environment
- A controlled testing platform

---
### Layer 2 — Identity & Access Control
#### Purpose
Centralizes **who can access what** inside the homelab.
#### Responsibilities
- Authentication
- Authorization
- Role separation
- Least privilege enforcement
#### Core Outcomes
- Active Directory or FreeIPA
- Domain-joined systems
- Centralized auth for:
  - Git
  - Dashboards
  - Internal apps
  - Admin access
#### Why It Matters
Identity is the backbone of real security environments.
This layer enables:
- Realistic attack scenarios
- Blue-team detection
- Proper access control modeling

---
### Layer 3 — Security Stack (Blue Team)
#### Purpose
Collects, analyzes, and correlates **security telemetry**.
This layer turns the homelab into a **mini SOC**.
#### Responsibilities
- Network monitoring
- Log aggregation
- Endpoint telemetry
- Detection and alerting
#### Core Outcomes
- IDS/IPS (Suricata)
- Network analysis (Zeek)
- SIEM (ELK / OpenSearch / Wazuh)
- Endpoint agents
- Honeypots
#### Why It Matters
This layer:
- Produces real security data
- Enables incident-style analysis
- Feeds the ML layer with meaningful inputs

---
### Layer 4 — Offensive & Adversary Simulation (Red Team)
#### Purpose
Generates **realistic attack data**.
#### Responsibilities
- Attack simulation
- Exploitation
- Lateral movement
- Malware behavior observation
#### Core Outcomes
- Kali / Parrot attacker systems
- Vulnerable Linux and Windows targets
- Active Directory attack lab
- Controlled adversary simulations
#### Why It Matters
Security data without attacks is meaningless.
This layer provides:
- Ground truth
- Labels for ML
- Realistic detection challenges

---
### Layer 5 — Data Science & Machine Learning
#### Purpose
Transforms raw telemetry into **insight and detection**.
This is where data science and cybersecurity intersect.
#### Responsibilities
- Data ingestion
- Feature engineering
- Model training
- Experiment tracking
- Inference deployment
#### Core Outcomes
- Central data storage (Postgres / TimescaleDB / object storage)
- ETL pipelines (Python)
- MLflow for experiments
- JupyterHub / VS Code Server
- Deployed ML inference APIs
- Dashboards showing ML-driven detections
- Self-hosted AI applications / models
#### Why It Matters
This layer is the homelab’s **differentiator**:
- Custom detections
- Behavioral models
- Real-time scoring
- Resume-grade ML + security integration

---
### Layer 6 — Observability & DevOps
#### Purpose
Ensures the homelab itself is **reliable, visible, and reproducible**.
#### Responsibilities
- Infrastructure monitoring
- Metrics and alerting
- Automation
- Configuration management
#### Core Outcomes
- Prometheus & Grafana
- Alerting
- Ansible automation
- Infrastructure-as-code where appropriate
#### Why It Matters
A complex system without observability becomes unmanageable.
This layer prevents the homelab from collapsing under its own weight.

---
### Layer 7 — Hybrid Cloud & External Integration
#### Purpose
Extends the homelab beyond the home network.
#### Responsibilities
- Cloud connectivity
- Hybrid deployments
- Comparative analysis (on-prem vs cloud)
#### Core Outcomes
- VPN peering to cloud providers
- Cloud-hosted services or attack nodes
- Terraform-managed cloud resources
#### Why It Matters
Most real environments are hybrid.
This layer adds realism and breadth.

---
### Personal & Self-Hosted Services (Cross-Cutting)
#### Philosophy
Personal services are **first-class citizens**, but never compromise:
- Security
- Segmentation
- Core learning goals
#### Examples
- Network-wide ad blocking (Pi-hole / AdGuard)
- Self-hosted AI chatbots
- Media servers
- Game servers
- Personal dashboards
- Home automation integrations
#### Placement Strategy
- Run in the **Services VLAN**
- Isolated from the Security Lab
- Managed like production services
- Logged and monitored where appropriate

---
### Core vs Stretch Goals
#### Core (Must-Have)
- Segmented network
- Firewall + switch
- Proxmox
- Security telemetry
- ML pipelines on real data
- Safe red/blue team interaction
#### Stretch (Add After Stabilization)
- Game servers
- Media platforms
- Advanced Kubernetes setups
- Multi-node clusters
- Experimental AI workloads
- Fancy dashboards and UX improvements

Stretch goals are added **only after**:
- Core layers are stable
- Documentation exists
- Recovery and backups are in place

---
### Final Perspective
This homelab is not:
- A random collection of servers
- A pile of self-hosted apps
- A single-purpose lab

It is:
- A **secure private cloud**
- A **mini SOC**
- A **data science platform**
- A **learning environment**
- A **personal infrastructure**

Built deliberately, layer by layer.

---
### Next Steps
- Each layer will have its own detailed documentation
- Implementation will follow milestones, not impulse
- Stretch features will be added without destabilizing the core

This document is the **architectural contract** for the homelab.
