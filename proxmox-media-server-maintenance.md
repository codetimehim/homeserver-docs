# Proxmox Media Server Maintenance & Reinstall Guide

> **Location**: `/home/claud/download/Homeserver/docs/proxmox-media-server-maintenance.md`
> 
> See also:
> - Architecture Guide: `proxmox-media-server-architecture.md` (same folder)
> - Video References: `../REFERENCES.md`

Quick reference for fixing broken components without full rebuild.

**Target Reader:** Administrator with working Proxmox server needing component repair.

---

## Quick Reference: Service Locations

| Service | Host/Container | IP:Port | Access Method |
|---------|--------------|---------|---------------|
| Proxmox Web UI | Host | `192.168.86.30:8006` | Browser HTTPS |
| Cockpit | Host | `192.168.86.30:9090` | Browser HTTPS |
| Jellyfin | LXC 100 | `192.168.86.46:8096` | Browser HTTP |
| SAMBA | Host | `192.168.86.30` | `\\IP\media` |
| SSH | Host | `192.168.86.30:22` | `root@192.168.86.30` |

---

## Jellyfin: Reinstall (Preserve Config)

### Scenario: Container corrupted, config intact

**Quick Reinstall:**
```bash
# SSH to Proxmox host
ssh root@192.168.86.30

# Check container status
pct status 100

# If container running but Jellyfin broken:
pct exec 100 bash
cd /opt/media-stack
docker compose down
docker compose pull jellyfin
docker compose up -d jellyfin
docker compose ps
```

### Scenario: Full container rebuild needed

**Preserve then rebuild:**
```bash
# SSH to host
pct stop 100

# Backup config (on host)
cp -r /var/lib/lxc/100/rootfs/opt/media-stack /root/media-backup-$(date +%Y%m%d)

# Or mount and copy:
ls /var/lib/lxc/100/rootfs/opt/media-stack/config/

# Rebuild container (see Architecture Guide)
# Then restore:
pct exec 100 mkdir -p /opt/media-stack
cpct exec 100 cp -r /opt/media-stack/config/* /opt/media-stack/config/
```

---

## Docker Compose: Full Stack Reinstall

```bash
# SSH to host
pct exec 100 bash
cd /opt/media-stack

# Stop & remove
docker compose down

# Remove old image
docker rmi jellyfin/jellyfin:latest

# Pull fresh
docker compose pull

# Recreate
docker compose up -d

# Verify
docker compose ps
docker logs --tail 20 media-stack-jellyfin-1
```

---

## ZFS: Check and Repair

### Check Pool Health
```bash
# On Proxmox host
zpool status tank
zfs list
```

### Repair Corruption
```bash
# Scrub (self-heal)
zpool scrub tank

# Monitor progress
zpool status tank -v
```

### Expand Pool (add disk)
```bash
# Attach new disk
zpool attach tank /dev/sdb /dev/sdc

# Check
zpool status tank
```

---

## SAMBA: Restart & Troubleshoot

### Restart Service
```bash
systemctl restart smbd
systemctl status smbd
```

### Test Connectivity
```bash
# From another Linux machine:
smbclient -L 192.168.86.30 -U mediauser

# Or mount test:
sudo mount -t cifs //192.168.86.30/media /mnt -o username=mediauser
```

### Check Permissions
```bash
# On Proxmox host
ls -la /tank/media/
getfacl /tank/media/
```

---

## LXC Container: Network Fix

### Scenario: Container lost internet
**Symptoms:** `ping 8.8.8.8` fails from inside container

**Fix:**
```bash
# On host
pct stop 100
pct start 100

# Verify
pct exec 100 -- ping -c 2 8.8.8.8
```

### Scenario: Persistent network issues
**Check bridge:**
```bash
ip addr show vmbr0
cat /etc/network/interfaces | grep -A 5 vmbr0

# Fix NAT if needed
iptables -t nat -A POSTROUTING -s 192.168.86.0/24 ! -d 192.168.86.0/24 -j MASQUERADE
```

---

## Known Issue: LXC Privilege Limitation

### Problem: Sonarr Cannot Run in Docker (LXC)

**Error:**
```
OCI runtime create failed: open sysctl net.ipv4.ip_unprivileged_port_start: permission denied
```

**Cause:** 
Sonarr Docker image requires privileged sysctl access. LXC unprivileged containers block this.

**Workarounds:**

#### Option A: Native Install on Proxmox Host

