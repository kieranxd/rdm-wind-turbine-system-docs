# IOTstack Installation

IOTstack is a Docker-based IoT services stack specifically designed for Raspberry Pi, but works great on any Linux system.

## Prerequisites

- VM with Ubuntu/Debian installed
- SSH access to the VM
- Sudo privileges
- Internet connectivity

## Installation Process

### Method 1: Quick Install (Recommended)

The simplest way to install IOTstack on an existing system:

1. **Install curl:**

   ```bash
   sudo apt install -y curl
   ```

2. **Run the IOTstack installer:**

   ```bash
   curl -fsSL https://raw.githubusercontent.com/SensorsIot/IOTstack/master/install.sh | bash
   ```

The `install.sh` script is designed to be run multiple times. If it discovers a problem, it will explain how to fix it, and you can safely re-run the script.

### Method 2: Manual Docker Installation

If you prefer to install Docker manually first:

```bash
# Remove old Docker versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository (for Debian)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group
sudo usermod -aG docker $USER
```

**Important:** Log out and back in for group changes to take effect.

Verify Docker:

```bash
docker --version
docker compose version
```

### 3. Clone IOTstack

```bash
cd ~
git clone https://github.com/SensorsIot/IOTstack.git IOTstack
cd IOTstack
```

### 4. Run the Menu to Select Services

```bash
./menu.sh
```

The menu will:

1. Check for dependencies and install them
2. Present a selection interface for choosing containers

**Select these services** (for this project):

- [x] InfluxDB
- [x] Grafana
- [x] Node-RED
- [x] Portainer

**Do NOT select** (we'll add these separately):

- [ ] Grafana Image Renderer (not available in default menu)
- [ ] WireGuard (we'll use Tailscale instead - see note below)

!!! info "Tailscale VPN for VM-to-Pi Communication"
    The VM uses Tailscale VPN to securely communicate with the Raspberry Pi over the internet without requiring port forwarding.
    
    **Recommended Setup**: Run Tailscale in Docker alongside IOTstack
    - Integrates cleanly with IOTstack services
    - Managed through docker-compose
    - Automatic restarts with container orchestration
    - Shared network access with Node-RED
    
    **Alternative**: Install Tailscale natively on the VM
    - System-level service
    - Independent of Docker stack
    - Simpler for some users
    
    Both the VM and Raspberry Pi connect to your Tailscale network (tailnet) and receive persistent IPs in the 100.x.x.x range. Node-RED uses these IPs for secure SSH/SCP communication.
    
    **Network Flow**: `Node-RED → Tailscale VPN → Raspberry Pi`
    
    See [Node-RED Tailscale Configuration](../services/node-red.md#tailscale-configuration) for complete setup instructions.

Navigate with arrow keys, select with Space, confirm with Enter.

!!! warning "Critical: Node-RED Add-on Nodes Required"
    If you select Node-RED, you **MUST** press the right-arrow key and choose at least one add-on node. If you skip this step, Node-RED will not build properly. Recommended add-on nodes:
    
    - `node-red-dashboard` (for UI dashboards)
    - `node-red-contrib-influxdb` (for InfluxDB integration)
    
    Even if you don't need these specific nodes, you must select at least one for the build to work correctly.

### 4. Generate docker-compose.yml

After selection, the menu will generate `~/IOTstack/docker-compose.yml`

### 5. Review Configuration

```bash
cat ~/IOTstack/docker-compose.yml
```

Verify the selected services are present.

## Start the Stack

### First Time Start

```bash
cd ~/IOTstack
docker compose up -d
```

This will:

- Download Docker images
- Create volumes for persistent data
- Start all containers
- May take 5-10 minutes

### Verify Containers

```bash
docker compose ps
```

All containers should show state `Up`.

### Check Logs

```bash
docker compose logs -f
```

Press Ctrl+C to exit logs.

## Initial Access

### Grafana

- URL: `http://<vm-ip>:3000`
- Default login: `admin` / `admin`
- Change password on first login

### InfluxDB

- URL: `http://<vm-ip>:8086`
- Create initial database:

```bash
docker exec -it influxdb influx
```

In the InfluxDB shell:

```sql
CREATE DATABASE iot_data
SHOW DATABASES
EXIT
```

### Node-RED

- URL: `http://<vm-ip>:1880`
- No default authentication (configure in settings)

### Portainer

- URL: `http://<vm-ip>:9000`
- Create admin user on first access

## Directory Structure

```
~/IOTstack/
├── docker-compose.yml          # Main stack definition
├── menu.sh                     # Menu for managing services
├── services/                   # Service definitions
├── volumes/                    # Persistent data
│   ├── grafana/
│   ├── influxdb/
│   ├── nodered/
│   └── portainer/
└── backups/                    # Backup location
```

## Stack Management Commands

### Start/Stop Stack

```bash
cd ~/IOTstack
docker compose up -d      # Start all services
docker compose down       # Stop all services
docker compose restart    # Restart all services
```

### Individual Container Management

```bash
docker compose restart grafana      # Restart one service
docker compose logs -f node-red     # View logs for one service
docker compose stop influxdb        # Stop one service
docker compose start influxdb       # Start one service
```

### Update Containers

```bash
cd ~/IOTstack
docker compose pull       # Pull new images
docker compose up -d      # Recreate containers with new images
```

### Backup

IOTstack includes backup scripts:

```bash
cd ~/IOTstack
./scripts/backup.sh
```

Backups stored in `~/IOTstack/backups/`

## Auto-Start on Boot

Docker is configured to auto-start containers by default. Verify:

```bash
docker inspect grafana | grep -A 5 RestartPolicy
```

Should show `"Name": "unless-stopped"` or `"Name": "always"`

If not, update `docker-compose.yml`:

```yaml
services:
  grafana:
    restart: unless-stopped
```

## Networking

All services are on the same Docker network, allowing them to communicate:

- Grafana can reach InfluxDB at `http://influxdb:8086`
- Node-RED can reach InfluxDB at `http://influxdb:8086`
- Services use internal DNS resolution

## Next Steps

1. [Configure InfluxDB](../services/influxdb.md)
2. [Configure Grafana](../services/grafana.md)
3. [Configure Node-RED](../services/node-red.md)
4. [Add Grafana Image Renderer](../services/grafana-renderer.md)

## Troubleshooting

### Port Conflicts

If ports are already in use:

```bash
sudo netstat -tlnp | grep :3000  # Check what's using port 3000
```

Edit `docker-compose.yml` to change ports:

```yaml
ports:
  - "3001:3000"  # Map host port 3001 to container port 3000
```

### Permission Issues

If you get permission denied errors:

```bash
sudo chown -R $USER:$USER ~/IOTstack/volumes/
```

### Container Won't Start

Check logs:

```bash
docker compose logs <service-name>
```

Remove and recreate:

```bash
docker compose rm <service-name>
docker compose up -d <service-name>
```
