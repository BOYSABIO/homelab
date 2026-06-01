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
### Connect via Tailscale from any device

The VS server sits on VLAN30. Remote players connect via OPNsense's Tailscale subnet routing — **not** directly to the server's own Tailscale IP.

Connect using:
```
10.30.x.x:42420
```

### Switching tailnets
If you need to move the server to a different tailnet:
```bash
sudo tailscale logout
sudo tailscale up 
```
Authenticate via the link provided, then verify connectivity.

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
