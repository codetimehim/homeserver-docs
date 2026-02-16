# Proxmox Media Server Architecture Guide

> **Location**: `/home/claud/download/Homeserver/docs/proxmox-media-server-architecture.md`
> 
> See also: 
> - Maintenance Guide: `proxmox-media-server-maintenance.md` (same folder)
> - Video References: `../REFERENCES.md`

## Overview

This guide walks through building a complete media server infrastructure on Proxmox, from bare metal to working Jellyfin streaming. Designed for first-time setup following video references [Part 1](https://youtu.be/zLFB6ulC0Fg) and [Part 2](https://youtu.be/Uzqf0qlcQlo).

**Target Architecture:**
```
Bare Metal (Proxmox Host)
    └── LXC Container (Ubuntu + Docker)
            ├── Jellyfin (Media Server)
            ├── Docker Compose
            └── Network Bridge (Host Mode)
    └── ZFS Pool (Media Storage)
    └── Cockpit (Web Management)
    └── SAMBA (File Shares)
```

---

## Why These Components?

### Proxmox VE
- **Type 1 Hypervisor** — lightweight, runs directly on hardware
- **Web UI + API** — manage VMs/containers without touching CLI
- **LXC Support** — Linux containers use shared kernel (vs full VMs)

### LXC vs VM
| Feature | LXC (Container) | VM (Full Virtualization) |
|---------|-----------------|--------------------------|
| Resource Usage | Minimal (shared kernel) | Higher (separate kernel) |
| Boot Time | Seconds | Minutes |
| Privilege | ⚠️ Limited for security | Full isolation |
| Use Case | Trusted workloads, Docker | Isolated, untrusted |

### Cockpit
- **Web-based Linux administration** — disk, networking, services
- **Browser access** — no SSH needed for basic management
- **Real-time graphs** — resource monitoring

### Docker + Compose
- **Infrastructure as Code** — define entire stack in YAML
- **Portable** — same config on any Docker host
- **Isolation** — apps don't conflict with system

### Jellyfin
- **Open-source media server** — Plex alternative
- **GPU transcoding** — offload encoding to GPU
- **No account required** — self-hosted privacy

### SAMBA
- **Windows file sharing protocol** — mount as network drive
- **Authentication** — user/password protection

### ZFS (Optional)
- **Copy-on-write snapshots** — instant backups
- **Self-healing** — detects bit rot
- **Compression** — transparent gzip/zstd

---

## Step-by-Step Installation

### Phase 1: Proxmox Host Setup (Video Part 1)

#### 1.1 Install Proxmox VE
1. Download ISO from [proxmox.com](https://www.proxmox.com)
2. Create bootable USB with Rufus (Windows) or `dd` (Linux)
3. Boot server from USB
4. Follow installer prompts:
   - Set hostname (e.g., `pve1`)
   - Configure network static IP: `192.168.86.30/24`
   - Gateway: `192.168.86.1`
   - DNS: `8.8.8.8`

#### 1.2 Post-Install Setup

**Update system:**
```bash
apt update && apt -y upgrade
```

**Install Cockpit (Web Management):**
```bash
apt install -y cockpit
systemctl enable --now cockpit.socket
```
Access: `https://192.168.86.30:9090`

**Configure SSH Key Access (for automation):**
```bash
mkdir -p /root/.ssh
# Add public key - REPLACE with your key:
echo "ssh-ed25519 AAAAC3Nza... openclaw" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

**Enable IP Forwarding (for containers):**
```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-forwarding.conf
sysctl --system
```

#### 1.3 Create ZFS Storage Pool

```bash
# Check available disks
lsblk

# Create ZFS pool (replace /dev/sdb with your disk)
zpool create tank /dev/sdb

# Set mountpoint
zfs set mountpoint=/tank tank

# Enable compression
zfs set compression=zstd tank

# Create datasets
zfs create tank/media
zfs create tank/backup
```

#### 1.4 Configure SAMBA Share

```bash
apt install -y samba

# Create user
useradd -r -s /bin/false mediauser
smbpasswd -a mediauser

# Edit config
cat >> /etc/samba/smb.conf << 'EOF'
[media]
path = /tank/media
browseable = yes
read only = no
guest ok = no
valid users = mediauser
EOF

systemctl restart smbd
```

---

### Phase 2: LXC Container Setup

#### 2.1 Create Container (via Web UI or CLI)

**Via CLI:**
```bash
# Download Ubuntu template first (in Web UI: Datacenter > Storage > CT Templates)

# Create container
pct create 100 /var/lib/vz/template/cache/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname media-stack \
  --memory 4096 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage local-zfs \
  --rootfs local-zfs:20

# Start container
pct start 100
```

#### 2.2 Post-Create Container Setup

```bash
# Enter container
pct exec 100 bash

# Update
apt update && apt install -y curl wget git

# Install Docker
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify Docker works
docker --version
```

#### 2.3 Mount ZFS Pool to Container

**On Proxmox host:**
```bash
# Add mountpoint (persistent)
pct set 100 -mp0 /tank/media,mp=/opt/media-stack/media-shared

# In container, verify
pct exec 100 ls -la /opt/media-stack/media-shared
```

---

### Phase 3: Docker Stack (Video Part 2)

#### 3.1 Create Directory Structure

```bash
# In LXC container
mkdir -p /opt/media-stack/config/jellyfin
mkdir -p /opt/media-stack/media-shared
```

#### 3.2 Docker Compose

Create `/opt/media-stack/docker-compose.yml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    network_mode: host
    volumes:
      - /opt/media-stack/config/jellyfin:/config
      - /opt/media-stack/media-shared:/media
    restart: unless-stopped
```

Start stack:
```bash
cd /opt/media-stack
docker compose up -d
docker compose ps
```

**Access Jellyfin:** `http://192.168.86.46:8096`

---

## Important: LXC Limitations

### What Works in LXC
- ✅ **Jellyfin** — doesn't need privileged access
- ✅ **Most Docker apps** — standard containers OK
- ✅ **VPN containers** — with `NET_ADMIN` capability

### What DOESN'T Work in Unprivileged LXC
- ❌ **Sonarr** (privileged container) — needs sysctl access
- ❌ **Gluetun** with full routing — privilege escalation
- ❌ **Modifying kernel parameters** — blocked by security

### Solutions for Blocked Apps

**Option A: Native Install on Proxmox Host**
```bash
# SSH to host (192.168.86.30)
# Install Sonarr natively
wget https://services.sonarr.tv/v1/download/main/latest?version=4&os=linux&arch=x64 -O /tmp/sonarr.tar.gz
tar -xzf /tmp/sonarr.tar.gz -C /opt/sonarr
# ... create systemd service
```

**Option B: Migrate to Full VM**
- Convert container to VM (more resources, full privileges)
- Reinstall stack in VM

---

## Verification Steps

| Component | Test | Expected Result |
|-----------|------|-----------------|
| Proxmox | `https://192.168.86.30:8006` | Web UI loads |
| Cockpit | `https://192.168.86.30:9090` | Login prompt |
| LXC Container | `pct status 100` | Shows "running" |
| Jellyfin | `http://192.168.86.46:8096` | Setup wizard |
| SAMBA | `\\192.168.86.30\media` | Windows Explorer |
| ZFS | `zfs list` | Shows pools |

---

## Video References

- **Part 1 (Physical → Proxmox)**: https://youtu.be/zLFB6ulC0Fg
- **Part 2 (Apps & Docker)**: https://youtu.be/Uzqf0qlcQlo

---

## Next Steps

See `proxmox-media-server-maintenance.md` for:
- Adding Sonarr (native install workaround)
- VPN setup with Gluetun
- Backup procedures
- Troubleshooting common issues