```bash
# SSH to host (NOT container)
cd /tmp
wget "https://services.sonarr.tv/v1/download/main/latest?version=4&os=linux&arch=x64" -O sonarr.tar.gz
mkdir -p /opt/sonarr
tar -xzf sonarr.tar.gz -C /opt/sonarr
useradd -r -s /bin/false sonarr
chown -R sonarr:sonarr /opt/sonarr

# Create systemd service
cat > /etc/systemd/system/sonarr.service << 'EOF'
[Unit]
Description=Sonarr
After=network.target

[Service]
User=sonarr
Group=sonarr
ExecStart=/opt/sonarr/Sonarr -nobrowser -data=/var/lib/sonarr
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now sonarr

# Access: http://192.168.86.30:8989
```

#### Option B: Migrate to Full VM

See Architecture Guide — create VM instead of container for full privilege support.

---

## SSH: Regenerate Keys

### If Keys Lost (Authentication Issues)

**On your local machine:**
```bash
# Generate new key pair
mkdir -p ~/.openclaw/keys
ssh-keygen -t ed25519 -f ~/.openclaw/keys/openclaw -N "" -C "openclaw"

# Show public key (add to Proxmox)
cat ~/.openclaw/keys/openclaw.pub
```

**On Proxmox host:**
```bash
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAAC3Nza... openclaw" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

**Test:**
```bash
ssh -i ~/.openclaw/keys/openclaw root@192.168.86.30
```

---

## VPN: Gluetun Setup (Video Reference)

Gluetun routes container traffic through VPN. **Note:** This requires elevated privileges that may conflict with LXC unprivileged mode.

### Alternative: Host-Level VPN

```bash
# Install WireGuard
apt install -y wireguard

# Configure (provider-specific)
# See: https://youtu.be/Uzqf0qlcQlo?t=1234 (timestamp approximate)

# Enable
cp /etc/wireguard/wg0.conf /etc/wireguard/
systemctl enable --now wg-quick@wg0
```

---

## Update Procedures

### Update Docker Images
```bash
pct exec 100 bash
cd /opt/media-stack

# Pull latest
docker compose pull

# Recreate with new images
docker compose up -d

# Cleanup old images
docker system prune -f
```

### Update System Packages
```bash
# Container
pct exec 100 -- apt update && apt upgrade -y

# Host
apt update && apt upgrade -y
```

### Update Proxmox
```bash
# Via Web UI: Datacenter > Updates
# Or CLI:
pveupgrade
```

---

## Emergency: Full Access Loss

**If SSH fails AND web UIs inaccessible:**

1. **Connect physical monitor/keyboard** to Proxmox server
2. Login locally as root
3. Check network: `ip addr`, `ping google.com`
4. Restart services: `systemctl restart pveproxy`
5. Emergency SSH enable: `systemctl start sshd`

---

## Dependency Chain

When troubleshooting, verify in this order:

```
1. Physical → Server powered, network up
2. Proxmox Host → Web UI accessible
3. Network → DHCP, DNS working
4. LXC Container → pct status 100 = running
5. Container Network → ping 8.8.8.8
6. Docker Service → docker ps
7. Jellyfin → http://192.168.86.46:8096
8. SAMBA → \\192.168.86.30\media
```

---

## File Locations Reference

| File/Path | Purpose |
|-----------|---------|
| `/opt/media-stack/docker-compose.yml` | Docker stack definition |
| `/opt/media-stack/config/jellyfin/` | Jellyfin persistent config |
| `/opt/media-stack/media-shared/` | Media library mount |
| `/var/lib/lxc/100/rootfs/` | Container filesystem (host) |
| `/tank/media/` | ZFS media pool |
| `/etc/samba/smb.conf` | SAMBA configuration |
| `/root/.ssh/authorized_keys` | SSH access keys |

---

## Troubleshooting Commands Cheat Sheet

```bash
# System state
htop
free -h
df -h

# Network
curl -s ifconfig.me  # Public IP
ss -tlnp            # Listening ports
ping 192.168.86.1   # Gateway

# Docker
docker ps -a
docker logs <container>
docker system df

# ZFS
zpool status
zfs list
zfs get compressratio tank

# Service status
systemctl status pveproxy
cockpit.service --status-all
```

---

## Video References

- **Part 1**: https://youtu.be/zLFB6ulC0Fg
- **Part 2**: https://youtu.be/Uzqf0qlcQlo

---

## Still Stuck?

Check Architecture Guide: `docs/proxmox-media-server-architecture.md`
