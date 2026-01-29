# Backup & Restore

Comprehensive backup and restore procedures.

## Backup Strategy

### Current Backup Infrastructure

This system uses two dedicated backup storage solutions:

**1. Proxmox VM Backups:**
- **Storage**: 2TB external SSD connected to Proxmox host
- **What**: Complete VM images and configurations
- **Schedule**: Daily automated backups
- **Retention**: Last 7 backups

**2. InfluxDB Data Backups:**
- **Storage**: RAID5 array (5x 2TB SSDs, ~8TB usable capacity)
- **What**: InfluxDB time-series database
- **Protection**: RAID5 provides redundancy (can survive 1 disk failure)
- **Schedule**: Continuous/automated backups

### What to Backup

1. **Proxmox VM** → 2TB External SSD
   - Complete VM image
   - VM configuration
   - Automated daily snapshots

2. **InfluxDB Data** → RAID5 Array
   - Time-series turbine data
   - Weather data
   - Historical measurements

3. **Configuration Files** → Manual/periodic backup
   - `docker-compose.yml`
   - Node-RED `settings.js`
   - Shell scripts on Raspberry Pi
   - Grafana dashboards

4. **Documentation** → Version control/manual backup
   - This manual
   - Network diagram
   - Passwords (encrypted)

### Backup Schedule

| Item | Storage | Frequency | Retention |
| ---- | ------- | --------- | --------- |
| Proxmox VM | 2TB External SSD | Daily (automated) | Last 7 backups |
| InfluxDB data | RAID5 Array | Continuous | All data (RAID5 protected) |
| Grafana dashboards | Manual | Weekly | 4 weeks |
| Node-RED flows | Manual | Weekly | 4 weeks |
| Configuration files | Manual | Weekly | 4 weeks |

## Proxmox VM Backup

### Current Configuration

**Storage**: 2TB external SSD connected to Proxmox host

The system is configured to automatically create daily VM backups with 7-day retention.

### Automated Backup (Already Configured)

The Proxmox backup job is configured with:

- **Storage**: 2TB External SSD
- **Schedule**: Daily at 2:00 AM
- **Mode**: Snapshot (VM continues running during backup)
- **Compression**: ZSTD
- **Retention**: Keep last 7 backups (oldest is automatically deleted)

To view/modify configuration:

1. Proxmox web UI → **Datacenter** → **Backup**
2. View existing backup job for the IOTstack VM
3. Modify if needed (storage, schedule, retention)

### Verify Backup Job

Check backup status:

1. **Datacenter** → **Backup**
2. View backup history and logs
3. Verify last backup completed successfully

Or via command line:

```bash
# List backups
pvesm list <storage-name>

# View backup logs
cat /var/log/pve/tasks/active
```

### Manual Backup

If needed, create an additional manual backup:

1. Select VM in Proxmox
2. **Backup** tab
3. Click **Backup now**
4. Choose:
   - **Storage**: 2TB External SSD
   - **Mode**: Snapshot (for running VM)
   - **Compression**: ZSTD
   - **Note**: Optional description
5. **Backup**

Backup file format: `vzdump-qemu-<vmid>-<timestamp>.vma.zst`

### Backup Storage Capacity

**2TB SSD Capacity:**
- Estimated VM size: ~20-30GB per backup
- 7 daily backups: ~210GB used
- Remaining capacity: ~1.8TB available for growth

!!! warning "Monitor Storage Usage"
    Check external SSD storage usage monthly. If backups grow larger, consider:
    
    - Reducing retention to 5 days
    - Cleaning up unused files in VM
    - Upgrading to larger external drive

### Download Backup for Offsite Storage

For additional offsite protection:

1. SSH to Proxmox host
2. Navigate to backup location
3. Copy to remote location:

```bash
# Find backups on external SSD
ls -lh /mnt/pve/<storage-name>/dump/

# Copy to offsite location
scp /mnt/pve/<storage-name>/dump/vzdump-qemu-*.vma.zst user@remote:/backups/
```

## InfluxDB Data Backup

### Current Configuration

**Storage**: RAID5 array with 5x 2TB SSDs (~8TB usable capacity)

The InfluxDB database is stored on a RAID5 array, providing:

- **Redundancy**: Can survive 1 disk failure without data loss
- **Capacity**: ~8TB usable storage for time-series data
- **Performance**: Fast read/write speeds for database operations

### RAID5 Protection

**How it works:**
- Data is striped across 4 disks with parity on the 5th
- If any single disk fails, data can be rebuilt from remaining disks
- **Critical**: Must replace failed disk promptly before second failure occurs

