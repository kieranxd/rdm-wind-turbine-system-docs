# Configuration Files

Reference for all key configuration files in the system.

## Docker Compose

**Location:** `~/IOTstack/docker-compose.yml`

!!! note "Authentication Tokens"
    Replace `YOUR_RENDERER_TOKEN_HERE` and `YOUR_AUTH_TOKEN_HERE` with actual tokens in production.

### Complete Configuration

```yaml
networks:
  iotstack_default:
    external: true
    name: "iotstack_default"

services:
  grafana:
    cap_drop:
      - "AUDIT_CONTROL"
      - "BLOCK_SUSPEND"
      - "DAC_READ_SEARCH"
      - "IPC_LOCK"
      - "IPC_OWNER"
      - "LEASE"
      - "LINUX_IMMUTABLE"
      - "MAC_ADMIN"
      - "MAC_OVERRIDE"
      - "NET_ADMIN"
      - "NET_BROADCAST"
      - "SYSLOG"
      - "SYS_ADMIN"
      - "SYS_BOOT"
      - "SYS_MODULE"
      - "SYS_NICE"
      - "SYS_PACCT"
      - "SYS_PTRACE"
      - "SYS_RAWIO"
      - "SYS_RESOURCE"
      - "SYS_TIME"
      - "SYS_TTY_CONFIG"
      - "WAKE_ALARM"
    container_name: "grafana"
    entrypoint:
      - "/run.sh"
    environment:
      - "TZ=Etc/UTC"
      - "GF_PATHS_DATA=/var/lib/grafana"
      - "GF_PATHS_LOGS=/var/log/grafana"
      - "PATH=/usr/share/grafana/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
      - "GF_PATHS_CONFIG=/etc/grafana/grafana.ini"
      - "GF_PATHS_HOME=/usr/share/grafana"
      - "GF_PATHS_PLUGINS=/var/lib/grafana/plugins"
      - "GF_PATHS_PROVISIONING=/etc/grafana/provisioning"
      - "GF_RENDERING_SERVER_URL=http://145.24.180.60:8081/render"
      - "GF_RENDERING_CALLBACK_URL=http://145.24.180.60:3000/"
      - "GF_RENDERING_RENDERER_TOKEN=YOUR_RENDERER_TOKEN_HERE"
      - "GF_SERVER_DOMAIN=145.24.180.60"
    hostname: "568c217d58c8"
    image: "grafana/grafana:latest"
    ipc: "private"
    labels:
      maintainer: "Grafana Labs <hello@grafana.com>"
      org.opencontainers.image.source: "https://github.com/grafana/grafana"
    logging:
      driver: "json-file"
      options: {}
    networks:
      - "iotstack_default"
    ports:
      - "3000:3000/tcp"
    restart: "unless-stopped"
    user: "0"
    volumes:
      - "/home/iotuser/IOTstack/volumes/grafana/data:/var/lib/grafana"
      - "/home/iotuser/IOTstack/volumes/grafana/log:/var/log/grafana"
    working_dir: "/usr/share/grafana"
  influxdb:
    command:
      - "influxd"
    container_name: "influxdb"
    entrypoint:
      - "/entrypoint.sh"
    environment:
      - "INFLUXDB_REPORTING_DISABLED=false"
      - "INFLUXDB_HTTP_AUTH_ENABLED=false"
      - "INFLUXDB_MONITOR_STORE_ENABLED=FALSE"
      - "TZ=Etc/UTC"
      - "INFLUXDB_HTTP_FLUX_ENABLED=false"
      - "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    hostname: "f5dd8a03e804"
    image: "influxdb:1.11"
    ipc: "private"
    logging:
      driver: "json-file"
      options: {}
    networks:
      - "iotstack_default"
    ports:
      - "8086:8086/tcp"
    restart: "unless-stopped"
    user: "0"
    volumes:
      - "/home/iotuser/IOTstack/backups/influxdb/db:/var/lib/influxdb/backup"
      - "/home/iotuser/IOTstack/volumes/influxdb/data:/var/lib/influxdb"
  nodered:
    cap_drop:
      - "AUDIT_CONTROL"
      - "BLOCK_SUSPEND"
      - "DAC_READ_SEARCH"
      - "IPC_LOCK"
      - "IPC_OWNER"
      - "LEASE"
      - "LINUX_IMMUTABLE"
      - "MAC_ADMIN"
      - "MAC_OVERRIDE"
      - "NET_ADMIN"
      - "NET_BROADCAST"
      - "SYSLOG"
      - "SYS_ADMIN"
      - "SYS_BOOT"
      - "SYS_MODULE"
      - "SYS_NICE"
      - "SYS_PACCT"
      - "SYS_PTRACE"
      - "SYS_RAWIO"
      - "SYS_RESOURCE"
      - "SYS_TIME"
      - "SYS_TTY_CONFIG"
      - "WAKE_ALARM"
    container_name: "nodered"
    entrypoint:
      - "./entrypoint.sh"
    environment:
      - "TZ=Europe/Amsterdam"
      - "NODE_OPTIONS=--dns-result-order=ipv4first"
      - "PATH=/usr/src/node-red/node_modules/.bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
      - "NODE_VERSION=20.19.5"
      - "YARN_VERSION=1.22.22"
      - "NODE_RED_VERSION=v4.1.1"
      - "NODE_PATH=/usr/src/node-red/node_modules:/data/node_modules"
      - "FLOWS=flows.json"
    hostname: "0b07c52e9073"
    image: "iotstack-nodered:latest"
    ipc: "private"
    labels:
      authors: "Dave Conway-Jones, Nick O'Leary, James Thomas, Raymond Mouthaan"
      org.opencontainers.image.source: "https://github.com/node-red/node-red-docker"
    logging:
      driver: "json-file"
      options: {}
    network_mode: "host"
    ports:
      - "1880:1880/tcp"
    restart: "unless-stopped"
    user: "0"
    volumes:
      - "/home/iotuser/IOTstack/volumes/nodered/data:/data"
      - "/home/iotuser/IOTstack/volumes/nodered/ssh:/root/.ssh"
    working_dir: "/usr/src/node-red"
  portainer-ce:
    container_name: "portainer-ce"
    entrypoint:
      - "/portainer"
    environment:
      - "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    hostname: "9dd856bf4ddd"
    image: "portainer/portainer-ce"
    ipc: "private"
    labels:
      io.portainer.server: "true"
      org.opencontainers.image.description: "Docker container management made simple, with the world's most popular GUI-based container management platform."
      org.opencontainers.image.title: "Portainer"
      org.opencontainers.image.vendor: "Portainer.io"
    logging:
      driver: "json-file"
      options: {}
    networks:
      - "iotstack_default"
    ports:
      - "8000:8000/tcp"
      - "9000:9000/tcp"
      - "9443:9443/tcp"
    restart: "unless-stopped"
    volumes:
      - "/home/iotuser/IOTstack/volumes/portainer-ce/data:/data"
      - "/var/run/docker.sock:/var/run/docker.sock"
    working_dir: "/"
  renderer:
    cap_drop:
      - "AUDIT_CONTROL"
      - "BLOCK_SUSPEND"
      - "DAC_READ_SEARCH"
      - "IPC_LOCK"
      - "IPC_OWNER"
      - "LEASE"
      - "LINUX_IMMUTABLE"
      - "MAC_ADMIN"
      - "MAC_OVERRIDE"
      - "NET_ADMIN"
      - "NET_BROADCAST"
      - "SYSLOG"
      - "SYS_ADMIN"
      - "SYS_BOOT"
      - "SYS_MODULE"
      - "SYS_NICE"
      - "SYS_PACCT"
      - "SYS_PTRACE"
      - "SYS_RAWIO"
      - "SYS_RESOURCE"
      - "SYS_TIME"
      - "SYS_TTY_CONFIG"
      - "WAKE_ALARM"
    command:
      - "server"
    container_name: "renderer"
    entrypoint:
      - "tini"
      - "--"
      - "/usr/bin/grafana-image-renderer"
    environment:
      - "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
      - "CHROME_BIN=/usr/bin/chromium"
      - "LANG=en_US.UTF-8"
      - "LC_ALL=en_US.UTF-8"
      - "AUTH_TOKEN=YOUR_AUTH_TOKEN_HERE"
      - "TZ=Europe/Amsterdam"
    hostname: "3a5022bd091e"
    image: "grafana/grafana-image-renderer:latest"
    ipc: "private"
    labels:
      maintainer: "Grafana team <hello@grafana.com>"
      org.opencontainers.image.source: "https://github.com/grafana/grafana-image-renderer/tree/master/Dockerfile"
    logging:
      driver: "json-file"
      options: {}
    networks:
      - "iotstack_default"
    ports:
      - "8081:8081/tcp"
    restart: "unless-stopped"
    user: "65532"
    working_dir: "/home/nonroot"
version: "3.6"
```

