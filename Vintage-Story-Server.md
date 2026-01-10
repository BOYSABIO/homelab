# Vintage Story Dedicated Server — Admin Guide
For: Ubuntu Server (v1.21+), Tailscale networking, `.vcdbs` world saves  

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
100.x.x.x:42420
### Optional: Enable MagicDNS (clean hostname)
In the Tailscale admin panel:
- Enable MagicDNS
- Give the server a name (e.g., vsserver)

Then connect using the hostname in tailscale

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
```bash
cd /home/vintagestory/server
sudo ./server.sh stop
rm -rf *
wget <new-server-download-link>
tar -xvf <downloaded-file>
cp ../server.sh .
sudo ./server.sh start
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
### Server loads a new world instead of yours
- Check SaveFileLocation path
- Ensure filename matches exactly (case-sensitive)
- Restart server
### “You are not on the whitelist”
Run:
```bash
sudo /home/vintagestory/server/server.sh command serverconfig whitelistmode off
```
### Cannot upload world via SCP
Fix permissions:
```bash
sudo chown -R vsserver:vintagestory /var/vintagestory
sudo chmod -R 775 /var/vintagestory
```
---