**Monitor RAID status:**

```bash
# Check RAID array status
cat /proc/mdstat

# Detailed RAID information
sudo mdadm --detail /dev/md0

# Check for disk errors
sudo smartctl -a /dev/sda  # Repeat for sdb, sdc, sdd, sde
```

!!! warning "RAID5 Limitations"
    RAID5 is NOT a backup - it's redundancy for hardware failure. It does NOT protect against:
    
    - Accidental deletion
    - Data corruption
    - Ransomware/malware
    - Multiple simultaneous disk failures
    
    Consider periodic manual backups for critical historical data.

### Manual InfluxDB Backup

For additional protection or before major changes:

```bash
# Create portable backup
docker exec influxdb influxd backup -portable -database iot_data /tmp/influxdb_backup

# Copy out of container
docker cp influxdb:/tmp/influxdb_backup ./influxdb_backup_$(date +%Y%m%d)

# Compress for storage
tar czf influxdb_backup_$(date +%Y%m%d).tar.gz influxdb_backup_$(date +%Y%m%d)
```

**Storage recommendations:**
- Store compressed backups on the 2TB external SSD
- Keep 2-3 monthly snapshots for historical recovery

### Other Docker Volumes Backup

**Grafana dashboards:**

```bash
docker cp grafana:/var/lib/grafana ./grafana_backup_$(date +%Y%m%d)
```

**Node-RED flows:**

```bash
docker cp nodered:/data ./nodered_backup_$(date +%Y%m%d)
```

### IOTstack Backup Script

IOTstack includes a backup script for all volumes:

```bash
cd ~/IOTstack
./scripts/backup.sh
```

Creates backup in `~/IOTstack/backups/`.

### Automated Backup (Optional)

Create cron job for periodic Grafana/Node-RED backups:

```bash
crontab -e
```

Add:

```cron
# Weekly backup of Grafana and Node-RED on Sundays at 3 AM
0 3 * * 0 docker cp grafana:/var/lib/grafana /backups/grafana_$(date +\%Y\%m\%d)
0 3 * * 0 docker cp nodered:/data /backups/nodered_$(date +\%Y\%m\%d)
```

## Configuration Backup

### Export Docker Compose

```bash
cp ~/IOTstack/docker-compose.yml ~/backups/docker-compose_$(date +%Y%m%d).yml
```

### Export Node-RED Flows

From Node-RED UI:

1. Menu → **Export**
2. Select **all flows**
3. Download JSON

Or from command line:

```bash
docker exec nodered cat /data/flows.json > flows_backup_$(date +%Y%m%d).json
```

### Export Grafana Dashboards

**Via UI:**

1. Dashboard → Share → Export
2. Save JSON

**Via API:**

```bash
# List dashboards
curl -s http://admin:password@<vm_ip>:3000/api/search?query= | jq

# Export specific dashboard
curl -s http://admin:password@<vm_ip>:3000/api/dashboards/uid/<uid> \
  > dashboard_backup_$(date +%Y%m%d).json
```

### Raspberry Pi Scripts

```bash
# From VM
scp <pi_username>@<pi_tailscale_ip>:/home/<pi_username>/*.sh ./pi_scripts_backup/
scp <pi_username>@<pi_tailscale_ip>:/usr/local/bin/gadget.sh ./pi_scripts_backup/
```

### Tailscale State

**If using Docker:**

```bash
cd ~/IOTstack
sudo tar czf tailscale_state_$(date +%Y%m%d).tar.gz volumes/tailscale/state/
```

**If using native install:**

```bash
# On VM
sudo tar czf tailscale_state_vm_$(date +%Y%m%d).tar.gz /var/lib/tailscale/

# On Pi
ssh <pi_username>@<pi_tailscale_ip> 'sudo tar czf /tmp/tailscale_state_pi.tar.gz /var/lib/tailscale/'
scp <pi_username>@<pi_tailscale_ip>:/tmp/tailscale_state_pi_$(date +%Y%m%d).tar.gz ./
```

**Note:** Tailscale state includes machine keys and configuration. Backing this up allows quick recovery without re-authentication.

## Restore Procedures

### Restore Proxmox VM

**Full VM Restore:**

1. In Proxmox, select target node
2. **Local** storage → **Backups**
3. Select backup file
4. Click **Restore**
5. Configure:
   - **VM ID**: Original or new
   - **Storage**: local-lvm
6. Click **Restore**
7. Wait for completion
8. Start VM

