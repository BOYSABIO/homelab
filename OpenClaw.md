# OpenClaw — Lab Setup & Learning Notes
## Scope and intent

| Aspect | What this setup is |
| ------ | ------------------ |
| **Purpose** | Learn OpenClaw capabilities, bootstrap identity and workflows, and use the workspace as a small **orchestration/planning hub** for future OpenClaw work—not the final “production home” for every domain. |
| **Placement** | Runs **mainly locally** on a workstation; not wired into the full Layer 0 segmentation story yet. |
| **Security posture** | **Deliberately relaxed** for learning: not modeled as multi-tenant, not held to the same bar as a segmented lab service. Treat it as a **trusted single-user assistant environment** until you intentionally harden or relocate it (for example a dedicated VM or stricter network placement). |

**Future separations you already had in mind** (not implemented here): career/job search, finance, cybersecurity (possibly a separate VM or OpenClaw instance later), and project-specific coding or data science in dedicated workspaces.

---
## Mental model: what this workspace is for

- **This workspace** → orchestration, planning, and learning how OpenClaw behaves (identity, channels, Control UI, memory, audits).
- **Later workspaces** → where heavier, domain-specific execution is expected to live.

That keeps the lab instance from trying to be everything at once while you still learn the platform.

---
## Identity bootstrap

Established a clear **assistant identity** and **human context** so behavior stays consistent across sessions.
### Assistant identity

| Field | Value                                                                                                          |
| ----- | -------------------------------------------------------------------------------------------------------------- |
| Name  | k*********                                                                                                     |
| Emoji | 🔒                                                                                                             |
| Style | Brutally honest, truth-seeking, no-nonsense, teaching-first, prioritizing understanding over shallow agreement |

### Human context

| Field       | Value                                                                                                                                                                                                   |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name        | Sabio                                                                                                                                                                                                   |
| Timezone    | Europe/Berlin                                                                                                                                                                                           |
| Preferences | Directness over flattery; concise answer first, then reasoning; pushback on weak plans; proactive in-repo notes/memory/docs; **explicit permission before actions outside the local machine/workspace** |

### Files created or updated during bootstrap
Typical OpenClaw workspace files touched in that phase:
- `IDENTITY.md`, `USER.md`, `SOUL.md`
- `memory/YEAR-MON-DAY.md` (daily memory)
- `AGENTS.md`, `TOOLS.md`, `OPENCLAW_NOTES.md`, `MEMORY.md`

*(Paths are relative to the OpenClaw workspace, not necessarily this homelab git repo.)*

### Git history in the OpenClaw workspace
Commits recorded during setup (in the **workspace** repository, for your own archaeology):
- Bootstrap identity and user profile  
- Remove bootstrap file after setup  
- Document workspace role and behavioral defaults  
- Add OpenClaw local admin cheat sheet  

`BOOTSTRAP.md` was moved to `.trash/BOOTSTRAP.md` rather than hard-deleted. Local git identity on that machine was set for workspace commits (user name aligned with the assistant identity).

---
## Behavioral calibration (operational contract)

These rules were written down so the assistant’s behavior matches how you want to work:
1. **Challenge bad plans** with reasoning—do not only execute.  
2. **Be proactive** with in-workspace notes, memory, and documentation.  
3. **Ask before** external or off-machine actions.  
4. **Answer concise-first**, then expand with reasoning.  
5. **Do not store secrets** or raw sensitive infrastructure detail in notes/memory unless strictly necessary—prefer redaction in docs.

---
## Telegram channel
1. Generated a **Telegram pairing code** in the OpenClaw flow for Telegram user.  
2. Approved pairing with the CLI so that identity could talk to **this** OpenClaw instance.

General form:
```bash
openclaw pairing approve telegram <PAIRING_CODE>
```

Use the code shown by your setup UI or CLI at pairing time; **do not commit pairing codes or reuse old ones** in documentation.

### Result
- Telegram was authorized for the account you paired.  
- Messaging this OpenClaw instance was verified end-to-end.

---
## Platform health check and config audit
Ran OpenClaw’s status/audit tooling and inspected `~/.openclaw/openclaw.json` (or the equivalent config path on your OS).

### Findings (summary)

| Topic                        | Observation                                                                                                         |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Gateway                      | Healthy and reachable                                                                                               |
| Telegram                     | Configured and working                                                                                              |
| Tailscale                    | Off (not used for remote access in this phase)                                                                      |
| Version                      | OpenClaw reported up to date                                                                                        |
| Control UI                   | **Insecure auth was on** at first (see hardening below)                                                             |
| `gateway.nodes.denyCommands` | Some entries were **not effective** as intended—worth revisiting                                                    |
| Trust model                  | Warning about **multi-user trust mismatch**—consistent with using this as a personal setup, not shared multi-tenant |
| Builtin memory               | Initially looked **unavailable or uninitialized** until deeper dive (see memory section)                            |

