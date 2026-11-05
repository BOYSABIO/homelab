# MuttMetrics Server

> Always-on host for **MuttMetrics**, the capture app a groomer uses on his phone to log every groom ([BOYSABIO/muttmetrics](https://github.com/BOYSABIO/muttmetrics)). Built and tracked in **muttmetrics#93**. This page is the homelab side: placement, access, hardening and backups. The app-side runbooks live in the muttmetrics repo (`docs/runbooks/release.md`, `backup.md`, `groomer-trial.md`).
>
> No IPs, tailnet names or client data on this page (public repo). Real values live in the private command center.

**Status:** 🔧 setup in progress (2026-11). Debian is installed and untouched since the install.

---

## 1. Why it's here and not in the salon

The salon has no network, so the box lives at home and the groomer reaches it **only over Tailscale** from his phone on mobile data. This replaces the old setup, where the app ran on the main workstation and only existed while that PC was awake.

## 2. Hardware and OS

| | |
|---|---|
| Machine | A refurbished ex-office OptiPlex (one of the spares rescued from e-waste) with a new SSD |
| OS | Debian, minimal (no desktop) |
| Disk layout | LVM, **no full-disk encryption** (decision below) |
| BIOS | Power on after AC loss (to set) |

## 3. Placement: VLAN 30 (SERVICES), port 4, unmanaged switch

The box shares switch port 4 with the Vintage Story server through an unmanaged switch ([[Physical-Topology]]). Both are in VLAN 30.

- **Between VLANs** (for example, the workstation on TRUSTED reaching it over SSH): OPNsense routes and filters, as usual ([[Firewall-Rules]]).
- **Inside port 4** (Vintage Story server ↔ MuttMetrics server): the traffic never leaves the unmanaged switch, so OPNsense can't see or filter it. **The host firewall is the only control there.**

## 4. Access model

```
groomer's phone ──Tailscale──► muttmetrics node (the box's own Tailscale)
                                  │ tailscale serve, HTTPS 443
                                  ▼
                               Caddy on 127.0.0.1 ──/api/*, adds X-API-Key──► API on 127.0.0.1:8000
                                                                                   │
                                                                     Postgres (Docker) on 127.0.0.1:5432
maintainer ──SSH (key only)──► box   (from the TRUSTED workstation, or over Tailscale)
```

- The box runs **Tailscale itself** (node `muttmetrics`, tagged, **key expiry disabled**), so the groomer's access doesn't depend on OPNsense's subnet route.
- **Nothing listens on the LAN except SSH.** The app, API and database bind to localhost. **No port forwarding, ever.**
- **HTTPS via `tailscale serve`** (decided 2026-11-05). WireGuard already encrypts the traffic; HTTPS adds a real certificate, no browser warning, and a secure context (needed for a future home-screen app). Trade-off accepted: the certificate's name (`muttmetrics.<tailnet>.ts.net`) appears in public Certificate Transparency logs. That reveals the name, not access: `*.ts.net` machine names only resolve inside the tailnet, to 100.x addresses that aren't routable on the internet.

## 5. Hardening checklist

**Host**
- [ ] Non-root sudo user; SSH **keys only** (`PasswordAuthentication no`, `PermitRootLogin no`)
- [ ] Updates: `unattended-upgrades` for security patches
- [ ] Timezone `Europe/Berlin` (backup timer and logs in local time)
- [ ] nftables, default **drop** inbound:
  - allow established/related;
  - allow `lo` and `tailscale0`;
  - allow SSH only from the workstation's address;
  - drop everything else, **including from the Vintage Story server**
- [ ] DHCP static mapping in VLAN 30 (infrastructure range .2–.20) ([[DHCP]])

**Switch**
- [x] Port 4 PVID = 30 confirmed on the Port Config page (2026-11-05)
- [ ] Remove port 4 (then the other access ports) from VLAN1, and turn ingress checking on for port 8 ([[Physical-Topology]] § VLAN1). Test with a VLAN-1 subinterface from this box, before and after

**Tailscale policy**
- [ ] **Before turning on HTTPS certificates:** check the tailnet DNS name (admin → DNS). Every certificate issued is in public logs **forever**, so pick a neutral tailnet name first. Only give certificates to machines whose names you don't mind being public
- [ ] Grant: groomer group → `tag:muttmetrics:443` only; the owner keeps admin
- [ ] **Narrow the existing `autogroup:member → VLAN 30 /24` rule** to the Vintage Story server's address and game port. Today any tailnet member can reach *every* VLAN 30 host through the OPNsense subnet route, which would include this box's SSH. The groomer doesn't need that route at all: he reaches the box through its own Tailscale node
- [ ] Policy tests:
  - groomer → `tag:muttmetrics:443` allowed;
  - groomer → `:22` and → the box's LAN address denied;
  - members → the Vintage Story server's game port still allowed
- [ ] Note the expiry date of the groomer's phone key (a phone that quietly logs out looks like "the app is down")

## 6. Data at rest and backups

- **No full-disk encryption on the box, on purpose.** Encrypting the system disk means typing a passphrase after every power cut, and the point of this box is that it comes back by itself. (Possible later: a TPM-unlocked encrypted volume for the data only. That protects a removed or discarded SSD, not a stolen machine.)
- Nightly `ops/backup.py` (DB dump + photos) on a systemd timer, as an unprivileged user.
- Second copy **outside the house** on a **LUKS-encrypted** external drive: the copy that travels is the one that gets encrypted. **No cloud.**
- Restore drill at setup, then on a fixed cadence (dates are logged privately).

## 7. Decision log

| Date | Decision |
|---|---|
| 2026-10-21 | Box at home, not in the salon (no network there); groomer reaches it over Tailscale only |
| 2026-11-05 | VLAN 30, switch port 4 via an unmanaged switch shared with the Vintage Story server; host firewall does the separating |
| 2026-11-05 | HTTPS with `tailscale serve`, Caddy on localhost; the CT-log name exposure is accepted |
| 2026-11-05 | No full-disk encryption (unattended reboot wins); the out-of-house backup drive is LUKS-encrypted |
