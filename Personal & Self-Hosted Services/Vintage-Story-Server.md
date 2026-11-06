# Vintage Story Dedicated Server — Admin Guide
For: Ubuntu Server (v1.22+), Tailscale networking, `.vcdbs` world saves  

**Active world:** `hermitvalley.vcdbs`  
**Game port:** `42420`  

---
## Server Paths (Important)
Essentially where everything is located on the machine.

**Server program files:**  
```bash
/home/vintagestory/server
```

**Worlds, configs, logs:**  
```bash
/var/vintagestory/data
```

**World save files (.vcdbs):**  
```bash
/var/vintagestory/data/Saves
```

**Main config file:**  
```bash
/var/vintagestory/data/serverconfig.json
```

---
## Starting, Stopping, Restarting the Server
Run these from anywhere:
### Start
```bash
sudo /home/vintagestory/server/server.sh start
```
### Stop
```bash
sudo /home/vintagestory/server/server.sh stop
```
### Restart
```bash
sudo /home/vintagestory/server/server.sh restart
```
### Status
```bash
sudo /home/vintagestory/server/server.sh status
```

### Send a server command (example: whitelist off)
```bash
sudo /home/vintagestory/server/server.sh command serverconfig whitelistmode off
```

---
## Share Server With Friends (Two Methods)
### Method 1 - Machine Invite Link
1. Open machines page in admin console
2. Find `vsserver` machine, open **menu** and press **share**
3. Two options:
	1. **Share by email**: Enter each friend's email address and select **Share** to send individual invite links.
	2. **Share by link**: Select **Copy invite link**, optionally toggle **Reusable link** (usable up to 1,000 times), then copy and send the link to friends.
4. Wait for each friend to accept the invite. Once accepted, the shared machine will appear in their Tailscale client as if it were on their own Tailnet.
5. Client will: Install Tailscale and sign in. Accept the invite link received. In the game client, enter the server's Tailscale IP address and the appropriate game port.

