# Huckleberry Habitat Model — ML Inference API + MLflow

**Container:** Proxmox LXC | **VLAN:** 30 (SERVICES) | **Deployed:** June 2026

The Huckleberry habitat suitability model (Microsoft capstone — see `PROJECTS/Huckleberry-Habitat-Suitability-Model/`) moved from a notebook/model artifact into deployed infrastructure: a Dockerized FastAPI inference service backed by an MLflow model registry, running on its own Proxmox LXC. This is the first piece of **Layer 5 — Data Science & Machine Learning** to actually exist (see [[PROJECTS/Homelab/homelab/Layer 5/README|Layer 5 README]]).

Source of truth for the application itself (API code, model, training pipeline) is the project's own repo: `PROJECTS/Huckleberry-Habitat-Suitability-Model/Capstone-Microsoft/` (GitHub: `BOYSABIO/Capstone-Microsoft`). The full step-by-step deploy guide lives there at `docs/homelab-deploy.md` — this page is the homelab-side record of what's actually running, how it sits in the network, and what broke during the real deploy.

---

## What's running

```text
[SERVICES VLAN — 10.zz.zz.0/24]
    │
    ├── http://[CT103-IP]:8000  →  FastAPI  (/predict, /health, /docs)
    └── http://[CT103-IP]:5000  →  MLflow UI (model registry, @production alias)
```

Two Docker Compose services on one LXC:

| Service | Image / build | Port | Role |
|---------|---------------|------|------|
| `mlflow` | `python:3.11-slim` + `pip install mlflow` at container start | 5000 | Tracking server + model registry, SQLite backend (`mlflow.db`), artifacts on a mounted volume (`mlartifacts/`) |
| `api` | built from the repo's `Dockerfile` | 8000 | FastAPI, loads the model tagged `@production` from MLflow at startup, serves `/predict` and `/health` |

The API does **not** bake a model into the image — it loads `models:/huckleberry-habitat@production` from MLflow on boot. Training does not happen on this container; it runs on the dev machine (or via SSH on the LXC with a full Python env), and only the resulting `mlflow.db` + `mlartifacts/` get copied over.

---

## LXC settings

| Setting | Value |
|---------|-------|
| Template | Ubuntu 24.04 (Debian 12 also works) |
| CPU | 2 cores |
| RAM | 4096 MiB (3072 MiB is the bare minimum — MLflow + API together are tight on 3 GB) |
| Disk | 20 GB+ |
| Network | vmbr0, VLAN 30 (SERVICES), static LAN IP |
| Unprivileged | ✅ Yes |
| Nesting | ✅ Yes — required for Docker inside the LXC |
| Firewall | ❌ Disabled (same reasoning as CT 100 / CT 101 — see [[Proxmox (Containers)|Proxmox (Containers)]]) |

Same container-firewall caveat applies as the other two SERVICES containers: leave the Proxmox per-container firewall off, OPNsense already enforces VLAN policy, and the firewall bridge has historically broken VLAN tagging on this setup.

---

## Getting it running

1. Docker installed inside the LXC (`docker-ce`, `docker-compose-plugin`, standard Docker apt repo).
2. Repo cloned to the LXC: `git clone https://github.com/BOYSABIO/Capstone-Microsoft.git`.
3. **Seeded the model store** — `mlflow.db` and `mlartifacts/` are gitignored (they contain the trained model and registry metadata), so they don't come from `git clone`. Copied from the dev machine via `scp`. Confirmed `huckleberry-habitat` had the `production` alias set in MLflow before relying on the API to load it.
4. Started via the repo's helper script rather than a bare `docker compose up`:

   ```bash
   chmod +x scripts/homelab_start.sh scripts/homelab_update.sh
   ./scripts/homelab_start.sh
   ```

   This starts MLflow first, polls `http://127.0.0.1:5000/` until it returns 200, *then* starts the API. Necessary because the `mlflow` service installs MLflow via `pip` on every cold start (no pre-built image) — on this hardware that took the full **15–20 minutes** on first boot. Starting both services at once races the API against an MLflow server that isn't listening yet.