**Notes:**

- Update IP addresses to match your network
- Adjust volume paths as needed

## Node-RED Settings

**Location:** `~/IOTstack/volumes/nodered/data/settings.js`

### Key Sections

```javascript
module.exports = {
    // Server settings
    uiPort: process.env.PORT || 1880,
    
    // Security
    adminAuth: {
        type: "credentials",
        users: [{
            username: "admin",
            password: "$2b$08$...",  // Generated hash
            permissions: "*"
        }]
    },
    
    // Editor settings
    editorTheme: {
        projects: {
            enabled: false
        }
    },
    
    // Logging
    logging: {
        console: {
            level: "info",
            metrics: false,
            audit: false
        }
    },
    
    // Context storage
    contextStorage: {
        default: {
            module: "memory"
        }
    }
}
```

## Grafana Environment Variables

### Required for Image Rendering

```bash
GF_SERVER_DOMAIN=192.168.178.164
GF_SERVER_ROOT_URL=http://192.168.178.164:3000/
GF_SERVER_SERVE_FROM_SUB_PATH=false
GF_RENDERING_SERVER_URL=http://grafana-image-renderer:8081/render
GF_RENDERING_CALLBACK_URL=http://192.168.178.164:3000/
GF_RENDERING_RENDERER_TOKEN=glsa_AzyNQ7uhzHzg4UCe5KC8fHUAiJPGjBaz_0d0c9ffb
```

