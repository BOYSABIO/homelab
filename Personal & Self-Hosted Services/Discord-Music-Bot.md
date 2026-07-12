# Discord Music Bot

**Stack:** Node.js (TypeScript) · discord.js · Riffy · Lavalink v4 · Docker Compose  
**Host:** Proxmox LXC CT 104 · `10.x.x.14` · VLAN 30 (SERVICES)  
**Repo:** https://github.com/BOYSABIO/discord-music-bot

---

## What It Is

Self-hosted Discord music bot for a private friends-and-family server. Vibe-coded. Uses a message-command prefix (`-`) rather than Discord slash commands, so no slash-command registration is needed.

Two Docker Compose services run together on CT 104:

- **bot** — Node.js/TypeScript process. Listens for messages starting with `-`, resolves audio tracks through Riffy, and controls Lavalink via the REST/WebSocket API.
- **lavalink** — Lavalink v4 (Java-based audio engine). Handles track searching, audio streaming, queue buffering, and voice-channel audio. Uses a custom Dockerfile that extends the base Lavalink image to bundle `yt-dlp`.

Traffic between the two containers is internal to the Docker bridge network (`discord_music_net`). **No inbound ports are exposed** — the bot makes outbound connections to Discord's API (TCP 443) and Discord voice (UDP), both from VLAN 30 through OPNsense.

---

## Architecture

```
Discord API (HTTPS 443 / UDP voice)
         ↕
  discord-music-bot  (Node.js/TS, ~50–100 MB)
         ↕  Docker internal network
  discord-music-lavalink  (Lavalink v4 + yt-dlp, ~512 MB–1 GB JVM)
```

**Audio sources Lavalink supports on this deploy:**

| Source | How |
|--------|-----|
| YouTube (search + URLs + playlists) | youtube-plugin v1.18.1 + yt-dlp |
| YouTube Music search | ytmsearch via LavaSrc |
| SoundCloud | Native Lavalink source |
| Bandcamp | Native Lavalink source |
| Twitch | Native Lavalink source |
| Vimeo | Native Lavalink source |
| Spotify (optional) | LavaSrc v4.8.1 — mirrors to YouTube/yt-dlp. Off by default, enable via `.env` |

The default search platform is `ytmsearch` (YouTube Music). Text searches like `-play never gonna give you up` go through YouTube Music first, then fall back to yt-dlp.

**Resource allocation on CT 104:** 2 cores / 2048 MiB RAM / 512 MiB swap / 8 GB disk. Lavalink's JVM is the memory consumer; the bot process itself is lightweight.

---

## Commands

All commands use the `-` prefix (e.g. `-play`). The bot requires **Message Content Intent** to be enabled in the Discord Developer Portal — without it, it receives no messages and silently ignores every command.

| Command | Description |
|---------|-------------|
| `-play <url or search>` | Add a YouTube/SoundCloud URL or text search to the queue. Playlists are fully supported — all tracks are enqueued at once. |
| `-pause` | Pause playback |
| `-resume` | Resume paused playback |
| `-skip` | Skip the current track and play the next one |
| `-stop` | Stop playback, clear the queue, and disconnect from the voice channel |
| `-queue` | Show the next 10 tracks in the queue (and total count) |
| `-nowplaying` | Show the current track title and duration |

**Bot behavior:**
- Replies in the text channel the command was sent from
- Posts `Now playing: **<title>**` when a new track starts
- Posts `Queue finished.` and disconnects when the queue empties
- If you run `-play` while already in a different voice channel than the bot, the bot moves to your channel
- Only one instance should run — two instances means every command is replied to twice

---

## Deployment

Full LXC creation, Docker install, clone, and `.env` setup: [[PROJECTS/discord-music-bot/docs/PROXMOX_DEPLOY|PROXMOX_DEPLOY.md]]

**LXC settings summary:**

| Setting | Value |
|---------|-------|
| CT ID | 104 |
| Hostname | `discord-bot` |
| Template | Ubuntu 24.04 |
| CPU / RAM / Swap | 2 cores / 2048 MiB / 512 MiB |
| Disk | 8 GB |
| VLAN | 30 (SERVICES) |
| IP | `10.x.x.14/24` |
| Gateway | `10.x.x.1` |
| Features | `nesting=1`, `keyctl=1` (required for Docker) |
| Proxmox firewall | Disabled (OPNsense handles policy) |

**Key Proxmox-specific step:** comment out the `ports:` block under `lavalink` in `docker-compose.yml` after cloning. On Proxmox, Lavalink must stay on the internal Docker network only — the port doesn't need to be exposed to the LXC's network interface.

App directory on CT 104: `/opt/discord-music-bot`

---

## Day-to-Day Operations

