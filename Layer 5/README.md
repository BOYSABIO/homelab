# Layer 5 — Data Science & Machine Learning

### Purpose

Layer 5 is where the homelab stops being pure infrastructure and starts doing something with data: ETL, model training, experiment tracking, and inference serving. Per the [[PROJECTS/Homelab/homelab/about|Architecture Overview]], this is the homelab's differentiator — custom models and real-time scoring running on real infrastructure, not a notebook on a laptop.

### Status

**In progress.** First component deployed June 2026.

| Component | Status |
|-----------|--------|
| Deployed ML inference API | ✅ [[Huckleberry-Habitat-Model\|Huckleberry Habitat Model]] — FastAPI `/predict`, Dockerized, running on a Proxmox LXC |
| MLflow (experiment tracking + model registry) | ✅ Running alongside the Huckleberry API, same LXC |
| Central data storage (Postgres / TimescaleDB / object storage) | ⬜ Not started |
| ETL pipelines | ⬜ Not started — Huckleberry's ETL runs outside the homelab, on the training host |
| JupyterHub / VS Code Server | ⬜ Not started |
| Dashboards for ML-driven detections | ⬜ Not started |

### Infrastructure dependency

Layer 5 doesn't run on its own hardware — it sits on top of [[PROJECTS/Homelab/homelab/Layer 1/README|Layer 1]]. Huckleberry specifically runs as CT 103, a Proxmox LXC on VLAN 30 (SERVICES), same as the webapp and LibreTranslate containers. Layer 1's [[Proxmox (Containers)\|Proxmox (Containers)]] doc covers the LXC-level settings; this doc covers everything above that (Docker Compose, MLflow, the app itself).

### Notes

Huckleberry is a feasibility-study model (habitat suitability prediction, not security telemetry) — it's not the SOC-style detection model envisioned for this layer in the original architecture doc. It's here because it's the first real "training happens elsewhere, serving happens on the homelab" pattern in this layer, and that pattern (MLflow registry + Dockerized API on an LXC) is exactly what later, more security-focused Layer 5 work will reuse.

### Documentation

| Document | What It Covers |
| -------- | -------------- |
| [[Huckleberry-Habitat-Model\|Huckleberry Habitat Model]] | Habitat suitability inference API + MLflow, deployed to a Proxmox LXC — settings, stack, update flow, troubleshooting |