### Optional Settings

```bash
# Anonymous access
GF_AUTH_ANONYMOUS_ENABLED=true
GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer

# SMTP for alerts
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.gmail.com:587
GF_SMTP_USER=your-email@gmail.com
GF_SMTP_PASSWORD=your-app-password
GF_SMTP_FROM_ADDRESS=your-email@gmail.com
```

## Raspberry Pi Network Configuration

### Static IP (dhcpcd)

**Location:** `/etc/dhcpcd.conf`

```bash
interface eth0
static ip_address=192.168.178.148/24
static routers=192.168.178.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

### Static IP (Netplan - Ubuntu)

**Location:** `/etc/netplan/01-netcfg.yaml`

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.178.148/24
      gateway4: 192.168.178.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Apply:

```bash
sudo netplan apply
```

## Raspberry Pi Boot Configuration

**Location:** `/boot/config.txt`

```bash
# Enable USB gadget mode
dtoverlay=dwc2

# PoE HAT fan control
dtparam=poe_fan_temp0=60000
dtparam=poe_fan_temp1=65000
dtparam=poe_fan_temp2=70000
dtparam=poe_fan_temp3=75000
```

**Location:** `/boot/cmdline.txt`

```
console=serial0,115200 console=tty1 root=PARTUUID=... rootfstype=ext4 fsck.repair=yes rootwait modules-load=dwc2,libcomposite
```

## Tailscale Configuration

### On VM (Docker)

**Location:** `~/IOTstack/docker-compose.yml` (see Docker Compose section above)

**Authenticate:**

```bash
docker exec tailscale tailscale up
```

**Get Tailscale IP:**

```bash
docker exec tailscale tailscale ip -4
```

**Check status:**

```bash
docker exec tailscale tailscale status
```

### On Raspberry Pi (Native)

**Install:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**Get Tailscale IP:**

```bash
tailscale ip -4
```

**Check status:**

```bash
sudo tailscale status
```

**Enable on boot:**

```bash
sudo systemctl enable tailscaled
```

### Tailscale Network IPs

- **VM Tailscale IP**: e.g., `100.64.1.10` (assigned automatically)
- **Raspberry Pi Tailscale IP**: e.g., `100.64.1.5` (assigned automatically)
- **IP Range**: Typically `100.x.x.x` (Tailscale CGNAT range)

## Systemd Service Files

### USB Gadget Service

**Location:** `/etc/systemd/system/usb-gadget.service`

```ini
[Unit]
Description=USB Gadget Mode Configuration
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/gadget.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl enable usb-gadget.service
```

## SSH Configuration

### SSH Server Config

**Location:** `/etc/ssh/sshd_config`

```bash
# Security settings
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
ChallengeResponseAuthentication no

