# MuttMetrics Server

> Always-on host for **MuttMetrics**, the capture app a groomer uses on his phone to log every groom ([BOYSABIO/muttmetrics](https://github.com/BOYSABIO/muttmetrics), built in muttmetrics#93). This page is the **homelab side**: placement, network access, how it fits next to the other servers. **The full rebuild procedure, with a check after every step, lives in the app repo:** [`docs/runbooks/home-server.md`](https://github.com/BOYSABIO/muttmetrics/blob/main/docs/runbooks/home-server.md), with every config file in [`ops/server/`](https://github.com/BOYSABIO/muttmetrics/tree/main/ops/server).
>
> No IPs, tailnet names or client data on this page (public repo). Real values live in the private command center.

**Status:** ✅ live since 2026-11-06. Real data was moved in, the groomer's phone reaches it by name, and backups run nightly. The groomer's first real visit from the salon is the last acceptance check.

---

## 1. Why it's here and not in the salon

The salon has no network, so the box lives at home and the groomer reaches it **only over Tailscale** from his phone on mobile data. Before, the app ran on the main workstation and only existed while that PC was awake.

## 2. Hardware and OS

| | |
|---|---|
| Machine | Dell OptiPlex 790 (2011, ex-office, rescued from e-waste), runs from a 1 TB SSD over **USB** (its only drive) |
| OS | Debian 13 "trixie", minimal; LVM; **no full-disk encryption** (it has to reboot unattended) |
| Power cut | The CMOS battery is dead, so after a real power cut someone presses the button (muttmetrics#217, icebox). Normal reboots, including the 04:30 update reboot, come back by themselves |
| Storage follow-ups | Internal SATA instead of USB; a NAS as a backup target (muttmetrics#220, icebox) |

## 3. Placement: VLAN 30 (SERVICES), port 4, unmanaged switch

The box shares switch port 4 with the Vintage Story server through an unmanaged switch ([[Physical-Topology]]). Both are in VLAN 30.

- **Between VLANs** (the workstation on TRUSTED reaching it over SSH): OPNsense routes and filters, as usual ([[Firewall-Rules]]).
- **Inside port 4** (Vintage Story server ↔ MuttMetrics server): the traffic never leaves the unmanaged switch, so OPNsense can't see or filter it. **Each host's own firewall is the only control there.** This one runs nftables with a default drop; since the Vintage Story server was tagged ([[Vintage-Story-Server]]), it can't reach this box through Tailscale either.

## 4. Access model

```
groomer's phone ──Tailscale──► muttmetrics node (the box's own Tailscale, tag:muttmetrics)
                                  │ tailscale serve: HTTPS 443, tailnet only (Funnel off)
                                  ▼
                               Caddy 127.0.0.1 ──/api/*, adds the API key──► API 127.0.0.1:8000
                                                                                   │
                                                                     Postgres (Docker) 127.0.0.1:5432
maintainer ──SSH (key only)──► box   (by name over Tailscale; or from the workstation on the LAN)
```

- **The box runs Tailscale itself** (node `muttmetrics`, tagged, so key expiry is off). The groomer's access doesn't depend on OPNsense's subnet route.
- **Nothing listens on the LAN except SSH.** App, API and database are on localhost. **No port forwarding, ever.**
- **HTTPS via `tailscale serve`:** a real certificate, no browser warning, a secure context. WireGuard already encrypts the traffic either way. Accepted: the name `muttmetrics.<tailnet>.ts.net` is in public Certificate Transparency logs. That reveals the name, not a way in.
- **Tailnet policy** (pattern in [[Tailscale]] §7): the groomer's group → `tag:muttmetrics` on `tcp:443` only, with a policy test (accept 443; deny 22/8000/5432). His phone has key expiry disabled.

## 5. Hardening (done 2026-11-05/06)

- [x] Updates: `unattended-upgrades`, auto-reboot at 04:30 when needed; timezone `Europe/Berlin`
- [x] DHCP reservation in the infrastructure range (.2–.20), matched on MAC only
- [x] Admin account with sudo; SSH **keys only**, one per device; no root; `AllowUsers` = the admin. The app account `muttmetrics` has no SSH and no general sudo (only `systemctl restart muttmetrics-api caddy`)
- [x] nftables in its own table (`inet host_fw`, no `flush ruleset`, so Tailscale's and Docker's tables survive a reload). Default drop: Tailscale in, SSH from the workstation, ICMP/ND/DHCP; everything else dropped and logged (`host_fw drop:`). Applied with a 5-minute automatic rollback as a safety net. Verified: subnet-routed SSH arrives as the router's address and is dropped
- [x] Docker: Postgres published on `127.0.0.1` only (published ports bypass the host firewall, so the binding is what protects it); rotating logs; `live-restore`
- [x] API as a sandboxed systemd service (`ProtectSystem=strict`, writes only to its data folder); restarts itself
- [x] Tailnet: tag, grant, test, the groomer's phone expiry; existing grants reviewed (the broad "members → all of VLAN 30" rule was replaced by per-server tags on 2026-11-06)
- [ ] Switch hygiene for port 4 (tracked privately; goes public once done)

## 6. Data and backups

- **No full-disk encryption on the box, on purpose** (power-cut recovery wins). A TPM-unlocked data volume isn't an option: the 790 predates TPM 2.0, and the volume group has no free space.
- Nightly backup at 00:30 (DB dump + photo copy) as the app account, with a check afterwards; a failed run shows as a failed systemd unit. A restore drill passed with real data on 2026-11-06.
- **Same disk** as the live data: covers mistakes, not a dead SSD or the house. The second copy is a **LUKS-encrypted drive kept outside the house** (muttmetrics#221, needs a drive). **No cloud.**

## 7. Decision log

| Date | Decision |
|---|---|
| 2026-10-21 | Box at home, not in the salon (no network there); the groomer reaches it over Tailscale only |
| 2026-11-05 | VLAN 30, switch port 4 via an unmanaged switch shared with the Vintage Story server; host firewall does the separating |
| 2026-11-05 | HTTPS with `tailscale serve`, Caddy on localhost; the CT-log name exposure is accepted; Funnel stays off |
| 2026-11-05 | No full-disk encryption (unattended reboot wins); the off-site backup drive is LUKS-encrypted |
| 2026-11-05 | Two accounts: an admin (key + sudo) and the app account (no SSH, one restart command) |
| 2026-11-06 | Cut over from the workstation; the groomer's phone switched to `https://` by name |
| 2026-11-06 | Vintage Story server tagged too; per-server grants replace "members → all of VLAN 30" ([[Tailscale]] §7) |