SSH into CT 104 first:
```bash
ssh root@10.x.x.14
cd /opt/discord-music-bot
```

### Check status
```bash
docker compose ps
```
Both `discord-music-bot` and `discord-music-lavalink` should show `Up`.

### View live logs
```bash
# Bot logs (login confirmation, command errors, track events)
docker compose logs -f bot

# Lavalink logs (track loading, plugin errors, voice connection)
docker compose logs -f lavalink
```

Expected healthy bot startup:
```
Logged in as YourBot#1234
Lavalink node "main" connected.
```

### Update to latest
```bash
git pull
docker compose up -d --build
```

### Restart bot only (no rebuild)
```bash
docker compose restart bot
```

### Restart after `.env` change
```bash
docker compose up -d --force-recreate bot
```

### Full stop
```bash
docker compose down
```

Both services use `restart: unless-stopped` — they come back automatically after the LXC reboots.

---

## Configuration

`.env` lives at `/opt/discord-music-bot/.env` on CT 104. It is gitignored — never committed. Must be created manually after cloning.

| Variable | Required | Notes |
|----------|----------|-------|
| `DISCORD_TOKEN` | ✅ | Bot token from Discord Developer Portal |
| `LAVALINK_PASSWORD` | Recommended | Shared secret between bot and Lavalink. Generate: `openssl rand -base64 24` |
| `SPOTIFY_ENABLED` | Optional | Set `true` to enable Spotify mirroring via LavaSrc |
| `SPOTIFY_CLIENT_ID` | If Spotify on | From Spotify Developer Dashboard |
| `SPOTIFY_CLIENT_SECRET` | If Spotify on | From Spotify Developer Dashboard |

`LAVALINK_HOST`, `LAVALINK_PORT`, and `LAVALINK_SECURE` are injected by `docker-compose.yml` automatically — do not set them in `.env` for the Proxmox deploy.

**Lavalink plugin versions** (in `lavalink/application.yml`, committed to repo):
- `youtube-plugin` 1.18.1
- `lavasrc-plugin` 4.8.1

These need periodic bumping when YouTube breaks — see Updating section below.

---

## Updating Lavalink Plugins

YouTube's API changes break the plugin periodically. Symptom: tracks fail to load, `-play` returns "No results found" even for known-good URLs.

Fix: bump the plugin version in `lavalink/application.yml`:

```yaml
lavalink:
  plugins:
    - dependency: "dev.lavalink.youtube:youtube-plugin:<new-version>"
      snapshot: false
    - dependency: "com.github.topi314.lavasrc:lavasrc-plugin:<new-version>"
      repository: "https://maven.lavalink.dev/releases"
      snapshot: false
```

Then rebuild just Lavalink (no need to touch the bot):
```bash
cd /opt/discord-music-bot
git pull   # if the version bump was committed
docker compose up -d --build lavalink
```

Releases:
- youtube-plugin: https://github.com/lavalink-devs/youtube-source/releases
- lavasrc-plugin: https://github.com/topi314/LavaSrc/releases

---

## Troubleshooting

**Bot doesn't respond to any commands**
1. `docker compose ps` — both containers should be `Up`
2. `docker compose logs -f bot` — check for `Logged in as` and `Lavalink node "main" connected`
3. In Discord Developer Portal → Bot → check that **Message Content Intent** is enabled
4. Make sure you're using the `-` prefix, not `/`

**Tracks fail to load / "No results found"**
YouTube broke again. Bump the plugin version — see Updating section above.

**Bot joins the voice channel but audio is silent**
OPNsense is blocking UDP egress from VLAN 30. Discord voice uses UDP for audio — check that outbound UDP (established/related) from VLAN 30 is allowed in OPNsense.

**Bot replies to every command twice**
Two bot containers are running simultaneously. `docker compose ps` to confirm. `docker compose down`, then `docker compose up -d`.

**`.env` not loading / password mismatch after copy from Windows**
Windows adds `\r` line endings. Fix before starting:
```bash
sed -i 's/\r/\n/g' .env
file .env   # must NOT say "CRLF" or "CR line terminators"
docker compose up -d --force-recreate
```

Full troubleshooting guide: [[PROJECTS/discord-music-bot/docs/TROUBLESHOOTING|TROUBLESHOOTING.md]]

---

## Network Requirements

| Direction | Protocol | Destination | Purpose |
|-----------|----------|-------------|---------|
| Outbound | TCP 443 | Discord API | Bot connection, commands |
| Outbound | UDP (established/related) | Discord voice servers | Audio streaming |
| Inbound | None | — | No inbound ports needed |

OPNsense rule: allow outbound HTTPS + established/related UDP from VLAN 30. No port forwarding or inbound rules required.