# Performance
UseDNS no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

### Authorized Keys

**Location:** `~/.ssh/authorized_keys`

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... node-red@iotstack
```

Permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## Sudo Configuration

**Location:** `/etc/sudoers.d/nopasswd`

```bash
hrtech ALL=(ALL) NOPASSWD: /usr/bin/tee
hrtech ALL=(ALL) NOPASSWD: /usr/bin/mount
hrtech ALL=(ALL) NOPASSWD: /usr/bin/umount
hrtech ALL=(ALL) NOPASSWD: /usr/bin/mkdir
hrtech ALL=(ALL) NOPASSWD: /bin/rm
hrtech ALL=(ALL) NOPASSWD: /bin/cp
```

Create with:

```bash
sudo visudo -f /etc/sudoers.d/nopasswd
```

## Firewall Rules

### UFW (Uncomplicated Firewall)

**On VM:**

```bash
sudo ufw allow from 192.168.178.0/24 to any port 3000  # Grafana
sudo ufw allow from 192.168.178.0/24 to any port 8086  # InfluxDB
sudo ufw allow from 192.168.178.0/24 to any port 1880  # Node-RED
sudo ufw allow from 192.168.178.0/24 to any port 9000  # Portainer
sudo ufw allow from 192.168.178.0/24 to any port 22    # SSH
sudo ufw enable
```

**On Raspberry Pi:**

```bash
# Allow SSH only through Tailscale interface
sudo ufw allow in on tailscale0 to any port 22
sudo ufw deny 22/tcp
sudo ufw enable
```

Check status:

```bash
sudo ufw status numbered
```

## Cron Jobs

### Automated Backups

**Location:** User crontab (`crontab -e`)

```cron
# Daily backup at 2 AM
0 2 * * * cd ~/IOTstack && ./scripts/backup.sh

# Weekly cleanup at 3 AM Sunday
0 3 * * 0 docker system prune -f

# Daily offsite sync at 4 AM
0 4 * * * rclone sync ~/IOTstack/backups/ remote:backups/
```

## InfluxDB Configuration

### Create Database

```bash
docker exec influxdb influx -execute "CREATE DATABASE iot_data"
```

### Retention Policy

```bash
docker exec influxdb influx -execute \
  "CREATE RETENTION POLICY 90_days ON iot_data DURATION 90d REPLICATION 1 DEFAULT"