**Partial Restore (extract files):**

```bash
# Extract backup
vzdump extract vzdump-qemu-100-*.vma.zst /tmp/restore/

# Mount disk image
guestmount -a /tmp/restore/disk.raw -i /mnt/restore/

# Copy files
cp /mnt/restore/path/to/file /destination/

# Unmount
guestunmount /mnt/restore/
```

### Restore Docker Volumes

**InfluxDB:**

```bash
docker cp ./influxdb_backup_20260107 influxdb:/tmp/restore
docker exec influxdb influxd restore -portable -db iot_data /tmp/restore
docker compose restart influxdb
```

**Grafana:**

```bash
docker compose stop grafana
docker cp ./grafana_backup_20260107/. grafana:/var/lib/grafana/
docker compose start grafana
```

**Node-RED:**

```bash
docker compose stop nodered
docker cp ./nodered_backup_20260107/. nodered:/data/
docker compose start nodered
```

### Restore Configuration

**Docker Compose:**

```bash
cp docker-compose_20260107.yml ~/IOTstack/docker-compose.yml
cd ~/IOTstack
docker compose up -d
```

**Node-RED Flows:**

1. Node-RED UI → Menu → **Import**
2. Select `flows_backup_20260107.json`
3. Import
4. Deploy

**Grafana Dashboards:**

1. Grafana UI → Dashboards → **Import**
2. Upload `dashboard_backup_20260107.json`
3. Select data source
4. Import

### Restore Raspberry Pi

**Scripts:**

```bash
scp pi_scripts_backup/*.sh <pi_username>@<pi_tailscale_ip>:/home/<pi_username>/
scp pi_scripts_backup/gadget.sh <pi_username>@<pi_tailscale_ip>:/tmp/
ssh <pi_username>@<pi_tailscale_ip> 'sudo mv /tmp/gadget.sh /usr/local/bin/ && sudo chmod +x /usr/local/bin/gadget.sh'
```

**Tailscale State:**

```bash
# On VM (Docker)
sudo tar xzf tailscale_state_20260107.tar.gz -C ~/IOTstack/
docker compose restart tailscale

# On VM (native)
sudo tar xzf tailscale_state_vm_20260107.tar.gz -C /
sudo systemctl restart tailscaled

# On Pi
scp tailscale_state_pi_20260107.tar.gz <pi_username>@<pi_tailscale_ip>:/tmp/
ssh <pi_username>@<pi_tailscale_ip> 'sudo tar xzf /tmp/tailscale_state_pi_20260107.tar.gz -C / && sudo systemctl restart tailscaled'
```

**Note:** After restoring Tailscale state, verify connectivity:

```bash
# Check status
tailscale status  # or: docker exec tailscale tailscale status

# If not authenticated, re-authenticate
tailscale up  # or: docker exec tailscale tailscale up
```

**Regenerate USB Drive:**

If `drive.bin` is corrupted:

```bash
ssh <pi_username>@<pi_tailscale_ip>
sudo dd if=/dev/zero of=/home/<pi_username>/drive.bin bs=1M count=100
sudo mkfs.vfat /home/<pi_username>/drive.bin
sudo mount -o loop /home/<pi_username>/drive.bin /mnt
sudo mkdir -p /mnt/SAMSUNG_E-Paper
sudo umount /mnt
```

## Disaster Recovery

### Complete System Loss

**Scenario**: Proxmox host failure or complete hardware loss

**Recovery Steps:**

1. **Reinstall Proxmox** on new/repaired hardware
2. **Reconnect 2TB external SSD** to new Proxmox host
3. **Restore VM from latest backup**:
   - In Proxmox: Local storage → Backups
   - Select most recent backup from external SSD
   - Click Restore, configure VM ID and storage
4. **Verify VM boots** and all Docker services start
5. **Check InfluxDB data** integrity (if RAID array survived)
6. **Test connectivity** to Raspberry Pi via Tailscale
7. **Verify end-to-end functionality** (data collection → display)

**Time Estimate:** 2-4 hours (if backup storage intact)

**Recovery Time Objective (RTO):** 4 hours  
**Recovery Point Objective (RPO):** 24 hours (last daily backup)

### RAID5 Disk Failure

**Scenario**: Single disk failure in RAID5 array

**Symptoms:**
- RAID degraded mode warning
- System still operational (data accessible)
- Performance may be reduced

**Recovery Steps:**

1. **Identify failed disk**:
   ```bash
   cat /proc/mdstat
   sudo mdadm --detail /dev/md0
   ```