### Method 2 - Invite Client to Tailnet (what we use)
1. Add the client as a tailnet **Member**.
2. The client installs Tailscale and signs in. In the game client they enter **`vsserver:42420`** (or `vsserver.<tailnet>.ts.net:42420` if the short name doesn't resolve).
3. Nothing else to configure: the tailnet policy already lets members, and the close group, reach the game port on `tag:vsserver` (see *Tailscale Setup* below).

> Method 1 (sharing the machine) is untested since the server was tagged (2026-11-06). Members are the supported path.
---
## World Management (Using .vcdbs Files)
### Upload a world from Windows
```powershell
scp "C:\Users\<YOU>\AppData\Roaming\VintagestoryData\Saves\MyWorld.vcdbs" \
vsserver@<SERVER_IP>:/var/vintagestory/data/Saves/
```
### Rename a world (optional but recommended)
```bash
cd /var/vintagestory/data/Saves  
sudo mv "MyWorld.vcdbs" myworld.vcdbs
```
### Point the server to your world
Edit:
```bash
sudo nano /var/vintagestory/data/serverconfig.json
```

Find:
```bash
"SaveFileLocation": "/var/vintagestory/data/Saves/default.vcdbs"
```

Replace with:
```bash
"SaveFileLocation": "/var/vintagestory/data/Saves/myworld.vcdbs"
```
Save + restart the server.

---
## Whitelist & Access Control
### Disable whitelist mode (open server)
```bash
sudo /home/vintagestory/server/server.sh command serverconfig whitelistmode off
```
### Enable whitelist mode
```bash
sudo /home/vintagestory/server/server.sh command serverconfig whitelistmode on
```
### Add a player to whitelist
```bash
sudo /home/vintagestory/server/server.sh command whitelist add <PlayerName>
```
### Remove a player
```bash
sudo /home/vintagestory/server/server.sh command whitelist remove <PlayerName>
```
---
## Server Networking
### LAN connection (inside your home)
Use this in Vintage Story:
<SERVER_LAN_IP>:42420

Example:
192.168.x.102:42420

---
## Tailscale Setup (Remote Play Without Port Forwarding)
### Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```
### Bring the server online
```bash
sudo tailscale up
```
Authenticate using the link provided.
### Get your Tailscale IP
```bash
tailscale ip -4
```
Example result:
100.x.x.x
### Tag the server (done 2026-11-06)

The server runs Tailscale itself as `vsserver`, **tagged `tag:vsserver`** (Machines → ⋯ → Edit ACL tags). Two reasons:

- **A user-owned server inherits its owner's rights.** Before tagging it belonged to the admin account, so the game server could reach everything the admin can: every device and subnet, on every port. A tagged device has no owner, so it can only *receive* what the policy grants and reach nothing. Verified: from the server, the MuttMetrics app now times out.
- **Tagged devices have key expiry off,** so the server doesn't silently drop off the tailnet after months.

Keep Tailscale updated on it: `sudo tailscale set --auto-update`.

### Connect via Tailscale from any device

Players connect **by name, straight to the server's own Tailscale node**:
```
vsserver:42420
```

The tailnet policy grants members and the close group `tcp:42420` and `udp:42420` on `tag:vsserver`, and nothing else; a policy test checks it ([[Tailscale]] §7).

*History:* from 2026-03 players had to use the server's VLAN 30 address through OPNsense's subnet route (`10.30.x.x:42420`). The only member rule then was "members → `10.zz.zz.0/24`", which covers the LAN address but not the server's Tailscale address, so connecting by name stopped working once the default allow-all policy was replaced. That broad rule also let every member reach every port on every VLAN 30 host, and it was removed on 2026-11-06.

### First new player: if they can't connect
1. Their Tailscale is on, and *Use Tailscale DNS* is on.
2. Try `vsserver:42420`, then the full `vsserver.<tailnet>.ts.net:42420`.
3. On the server, while they try: `sudo journalctl -u tailscaled -f | grep -i drop`. A `Drop … :42420` line means the policy doesn't cover their account (admin console → Access controls → *Preview rules* for them). No line at all means their Tailscale isn't connected.

### Switching tailnets
If you need to move the server to a different tailnet:
```bash
sudo tailscale logout
sudo tailscale up --hostname=vsserver --advertise-tags=tag:vsserver
```
Authenticate via the link provided (the tag must exist in the new tailnet's `tagOwners`), then verify connectivity.

---
## Useful Maintenance Commands

### View logs
```bash
ls /var/vintagestory/data/Logs  
sudo tail -f /var/vintagestory/data/Logs/server-main.log
```
### Check disk space
```bash
df -h
```
### Check memory usage
```bash
free -h
```
### Update server files manually (recommended method)

**Important:** Check the .NET runtime requirement before updating. VS 1.22.x requires .NET 10 — if you're jumping a major version, you may need to install a new runtime first.

Check installed runtimes:
```bash
dotnet --list-runtimes | grep NETCore
```

Install .NET 10 if missing (Ubuntu 24.04):
```bash
sudo apt-get update && sudo apt-get install -y dotnet-runtime-10.0
```

Then update the server:
```bash
# 1. Back up server.sh (it lives in the install dir and will be wiped)
cp /home/vintagestory/server/server.sh /home/vintagestory/server.sh.bak

# 2. Stop the server
sudo /home/vintagestory/server/server.sh stop

# 3. Back up world data
cp -r /var/vintagestory/data/Saves/ ~/vs_saves_backup_$(date +%Y%m%d)

# 4. Wipe install dir and download new version
cd /home/vintagestory/server
sudo rm -rf *
sudo wget https://cdn.vintagestory.at/gamefiles/stable/vs_server_linux-x64_<VERSION>.tar.gz

# 5. Extract and restore server.sh
sudo tar -zxvf vs_server_linux-x64_<VERSION>.tar.gz
cp /home/vintagestory/server.sh.bak /home/vintagestory/server/server.sh

# 6. Start
sudo ./server.sh start
```

Verify the update worked:
```bash
grep "Game Version\|Savegame\|Remapper" /var/vintagestory/data/Logs/server-main.log | tail -5
```
The `Remapper` line is normal on major version jumps — it migrates world data automatically.
---
## Mods

Mod files (`.zip`) go in:
```bash
/var/vintagestory/data/Mods/
```

### Currently installed mods (as of 1.22.1)

| Mod | Version | Notes |
|-----|---------|-------|
| BetterRuins | 0.6.2 | Worldgen — ruins, structures |
| BetterTraders | 0.2.0 | New trader types |
| ChiselTools | 1.17.2 | Extended chisel tool modes |
| Draconis | 1.4.2 | Dragons — hatch, raise, ride |
| Rivers | 5.0.1 | Worldgen — rivers (worldgen mod, see note below) |
| Jaunt | 3.0.0-rc.3 | Dependency for Draconis |
| AttributeRenderingLibrary | 3.1.4 | Dependency for Draconis |

**Worldgen mod note:** Rivers and BetterRuins only affect ungenerated chunks. Adding them to an existing world creates visible chunk border seams where old and new terrain meet. This is cosmetic and expected — always back up the world before adding worldgen mods.

### Adding mods

```bash
# 1. Back up the world first
cp -r /var/vintagestory/data/Saves/ ~/vs_saves_premods_$(date +%Y%m%d)

# 2. Stop the server
sudo /home/vintagestory/server/server.sh stop

# 3. Download mods directly into the Mods folder
cd /var/vintagestory/data/Mods
wget <mod-download-url>

# If SCP from Windows fails with "Permission denied", upload to home dir first:
# scp file.zip vsserver@<IP>:/home/vsserver/
# then: sudo mv /home/vsserver/*.zip /var/vintagestory/data/Mods/

# 4. Start and verify
sudo /home/vintagestory/server/server.sh start
grep -i "mod\|error\|fatal" /var/vintagestory/data/Logs/server-main.log | head -50
```

A clean load looks like: `Found X mods (0 disabled)` and `JsonPatch Loader: X patches total ... no errors`.

### Client-side mods (Windows)

Clients must have matching mods installed. Copy the same `.zip` files into:
```
%AppData%\VintagestoryData\Mods\
```

You can pull mods from the server directly via SCP:
```powershell
scp vsserver@<SERVER_IP>:/var/vintagestory/data/Mods/*.zip C:\Users\<YOU>\Downloads\vsmods\
```

---
## Backup Your World
### Create a backup
```bash
sudo cp /var/vintagestory/data/Saves/myworld.vcdbs \
/var/vintagestory/data/Saves/myworld-backup-$(date +%F).vcdbs
```
### Download a backup to Windows
```powershell
scp vsserver@<SERVER_IP>:/var/vintagestory/data/Saves/myworld.vcdbs \
"C:\Users\<YOU>\Desktop\myworld-backup.vcdbs"
```
---
## Quick Troubleshooting

### Server fails to start after major version update
Check if a new .NET runtime is required:
```bash
sudo -u vintagestory dotnet /home/vintagestory/server/VintagestoryServer.dll --dataPath /var/vintagestory/data/
```
This runs the server directly and shows the actual error. VS 1.22.x requires .NET 10 — install it if missing.

### Server loads a new world instead of yours
- Check `SaveFileLocation` in `serverconfig.json`
- Filename is case-sensitive
- Restart server

### “You are not on the whitelist”
As of VS 1.20, servers default to whitelist/invite-only mode. Run:
```bash
sudo /home/vintagestory/server/server.sh command “serverconfig whitelistmode off”
```

### Remote player can't connect via Tailscale
Make sure they're running:
```
tailscale up 
```
Then connect to `10.30.x.x:42420`. Ping won't work (ICMP blocked by OPNsense firewall rule) but the game connection will.

### Cannot upload mods/world via SCP (Permission denied)
Upload to home dir first, then move:
```bash
# On Windows:
scp file.zip vsserver@10.30.x.x:/home/vsserver/
# On server:
sudo mv /home/vsserver/*.zip /var/vintagestory/data/Mods/
```

### Cannot upload world via SCP
Fix permissions:
```bash
sudo chown -R vsserver:vintagestory /var/vintagestory
sudo chmod -R 775 /var/vintagestory
```
---