```

### Show Configuration

```bash
docker exec influxdb influx -execute "SHOW DATABASES"
docker exec influxdb influx -execute "SHOW RETENTION POLICIES ON iot_data"
```

## Environment Variables Summary

### Grafana

| Variable | Value | Purpose |
| -------- | ----- | ------- |
| GF_SERVER_DOMAIN | 192.168.178.164 | Server address |
| GF_SERVER_ROOT_URL | http://192.168.178.164:3000/ | Base URL |
| GF_RENDERING_SERVER_URL | http://grafana-image-renderer:8081/render | Renderer endpoint |
| GF_RENDERING_CALLBACK_URL | http://192.168.178.164:3000/ | Callback for renderer |
| GF_RENDERING_RENDERER_TOKEN | glsa_... | Auth token |

### Grafana Image Renderer

| Variable | Value | Purpose |
| -------- | ----- | ------- |
| AUTH_TOKEN | glsa_... | Must match Grafana token |
| HTTP_HOST | 0.0.0.0 | Listen on all interfaces |
| HTTP_PORT | 8081 | Service port |
| BROWSER_TZ | Europe/Amsterdam | Timezone for rendering |

### Node-RED

| Variable | Value | Purpose |
| -------- | ----- | ------- |
| TZ | Europe/Amsterdam | Timezone |
| NODE_RED_CREDENTIAL_SECRET | (optional) | Encrypt credentials |

### Tailscale

| Variable | Value | Purpose |
| -------- | ----- | ------- |
| TS_AUTHKEY | (optional) | Auth key for unattended setup |
| TS_STATE_DIR | /var/lib/tailscale | State directory |
| TS_EXTRA_ARGS | --advertise-tags=tag:iotstack | Additional arguments |

## File Locations Reference

| File/Directory | Location | Purpose |
| -------------- | -------- | ------- |
| Docker Compose | `~/IOTstack/docker-compose.yml` | Stack definition |
| InfluxDB data | `~/IOTstack/volumes/influxdb/data/` | Database storage |
| Grafana data | `~/IOTstack/volumes/grafana/data/` | Dashboards, config |
| Node-RED flows | `~/IOTstack/volumes/nodered/data/flows.json` | Flow definitions |
| Node-RED settings | `~/IOTstack/volumes/nodered/data/settings.js` | Configuration |
| Tailscale state | `~/IOTstack/volumes/tailscale/state/` | Tailscale configuration |
| Backups | `~/IOTstack/backups/` | Backup storage |
| Pi scripts | `/home/hrtech/*.sh` | Shell scripts |
| Gadget script | `/usr/local/bin/gadget.sh` | USB gadget setup |
| USB backing file | `/home/hrtech/drive.bin` | Mass storage image |
| Incoming images | `/home/hrtech/incoming_images/` | From Node-RED |
| Processed images | `/home/hrtech/eink_usb/SAMSUNG_E-Paper/` | Ready for display |

## Default Ports

| Service | Port | Protocol | Access |
| ------- | ---- | -------- | ------ |
| Grafana | 3000 | HTTP | Web UI |
| InfluxDB | 8086 | HTTP | API |
| Node-RED | 1880 | HTTP | Web UI |
| Portainer | 9000 | HTTP | Web UI |
| Image Renderer | 8081 | HTTP | Internal |
| SSH | 22 | TCP | Terminal |
| PoE+ | N/A | Ethernet | Power |

## API Credentials

### eGauge XML API

**Endpoint:** `https://egauge84432.egaug.es/cgi-bin/egauge?inst`

**Authentication:** Digest Auth
- **Username:** `HR_RDM`
- **Password:** (contact system administrator)

**Register Types:**

The eGauge XML API returns instantaneous values for various register types. According to the eGauge XML API documentation, the main register types include:

- **Power registers** (e.g., `S6_*`, `S5_*`, `S2_*`): Grid power measurements in watts
- **Orientation registers** (e.g., `IM_*`): Turbine orientation and position data
- **Vibration registers** (e.g., `VM_*`): Vibration monitoring sensors
- **Frequency registers**: Grid and generator frequency in Hz
- **Voltage registers**: Line voltage measurements

The Node-RED flow sanitizes these register names by converting them to snake_case and adding appropriate prefixes (e.g., `grid_`, `orientation_`, `vibration_`).

**Response Format:** XML containing register values with metadata

**Documentation:** See `egauge-xml-api.pdf` in project root for complete register type specifications and data structures.

### Weather API

**Endpoint:** `https://weerlive.nl/api/weerlive_api_v2.php`

**Authentication:** API Key
- **Key:** `3c2224ecab`
- **Location:** `51.89908,4.42348` (Rotterdam coordinates)

**No additional authentication required** - API key is passed as URL parameter.

## Quick Reference Commands

```bash
# Start all services
cd ~/IOTstack && docker compose up -d

# Stop all services
cd ~/IOTstack && docker compose down

# View logs
docker compose logs -f <service>

# Restart service
docker compose restart <service>

# Backup
cd ~/IOTstack && ./scripts/backup.sh

# InfluxDB CLI
docker exec -it influxdb influx

# Check gadget status
cat /sys/kernel/config/usb_gadget/EInkGadget/UDC

# SSH to Pi (via Tailscale)
ssh hrtech@<pi-tailscale-ip>  # e.g., ssh hrtech@100.64.1.5

# Trigger refresh (via Tailscale)
ssh hrtech@<pi-tailscale-ip> '/home/hrtech/refresh_script.sh'

# Check Tailscale status
docker exec tailscale tailscale status  # On VM (if Docker)
sudo tailscale status  # On Pi or VM (if native)
```