2. **Order replacement disk** (same size or larger)

3. **Replace failed disk**:
   ```bash
   # Remove failed disk from array
   sudo mdadm --manage /dev/md0 --remove /dev/sdX
   
   # Physically replace the disk
   
   # Add new disk to array
   sudo mdadm --manage /dev/md0 --add /dev/sdX
   ```

4. **Monitor rebuild**:
   ```bash
   watch cat /proc/mdstat
   ```

5. **Verify array health** after rebuild completes

**Time Estimate:** 4-12 hours (depending on rebuild speed)

!!! danger "Critical: Act Immediately"
    RAID5 can only survive ONE disk failure. If a second disk fails during rebuild, ALL DATA IS LOST. Replace failed disks promptly!

### RAID5 Complete Failure

**Scenario**: Multiple disk failures or RAID controller failure

**Recovery Steps:**

1. **Accept InfluxDB data loss** (if no recent backup exists)
2. **Restore VM** from 2TB external SSD backup
3. **Reconfigure RAID5** or use alternative storage
4. **System starts collecting new data immediately**
5. **Historical data lost** (time-series data unrecoverable without backup)

**Prevention**: Create monthly InfluxDB backups to external SSD

### Data Loss (InfluxDB)

**If manual backup available:**

1. Stop InfluxDB: `docker compose stop influxdb`
2. Restore from backup:
   ```bash
   docker exec influxdb influxd restore -portable -db iot_data /tmp/restore
   ```
3. Start InfluxDB: `docker compose start influxdb`
4. Verify data: Check Grafana dashboards

**If no backup (RAID5 failed completely):**

1. Accept historical data loss
2. System continues collecting new data
3. Dashboards will only show data from restore point forward

**Lesson**: Implement monthly InfluxDB backups to external SSD

### External SSD Failure

**Scenario**: 2TB external SSD containing VM backups fails

**Impact:**
- VM backups lost
- Active VM still running (no immediate impact)
- No recovery point if VM fails

**Recovery Steps:**

1. **Replace external SSD immediately**
2. **Reconfigure Proxmox backup job** to new storage
3. **Create new manual backup** as soon as possible
4. **Consider implementing offsite backup** to prevent future single-point failure

**Time to Recovery:** Immediate (VM still running), but no safety net until new backup exists

### Raspberry Pi Failure

**Replacement Steps:**

1. **Image new SD card** with Raspberry Pi OS
2. **Configure network** (static IP, Tailscale)
3. **Install scripts** from backup
4. **Configure SSH keys** from VM
5. **Set up gadget mode** (see [Gadget Mode Setup](../raspberry-pi/gadget-mode.md))
6. **Test** image transfer and display update

**Time Estimate:** 1-2 hours

## Offsite Backup

### Why Offsite?

Protects against:

- Physical disaster (fire, flood)
- Theft
- Hardware failure

### Methods

**Cloud Storage:**

```bash
# Install rclone
sudo apt install rclone

# Configure cloud provider
rclone config

# Sync backups
rclone sync ~/IOTstack/backups/ remote:backups/
```

**Remote Server:**

```bash
# Rsync to remote
rsync -avz ~/IOTstack/backups/ user@remote-server:/backups/
```

**Automated:**

```bash
crontab -e
```

Add:

```cron
# Daily offsite backup at 4 AM
0 4 * * * rclone sync ~/IOTstack/backups/ remote:backups/
```

## Backup Verification

### Test Backups Regularly

**Monthly Test:**

1. Restore to test VM
2. Verify services start
3. Check data is accessible
4. Delete test VM

### Verification Checklist

- [ ] Backup files are not corrupted
- [ ] Backup size is reasonable (not 0 bytes)
- [ ] Restore completes without errors
- [ ] All services start after restore
- [ ] Data is accessible and current
- [ ] Configuration is preserved

## Backup Storage

### Current Storage Infrastructure

**Primary Backup Storage:**

1. **2TB External SSD** (Proxmox VM backups)
   - Connected to Proxmox host
   - Dedicated for VM snapshots
   - Daily automated backups with 7-day retention
   - Current usage: ~210GB (7 backups × 30GB each)
   - Available capacity: ~1.8TB

2. **RAID5 Array** (InfluxDB data)
   - 5× 2TB SSDs in RAID5 configuration
   - Usable capacity: ~8TB
   - Continuous data protection with single-disk failure tolerance
   - Current usage: Variable (depends on data retention)
   - Dedicated for time-series database storage

### Storage Capacity Planning

