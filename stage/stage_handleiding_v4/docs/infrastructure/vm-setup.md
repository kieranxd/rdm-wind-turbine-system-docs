# VM Configuration

This guide covers creating and configuring the Linux VM that will host the IOTstack.

## Create New VM

### From Proxmox Web Interface

1. Click **Create VM** button (top right)
2. **General** tab:
   - **Node**: Select your Proxmox node
   - **VM ID**: Auto-assigned or choose (e.g., 100)
   - **Name**: `iot-stack` (or your preferred name)

3. **OS** tab:
   - **ISO image**: Select Debian or Ubuntu server (latest stable)
   - **Type**: Linux
   - **Version**: 6.x (latest kernel)

4. **System** tab:
   - **Graphic card**: Default (VGA)
   - **Machine**: Default (i440fx or q35)
   - **BIOS**: Default (SeaBIOS)
   - **SCSI Controller**: VirtIO SCSI
   - **Qemu Agent**: ✓ Enabled (recommended)

5. **Disks** tab:
   - **Bus/Device**: SCSI
   - **Storage**: local-lvm (or your storage)
   - **Disk size**: 50 GB minimum (80-100 GB recommended)
   - **Cache**: Default (No cache)
   - **Discard**: ✓ Enabled (if using SSD)
   - **SSD emulation**: ✓ Enabled (if Proxmox storage is SSD)

6. **CPU** tab:
   - **Sockets**: 1
   - **Cores**: 2 minimum (4 recommended)
   - **Type**: host (best performance)

7. **Memory** tab:
   - **Memory (MiB)**: 4096 minimum (8192 recommended for Grafana)

8. **Network** tab:
   - **Bridge**: vmbr0
   - **Model**: VirtIO (paravirtualized)
   - **Firewall**: ✓ Enabled

9. **Confirm** tab:
   - **Start after created**: ✓ Optional
   - Click **Finish**

## OS Installation

### Boot and Install

1. Select the VM and click **Start**
2. Open **Console** (noVNC or SPICE)
3. Follow the Ubuntu/Debian installation:
   - **Language**: English (or your preference)
   - **Keyboard**: Your layout
   - **Network**: Should auto-configure via DHCP
   - **Mirror**: Default or choose closest
   - **Disk**: Use entire disk (no LVM needed)
   - **User**: Create user (e.g., `iotadmin`)
   - **Hostname**: `iotstack` or your preference
   - **Software**: Install OpenSSH server

### Set Static IP (Recommended)

After installation, configure static IP for reliability:

#### For Debian

Edit `/etc/network/interfaces`:

```
auto ens18
iface ens18 inet static
    address <vm_ip>
    netmask 255.255.255.0
    gateway <gateway_ip>
    dns-nameservers 8.8.8.8 8.8.4.4
```

Restart networking:

```bash
sudo systemctl restart networking
```

#### For Ubuntu (Netplan)

Edit `/etc/netplan/00-installer-config.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens18:  # Your interface name (check with 'ip a')
      dhcp4: no
      addresses:
        - <vm_ip>/24  # Your desired IP
      gateway4: <gateway_ip>  # Your gateway
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Apply:

```bash
sudo netplan apply
```

## Post-Installation Setup

### Update System

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

### Install QEMU Guest Agent

This allows Proxmox to properly manage the VM:

```bash
sudo apt install qemu-guest-agent -y
sudo systemctl start qemu-guest-agent
sudo systemctl enable qemu-guest-agent
```

Verify in Proxmox: VM should show IP address and system info.

### Install Essential Tools

```bash
sudo apt install -y git curl wget vim htop net-tools
```

## VM Optimization

### Enable TRIM (for SSD)

If Proxmox storage is on SSD:

```bash
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
```

### Configure Swap

For 4GB RAM systems, add swap:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Adjust swappiness:

```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Snapshot (Before IOTstack Installation)

Take a snapshot before proceeding:

1. In Proxmox web interface, select the VM
2. Go to **Snapshots** tab
3. Click **Take Snapshot**
4. Name: `pre-iotstack`
5. Description: "Clean OS install before Docker"

This allows easy rollback if needed.

## Next Steps

Proceed to [IOTstack Installation](iotstack.md) to set up the Docker-based IoT services.