### Security interpretation (how to think about it)
- **Alone**, on a trusted machine, this is a reasonable **personal assistant** setup.  
- It is **not** a design where untrusted users or untrusted networks should hit the same instance.  
- Anything that exposes the gateway or Control UI beyond localhost deserves a separate threat-model pass (Tailscale, reverse proxy, etc.).

---
## Control UI: hardening step
### Change
`gateway.controlUi.allowInsecureAuth` was set from **`true` → `false`**.

### Why
- With insecure auth enabled, the local dashboard is easier to open but weaker from an audit perspective.  
- Disabling it aligns the Control UI with **token-based** access and clears part of the security audit noise.

### Effect you saw
- Security audit warnings dropped (e.g. from five to three in run—exact counts depend on version and config).  
- The dashboard still worked when the **gateway token** was supplied.  
- Local access now **requires** that token instead of anonymous/insecure access.

### How to open the Control UI
1. Get the local URL and gateway state:
   ```bash
   openclaw status
   ```
2. Typical local URL shape:
   ```text
   http://127.0.0.1:18789/
   ```
3. If the UI shows **unauthorized: gateway token missing**, the service is behaving as expected under stricter auth—supply the token.
4. Token location (local machine only—**never commit this**):
   - File: `~/.openclaw/openclaw.json` (path may vary by OS)  
   - Field: `gateway.auth.token`
Example inspection (Unix-style shell):
```bash
grep -n '"token"' ~/.openclaw/openclaw.json
```
Paste the token into the dashboard when prompted. 

---
## Builtin memory system
Spent time on **built in OpenClaw memory** because early status looked broken or empty.

### Symptoms
- Memory seemed enabled but not really “there.”  
- `memory/YEAR-MON-DAY.md` existed with real content, and scans could see files, but **`openclaw memory search`** found nothing useful.  
- `openclaw memory status` showed **0 indexed files / 0 chunks**.

### Commands that helped
```bash
openclaw memory --help
openclaw memory status
openclaw memory status --deep
openclaw memory status --json
openclaw memory index --force
openclaw memory index --agent main --verbose
openclaw memory search "Your Name"
openclaw memory search "Bot Name"
openclaw memory search "Telegram"
```

### Root cause (verbose indexing)
Verbose output showed something like:
> Skipping memory file sync in FTS-only mode (no embedding provider)

### Interpretation
- **Codex OAuth alone** does not give a working **embedding** stack for memory path.  
- Without a **supported embedding provider**, the pipeline effectively skips the behavior expected for full semantic memory.  
- **FTS-only** mode does not match the “everything is searchable the way I imagined” mental model.

### Practical conclusion (as of this setup)
- **Authoritative memory for daily work** remains the **plain files** you maintain: `memory/YYYY-MM-DD.md`, `MEMORY.md`, `USER.md`, `AGENTS.md`, and other repo docs.  
- **Builtin memory search** becomes a priority again when you add a **supported embedding provider** and re-index.

---
## Operational reference (maintenance)
### Pairing (any channel)
```bash
openclaw pairing approve <channel> <PAIRING_CODE>
```
Pair again when you add a new account, channel, or OpenClaw instance.

### Health, security, gateway, memory
```bash
openclaw status
openclaw security audit
openclaw security audit --deep
openclaw update status
openclaw gateway status
openclaw gateway restart
openclaw memory status
openclaw memory status --deep
openclaw memory index --force
```

---

## Current baseline (after this documentation was written)
- Assistant identity and user preferences are defined and documented in the workspace.  
- Telegram works for this instance.  
- Control UI works with **stricter token auth**.  
- Workspace role is clear: **orchestrator / learning hub**, not all execution.  
- Daily and long-form file-based memory practices exist.  
- Builtin memory **search** remains limited until embeddings are configured.  
- Tailscale and stronger network placement are **future** items, not required for the initial learning loop.

---
## Recommended next steps (when you want them)
- Add model/provider options and optional **cheap vs strong** routing.  
- Spin **additional workspaces by domain** as work splits out.  
- Consider a **separate OpenClaw instance or VM** for security-sensitive workflows.  
- Enable **Tailscale** (or similar) with a deliberate ACL model when you need remote access.  
- Revisit **builtin memory search** after configuring a **supported embedding provider** and re-running indexing.