**2TB External SSD:**
- Estimated VM backup size: 20-30GB per backup
- 7 daily backups: ~210GB
- Growth headroom: ~1.8TB available
- **Recommendation**: Monitor monthly, consider cleanup if usage exceeds 1TB

**RAID5 Array:**
- InfluxDB growth rate: ~500MB per month (estimate)
- Current capacity: ~8TB usable
- **Recommendation**: Monitor RAID status weekly, plan for expansion if usage exceeds 6TB

### Storage Monitoring

**Check External SSD usage:**

```bash
# On Proxmox host
df -h /mnt/pve/<backup-storage-name>
```

**Check RAID5 array usage:**

```bash
# RAID health
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Storage usage
df -h /path/to/raid/mount
```

### Storage Locations

**Local (Current Setup):**

- ✅ 2TB External SSD (Proxmox host) - VM backups
- ✅ RAID5 Array (8TB usable) - InfluxDB data
- ⚠️ **Missing**: Offsite backup location

**Offsite (Recommended):**

- Cloud storage (AWS S3, Backblaze B2, Google Drive)
- Remote server at different physical location
- Encrypted external drive stored offsite

**Best Practice:** 3-2-1 Rule

- **3** copies of data
  - Original (VM/database)
  - Local backup (2TB SSD / RAID5)
  - **Missing: Offsite copy**
- **2** different media types ✅ (SSD + RAID array)
- **1** offsite copy ⚠️ **Not yet implemented**

!!! warning "Offsite Backup Recommended"
    Current setup protects against hardware failure but NOT against:
    
    - Physical disaster (fire, flood) at Proxmox location
    - Theft of hardware
    - Complete site loss
    
    **Action**: Implement periodic offsite backup of critical data (VM snapshots, InfluxDB monthly snapshots)

## Backup Security

### Encryption

Encrypt backups before offsite storage:

```bash
# Encrypt
gpg -c backup_file.tar.gz

# Decrypt
gpg backup_file.tar.gz.gpg
```

### Access Control

- Limit who can access backups
- Use strong passwords
- Enable MFA on cloud storage

### Audit Logs

Monitor backup access:

```bash
# Check recent access
ls -lt ~/IOTstack/backups/ | head
```

## Documentation Backup

### Critical Documents

- This manual (all markdown files)
- Network diagrams
- Password vault export
- Contact information
- License keys

### Version Control

Store documentation in Git:

```bash
cd /path/to/docs
git init
git add .
git commit -m "Initial commit"
git remote add origin <repository-url>
git push -u origin main
```

### Offsite Documentation

- Print important pages
- Store in physical safe
- Share with trusted colleague

## Recovery Time Objectives

| Scenario | Target RTO | Target RPO |
| -------- | ---------- | ---------- |
| Single container failure | 5 minutes | 0 (no data loss) |
| VM failure | 30 minutes | 24 hours |
| Complete system loss | 4 hours | 24 hours |
| Data corruption | 1 hour | 24 hours |

**RTO:** Recovery Time Objective (how fast can we restore?)
**RPO:** Recovery Point Objective (how much data can we lose?)

## Backup Maintenance

### Regular Tasks

**Weekly:**

- Verify backups completed successfully
- Check backup storage usage

**Monthly:**

- Test restore procedure
- Review backup retention
- Clean old backups

**Quarterly:**

- Review backup strategy
- Update documentation
- Test disaster recovery plan

### Monitoring

Set up alerts for:

- Backup job failures
- Low disk space on backup storage
- Backup age >48 hours

## Backup Costs

### Storage Costs

Estimate monthly:

- Local NAS: $0 (one-time hardware cost)
- Cloud storage (250GB): $5-15/month
- Remote server: $10-50/month

### Time Costs

- Setup: 4 hours initially
- Maintenance: 1 hour/month
- Testing: 2 hours/quarter

## Retention Policies

### Short-Term (Daily)

- Keep 7 days of daily backups
- Automatic deletion of older backups

### Medium-Term (Weekly)

- Keep 4 weeks of weekly backups
- Manually delete older backups

### Long-Term (Monthly)

- Keep 3-12 months of monthly backups
- Archive to cheaper storage

### Script Example

```bash
#!/bin/bash
# cleanup_old_backups.sh

BACKUP_DIR=~/IOTstack/backups
DAYS_TO_KEEP=7

find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$DAYS_TO_KEEP -delete
```

Run via cron:

```cron
0 5 * * 0 /path/to/cleanup_old_backups.sh
```