5. Smoke-tested locally on the LXC:

   ```bash
   curl -s http://localhost:8000/health
   curl -s -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d '{...12 feature fields...}'
   ```

   Full request schema is in the project README's REST API section.

---

## LAN access

- **API (`:8000`)** — works from any machine on the LAN with no extra config: `http://[CT103-IP]:8000/docs`.
- **MLflow UI (`:5000`)** — MLflow 3.x rejects unrecognized `Host` headers, so browsing from another machine on the LAN throws `Invalid Host header` until the LXC's own LAN address is added to `MLFLOW_ALLOWED_HOSTS` in a **local `.env` on the LXC** (copied from `.env.example`, never committed — same private-IP discipline as the rest of this repo). Restart with `docker compose up -d mlflow` after editing. SSH tunneling (`ssh -L 5000:127.0.0.1:5000 ...`) is the no-config alternative.
- Proxmox host firewall / LXC `ufw` (if enabled) need 8000 and 5000 open to the LXC. Ports are LAN-only — not port-forwarded to the internet.

---

## Updating

**Code changes** (after merging to `main` on the project repo):

```bash
cd ~/Capstone-Microsoft
./scripts/homelab_update.sh
```

Stops the stack, `git pull`s, rebuilds the API image, restarts MLflow-then-API in order.

**New model only** (no code change): set the new version's alias to `production` in the MLflow UI, then `docker compose restart api`. No rebuild, no `git pull` needed — the API re-reads the registry pointer on restart.

---

## Troubleshooting / lessons learned

| Symptom | Cause / fix |
|---------|-------------|
| MLflow `curl :5000` → `000` for 15+ minutes | Normal on this hardware — the container is running `pip install mlflow` from scratch every cold start, not loading a pre-built image. Wait it out; don't restart the stack mid-install. |
| API exits immediately at startup | `mlflow.db` / `mlartifacts/` missing (forgot the `scp` step) or no `@production` alias set on `huckleberry-habitat` yet. |
| `Child process died` in MLflow logs | Out-of-memory — MLflow + API together are tight on 3 GB. Bumped the LXC to 4 GB RAM; compose already runs `--workers 1` to keep the footprint down. |
| `Invalid Host header` opening MLflow UI from another LAN machine | MLflow 3.x Host-header allowlist — add the LXC's LAN IP to `MLFLOW_ALLOWED_HOSTS` in a local `.env` on the LXC (§ LAN access above). |
| Model artifacts trained on Windows fail to load on the Linux container | Windows logs artifact paths as `file:///C:/...`; `scripts/mlflow_docker_prepare.py` runs automatically before MLflow starts to rewrite these for Linux — already baked into the compose command, no manual step needed. |
| API serving a stale model after promoting a new version | `docker compose restart api` — the API only re-reads the `@production` pointer on startup, not live. |

The big general lesson carried over from the webapp/LibreTranslate deploys: don't trust a single `docker compose up` to sequence dependent services correctly on slow/low-RAM homelab hardware, even with `depends_on: condition: service_healthy` — the healthcheck's `start_period` has to be generous (this stack uses 900s), and a polling helper script is more debuggable than relying on Compose's own retry logic when something is this slow to boot.

---

## Optional: run on boot

A systemd unit wrapping `homelab_start.sh` / `docker compose down` is documented in the project's own `docs/homelab-deploy.md` (§7) — not yet enabled on this LXC.

---

## See also

- [[PROJECTS/Homelab/homelab/Layer 5/README|Layer 5 README]] — where this fits in the broader Data Science & ML layer
- [[PROJECTS/Homelab/homelab/Layer 1/README|Layer 1 README]] — the Proxmox/LXC substrate this runs on, and how CT 103 relates to CT 100/101
- [[Proxmox (Containers)|Proxmox (Containers)]] — CT 100 / CT 101 / CT 103 LXC-level setup, same VLAN, same firewall caveats
- `PROJECTS/Huckleberry-Habitat-Suitability-Model/Capstone-Microsoft/docs/homelab-deploy.md` — full step-by-step deploy guide (Issue #11)
- `PROJECTS/Huckleberry-Habitat-Suitability-Model/Capstone-Microsoft/FUTURE_TASKS.md` — architecture (serving vs. training, update flows) and planned homelab CD automation
