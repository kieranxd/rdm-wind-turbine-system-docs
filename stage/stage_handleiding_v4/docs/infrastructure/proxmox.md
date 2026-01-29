# Proxmox Host Setup

## Prerequisites

- Physical server or compatible hardware
- Network connectivity
- Access to Proxmox installation media

## Installation Steps

### 1. Install Proxmox VE

1. Download Proxmox VE ISO from [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
2. Create bootable USB or mount ISO in server
3. Boot from installation media
4. Follow the installation wizard:
   - Select target disk
   - Set timezone and keyboard layout
   - Configure network (static IP recommended)
   - Set root password
   - Set hostname and domain

### 2. Post-Installation Configuration

#### Access Web Interface

1. Navigate to `https://<proxmox-ip>:8006` in your browser
2. Log in with username `root` and your password
3. Accept the self-signed certificate warning

#### Update System

```bash
apt update
apt dist-upgrade -y
```

#### Configure Storage

1. Go to **Datacenter → Storage**
2. Verify local and local-lvm storage pools
3. Add additional storage if needed

## Network Configuration

### Bridge Configuration

The default bridge (`vmbr0`) should be configured during installation.

Verify in **Proxmox Node → System → Network**:

- **Name**: `vmbr0`
- **IPv4/CIDR**: `<proxmox-ip>/<netmask>`
- **Gateway (IPv4)**: `<gateway>`
- **Autostart**: `true`
- **Bridge ports**: `<physical-interface>`

This bridge allows VMs to access the network.

## Resource Allocation

### Recommended Minimum Resources

For the IoT Stack VM:

- **CPU**: 2 cores
- **RAM**: 4GB
- **Disk**: 50GB
- **Network**: Bridged to vmbr0

These can be adjusted based on your Proxmox host's capabilities.

## Backup Configuration

### Automated Backups

1. Go to **Datacenter → Backup**
2. Click **Add** to create a backup job
3. Configure:
   - **Storage**: Select backup destination
   - **Schedule**: Daily at 2:00 AM recommended
   - **Selection mode**: All or select specific VMs
   - **Mode**: Snapshot (allows backup of running VMs)
   - **Compression**: ZSTD (good balance)
   - **Retention**: Keep last 7 backups

### Manual Backup

To manually backup a VM:

1. Select the VM
2. Go to **Backup** tab
3. Click **Backup now**
4. Select storage and mode
5. Wait for completion

## Security Recommendations

### Firewall

1. Enable Proxmox firewall: **Datacenter → Firewall**
2. Add rules to allow:
   - SSH (port 22) from trusted networks
   - Web interface (port 8006) from trusted networks
   - Deny all other inbound traffic by default

### Updates

Keep Proxmox updated:

```bash
apt update
apt dist-upgrade -y
```

Run monthly or enable automatic security updates.

### User Management

Instead of using root for everything:

1. Go to **Datacenter → Users**
2. Create administrative user
3. Assign appropriate permissions
4. Enable two-factor authentication

## Useful Commands

### Check VM Status

```bash
qm list
```

### Start/Stop VM

```bash
qm start <vmid>
qm stop <vmid>
qm shutdown <vmid>  # Graceful shutdown
```

### Monitor Resources

```bash
pvesh get /cluster/resources
```

### View Logs

```bash
tail -f /var/log/pveproxy/access.log  # Web access
journalctl -u pvedaemon -f            # Proxmox daemon
```

## Next Steps

Once Proxmox is installed and configured, proceed to [VM Configuration](vm-setup.md) to create the virtual machine for the IoT stack.
