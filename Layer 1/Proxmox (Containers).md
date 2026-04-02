# Proxmox Homelab Setup — Full Documentation
### From Fresh Install to Running a Next.js Web App with LibreTranslate
---
## Table of Contents

1. [Network Architecture](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#network-architecture)
2. [Initial Proxmox Host Setup](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#initial-proxmox-host-setup)
    - [Fixing the System Clock](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#fixing-the-system-clock)
    - [Disabling Enterprise Repositories](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#disabling-enterprise-repositories)
    - [Adding the Free Community Repository](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#adding-the-free-community-repository)
3. [Creating LXC Containers](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#creating-lxc-containers)
    - [Downloading an LXC Template](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#downloading-an-lxc-template)
    - [Container Settings Reference](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#container-settings-reference)
    - [The Proxmox Firewall + VLAN Bug](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#the-proxmox-firewall--vlan-bug)
    - [Enabling Root SSH on Ubuntu 24.04](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#enabling-root-ssh-on-ubuntu-2404)
4. [Webapp Container Setup](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#webapp-container-setup)
    - [Installing Node.js](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#installing-nodejs)
    - [Cloning a Private GitHub Repository](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#cloning-a-private-github-repository)
    - [Configuring the Environment](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#configuring-the-environment)
    - [Migrating the SQLite Database](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#migrating-the-sqlite-database)
    - [Building and Running with PM2](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#building-and-running-with-pm2)
    - [Deployment Script](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#deployment-script)
5. [LibreTranslate Container Setup](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#libretranslate-container-setup)
    - [Installing Docker](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#installing-docker)
    - [Running LibreTranslate](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#running-libretranslate)
6. [MGMT VLAN Internet Access](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#mgmt-vlan-internet-access)
7. [Key Caveats and Lessons Learned](https://claude.ai/chat/810aff84-3cfe-4e8a-a1f6-7abb91461a07#key-caveats-and-lessons-learned)

---
## Network Architecture

| VLAN     | ID  | Purpose                           |
| -------- | --- | --------------------------------- |
| MGMT     | 10  | Proxmox host management           |
| TRUSTED  | 20  | Personal devices / workstations   |
| SERVICES | 30  | Self-hosted services / containers |

- **Proxmox host IP**: `10.x.x.3` (VLAN10)
- **Webapp LXC**: `10.x.x.10` (VLAN30)
- **LibreTranslate LXC**: `10.x.x.11` (VLAN30)
- **Router/Gateway for VLAN30**: `10.x.x.1`
- **Firewall**: OPNsense handles inter-VLAN routing and firewall rules
- **Proxmox is connected via a trunk port** — the physical NIC carries tagged VLANs, and `vmbr0` is a VLAN-aware bridge (`bridge-vids 2-4094`)

---
## Initial Proxmox Host Setup
### Fixing the System Clock
> **Why this matters:** On a fresh Proxmox install, the hardware clock (RTC) may be wrong. APT validates repository signatures using timestamps — if your system clock is in the past, signatures will appear "not yet valid" and `apt update` will fail entirely with errors like `Not live until 2026-XX-XXTXX:XX:XXZ`.

**Symptoms:**
```
Sub-process /usr/bin/sqv returned an error code (1), error message is:
Verifying signature: Not live until 2026-04-01T19:46:29Z
```

**Fix:**
```bash
# Disable NTP temporarily (required before manual set)
timedatectl set-ntp false

# Set the correct date and time
timedatectl set-time "2026-04-02 04:00:00"

# Sync hardware clock to system clock
hwclock --systohc

# Re-enable NTP
timedatectl set-ntp true

# Restart whichever NTP service is running
systemctl restart chrony   # or systemd-timesyncd

# Verify everything is correct
timedatectl status
```

**Expected output:**
```
Local time: Thu 2026-04-02 08:28:53 CEST
System clock synchronized: yes
NTP service: active
```
> **Caveat:** You must run `timedatectl set-ntp false` before `set-time`, otherwise it will return `Failed to set time: Automatic time synchronization is enabled`.

---
### Disabling Enterprise Repositories
Proxmox ships with enterprise repositories enabled by default. Without a paid subscription, these return `401 Unauthorized` errors. There are **two file formats** to be aware of — both `.list` files and `.sources` files may be present.

**Find all enterprise repo references:**
```bash
grep -r "enterprise.proxmox.com" /etc/apt/
```

**For `.sources` files** (Proxmox 9 / newer format — uses deb822 format):
```bash
# Add "Enabled: no" to disable them
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources
```
> **Caveat:** The `.sources` format does not have an `Enabled: yes` line by default — omitting it means enabled. You cannot use `sed` to replace it; you must append `Enabled: no`.

**For `.list` files** (older format — comment out the line):
```bash
echo "# deb https://enterprise.proxmox.com/debian/pve trixie InRelease" > /etc/apt/sources.list.d/pve-enterprise.list
echo "# deb https://enterprise.proxmox.com/debian/ceph-squid trixie InRelease" > /etc/apt/sources.list.d/ceph.list
```

---
### Adding the Free Community Repository
> **Note:** Proxmox 9 uses Debian **trixie** as its base, not bookworm. Make sure to use the correct codename.

```bash
# Add the no-subscription (free) community repo
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" >> /etc/apt/sources.list

# Verify sources.list has no duplicates
cat /etc/apt/sources.list

# Remove any leftover bookworm line if present
sed -i '/bookworm/d' /etc/apt/sources.list

# Update and upgrade
apt update && apt dist-upgrade -y
```
> **Caveat:** If you accidentally add the repo twice, you'll see warnings like `Target Packages is configured multiple times`. Remove duplicates manually with `nano /etc/apt/sources.list`

---
## Creating LXC Containers
### Downloading an LXC Template
Before creating a container, you need a template image:
1. In the Proxmox web UI, click **local** storage in the left sidebar
2. Click **CT Templates** → **Templates** button
3. Search for `ubuntu` and download **Ubuntu 24.04**
4. Wait for `TASK OK` — this confirms the download and checksum verification succeeded

---
### Container Settings Reference
These settings were used for both containers in this setup:

| Setting                 | Value             | Notes                                           |
| ----------------------- | ----------------- | ----------------------------------------------- |
| Unprivileged            | ✅ Yes             | More secure, recommended                        |
| Nesting                 | ✅ Yes             | Required if running Docker inside the container |
| Template                | Ubuntu 24.04      |                                                 |
| Disk                    | 8GB+              | 2GB is not enough for LibreTranslate            |
| CPU                     | 1 core            | Sufficient for both use cases                   |
| Memory (webapp)         | 1024 MiB          |                                                 |
| Memory (libretranslate) | 2048 MiB          | ~1-2GB per language model                       |
| Bridge                  | vmbr0             |                                                 |
| VLAN Tag                | 30                | Routes container into SERVICES VLAN             |
| IPv4                    | Static            | Assign a fixed IP so it's always reachable      |
| Gateway                 | 10.x.x.1          | OPNsense VLAN30 gateway                         |
| Firewall                | ❌ Disabled        | See caveat below — critical                     |
| DNS                     | Use host settings | Inherits from Proxmox host                      |

---
### The Proxmox Firewall + VLAN Bug
> **This is the most important caveat in this entire document.**
When you enable the Proxmox container firewall (the checkbox on the Network tab during creation), Proxmox inserts a firewall bridge between the container and `vmbr0`. The bridge chain looks like this:

```
vmbr0 → fwpr100p0 → fwln100i0 → fwbr100i0 → veth100i0 → container
```

On a VLAN-aware bridge, VLAN tags are only applied to `fwpr100p0` — the other interfaces in the chain (`fwln100i0`, `veth100i0`) only have VLAN 1. This means packets arrive at `veth100i0` but never enter the container's network namespace.

**Symptoms:**
- Container shows correct IP in Proxmox UI
- Container can ping out (gateway, other hosts)
- Nothing can ping or connect to the container inbound
- `tcpdump -i veth100i0 icmp` shows packets arriving but no replies
- `tcpdump` inside the container shows nothing at all
- `bridge vlan show` reveals `veth100i0` only has VLAN 1, not VLAN 30

**Fix — disable the Proxmox firewall on the container:**
```bash
# Stop the container first
pct stop 100

# Set firewall=0 in the network config
pct set 100 --net0 name=eth0,bridge=vmbr0,tag=30,ip=10.x.x.10/24,gw=10.x.x.1,firewall=0

# Start it again
pct start 100
```
> **Design note:** Since OPNsense handles all inter-VLAN firewall rules, the Proxmox container firewall is redundant in this setup. It is safe to leave it disabled. OPNsense is the correct place to enforce network policy.

---
### Enabling Root SSH on Ubuntu 24.04
Ubuntu 24.04 LXC containers disable root SSH login by default. After creating a container, run this inside the Proxmox console (CT → Console → press Enter to wake it up):

```bash
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl restart ssh
```

> **Caveat:** If you run `apt upgrade` after this, it may prompt about `sshd_config` being locally modified. Always choose **"keep the local version currently installed"** — otherwise the upgrade will overwrite your config and lock you out of SSH.

You can then SSH in from your workstation:
```bash
ssh root@10.x.x.10
```

---
## Webapp Container Setup
**Container:** CT 100 | IP: `10.x.x.10` | Hostname: `webapp`
### Installing Node.js
```bash
apt update && apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs git
node --version
npm --version
```

---
### Cloning a Private GitHub Repository
GitHub no longer accepts passwords for Git operations. You need a **Personal Access Token (PAT)**:
1. Go to GitHub → Profile → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Set expiration to **90 days**
4. Check the **repo** scope only
5. Copy the token immediately (shown once only — starts with `ghp_`)

**Clone a specific branch of a private repo:**
```bash
git clone -b dev https://YOURUSERNAME@github.com/YOURUSERNAME/REPONAME.git /opt/webapp
```

When prompted for password, paste the PAT token.

---
### Configuring the Environment
```bash
cd /opt/webapp

# Copy the example env file
cp .env.example .env

# Generate a secure random secret
openssl rand -base64 32

# Edit the env file
nano .env
```

**Key variables to set:**
```env
NEXTAUTH_SECRET=<output of openssl rand -base64 32>
NEXTAUTH_URL=http://10.x.x.10:3000
LIBRETRANSLATE_URL=http://10.x.x.11:5000
DATABASE_URL="file:./dev.db"
```
> **Caveat:** Use the actual server IP for `NEXTAUTH_URL`, not `localhost`. NextAuth uses this URL for redirect callbacks — if set to localhost, authentication redirects will fail when accessing from another device on the network.

---
### Migrating the SQLite Database
The `prisma/dev.db` file is listed in `.gitignore` and is never committed to the repo. It must be copied manually from your development machine.

**From your local Windows machine:**
```powershell
scp C:\Users\SABIO\Documents\GitHub\Accountable\prisma\dev.db root@10.x.x.10:/opt/webapp/prisma/dev.db
```

**Then sync the Prisma schema:**
```bash
cd /opt/webapp
npx prisma generate
npx prisma db push
```

Expected output: `The database is already in sync with the Prisma schema.`

> **Note:** Only run `npx prisma migrate dev` if you need to apply new migrations. For an existing database that matches the current schema, `db push` is sufficient and non-destructive.

---
### Building and Running with PM2
> **Why not `npm run dev`?** `npm run dev` is a development server — it watches files, recompiles on the fly, and is not optimized for performance. It also stops when you close your SSH session. PM2 runs the app as a proper background service that survives SSH disconnects and server reboots.

**Install missing dependencies if build fails:**
```bash
# Example: framer-motion was missing from package.json
npm install framer-motion
```

**Build the production app:**
```bash
npm run build
```

> **Note:** During the Next.js build you may see `Dynamic server usage` warnings for API routes that use `headers`. These are informational only — the build still succeeds and the routes work correctly at runtime.

**Install PM2 and start the app:**
```bash
npm install -g pm2

# Start the app
pm2 start npm --name "webapp" -- start

# Save the process list so PM2 restores it after reboot
pm2 save

# Configure PM2 to start automatically on boot
pm2 startup
```

The app is now accessible at `http://10.x.x.10:3000`.

---
### Deployment Script
When you push changes to GitHub, SSH into the container and run this script to pull and redeploy:

```bash
# Create the deploy script
cat > /opt/deploy.sh << 'EOF'
#!/bin/bash
cd /opt/webapp
git pull
npm install
npm run build
pm2 restart webapp
echo "Deploy complete!"
EOF

chmod +x /opt/deploy.sh
```

**Usage after pushing changes:**
```bash
ssh root@10.x.x.10
/opt/deploy.sh
```
> **Future improvement:** This can be automated with a GitHub webhook so the container deploys automatically on every push to the `dev` branch.

---
## LibreTranslate Container Setup
**Container:** CT 101 | IP: `10.x.x.11` | Hostname: `libretranslate`

### Installing Docker
```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify
docker --version
```

> **Note:** The `nesting=1` feature must be enabled on the LXC container (set during creation) for Docker to work inside an unprivileged LXC.

---
### Running LibreTranslate

```bash
mkdir -p /opt/libretranslate

cat > /opt/libretranslate/docker-compose.yml << 'EOF'
services:
  libretranslate:
    image: libretranslate/libretranslate
    restart: always
    ports:
      - "5000:5000"
    environment:
      - LT_HOST=0.0.0.0
      - LT_LOAD_ONLY=en,de
EOF

cd /opt/libretranslate
docker compose up -d

# Watch startup logs
docker compose logs -f
```

**Verify it's working:**
```bash
curl http://localhost:5000/health
# Expected: {"status":"ok"}
```

LibreTranslate is now accessible from the webapp container at `http://10.x.x.11:5000`.

> **Resource note:** LibreTranslate uses approximately 1-2GB RAM per language loaded. This setup loads only English and German (`en,de`), which is why 2048 MiB was allocated to this container. Adding more languages requires more RAM.

> **Restart policy:** `restart: always` ensures LibreTranslate automatically restarts when the container reboots.

---
## MGMT VLAN Internet Access
During the initial setup, internet access was temporarily enabled for the MGMT VLAN (VLAN10) in OPNsense in order to:
- Run `apt update` and `apt dist-upgrade` on the Proxmox host
- Download the Ubuntu 24.04 LXC template
- Install packages on the Proxmox host

**Once setup is complete, internet access for MGMT can be turned off.** The containers on VLAN30 route through OPNsense independently — Proxmox does not proxy their traffic. Revoking internet from VLAN10 has zero effect on the running containers.

**Rule of thumb:**
- Internet for MGMT **ON** → when updating Proxmox, downloading templates, or installing host packages
- Internet for MGMT **OFF** → all other times (day to day operation)

This follows the principle of least privilege for management networks and is consistent with enterprise network design.

---
## Key Caveats and Lessons Learned
### 1. System Clock Must Be Correct Before APT Works
APT verifies GPG signatures using timestamps. A wrong system clock causes all repo signatures to fail. Always fix the clock first before attempting any package operations on a fresh install.
### 2. Proxmox 9 Uses Trixie, Not Bookworm
The community repo line must use `trixie` as the suite name. Using `bookworm` adds an extra unnecessary repo entry.
### 3. Enterprise Repos Use `.sources` Format in Proxmox 9
The newer deb822 `.sources` format does not have an `Enabled:` line by default (omission = enabled). You cannot `sed` replace it — you must append `Enabled: no`.
### 4. Proxmox Container Firewall Breaks VLAN Tagging
This is the biggest gotcha. Enabling the Proxmox firewall on a container running on a VLAN-aware bridge inserts a firewall bridge that doesn't properly pass VLAN tags through to the container. The container can send traffic out but receives nothing inbound. The fix is `firewall=0` in the container's network config. Since OPNsense already handles firewall policy, this is not a security concern.
### 5. Ubuntu 24.04 Blocks Root SSH by Default
Always add `PermitRootLogin yes` to `/etc/ssh/sshd_config` immediately after creating an Ubuntu 24.04 container. When upgrading packages later, keep your local version of `sshd_config` when prompted.
### 6. GitHub Requires Personal Access Tokens
Passwords are no longer accepted for Git over HTTPS. Generate a classic PAT with `repo` scope. Include your username in the clone URL to avoid authentication confusion.
### 7. NEXTAUTH_URL Must Be the Real Server IP
Setting `NEXTAUTH_URL=http://localhost:3000` will break authentication when accessed from any other device. Always use the container's actual IP address.
### 8. Never Run `npm run dev` in Production
Use `npm run build` + PM2 + `npm start` for any server deployment. `npm run dev` is not persistent, not optimized, and stops when your SSH session ends.
### 9. SQLite `dev.db` Is Not in Git
The database file is gitignored. It must be manually copied to the server with `scp`. After copying, run `npx prisma generate && npx prisma db push` to ensure the schema is in sync.
### 10. LXC Disk Size for LibreTranslate
2GB disk is not enough for LibreTranslate — the language models take significant space. Use at least 8GB. RAM should be 2GB minimum for two languages (en + de).