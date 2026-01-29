# Regular Maintenance Tasks

This guide outlines routine maintenance to keep the system running smoothly.

## Daily Tasks

### Monitor System Status

**Check Grafana Dashboard**

1. Navigate to `http://<vm_ip>:3000`
2. Verify dashboard is displaying current data
3. Check for data gaps or anomalies

**Check E-Ink Display**

1. Verify display shows latest image
2. Check image quality and clarity
3. Ensure refresh is occurring

**Quick Health Check**

```bash
# SSH to VM
ssh <vm_username>@<vm_ip>

# Check all containers
cd ~/IOTstack
docker compose ps
```

All should show "Up" status.

## Weekly Tasks

### Review Logs

**Node-RED Logs**

```bash
docker compose logs --tail=100 nodered
```

Look for:

- HTTP request errors
- InfluxDB write failures
- SSH/SCP errors

**Grafana Logs**

```bash
docker compose logs --tail=100 grafana
```

Look for:

- Render errors
- Data source issues

**InfluxDB Logs**

```bash
docker compose logs --tail=100 influxdb
```

Look for:

- Memory warnings
- Write errors

### Check Disk Space

**On VM:**

```bash
df -h
```

Ensure root partition has >20% free.

**Docker Volumes:**

```bash
docker system df
```

Shows Docker disk usage.

### Verify Backups

Check that automated backups are running:

```bash
# Proxmox backups
ls -lh /path/to/backup/directory/

# IOTstack backups
ls -lh ~/IOTstack/backups/
```

### Check Raspberry Pi

```bash
ssh <pi_username>@<pi_tailscale_ip>

# Disk space
df -h

# Temperature
vcgencmd measure_temp

# Throttling
vcgencmd get_throttled
```

Output `0x0` means no issues.

### Check Tailscale Connectivity

**Verify VM Tailscale status:**

```bash
# If using Docker
docker exec tailscale tailscale status

# If native install
sudo tailscale status
```

**Verify Pi connectivity via Tailscale:**

```bash
ping -c 4 <pi_tailscale_ip>
```

**Test SSH over Tailscale:**

```bash
docker exec nodered ssh <pi_username>@<pi_tailscale_ip> 'echo OK'
```

Should return "OK" without errors.

## Monthly Tasks

### Update Docker Images

**Pull Latest Images:**

```bash
cd ~/IOTstack
docker compose pull
```

**Apply Updates:**

```bash
docker compose up -d
```

Containers will be recreated with new images.

**Verify:**

```bash
docker compose ps
docker compose logs -f
```

Check for errors after update.

### Update Raspberry Pi OS

```bash
ssh <pi_username>@<pi_tailscale_ip>
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Wait for reboot, verify services:

```bash
ssh <pi_username>@<pi_tailscale_ip>
systemctl status usb-gadget
sudo tailscale status
```

### Update Proxmox Host

```bash
ssh root@<proxmox-ip>
apt update
apt dist-upgrade -y
```

Reboot if kernel updated.

### Clean Docker System

Remove unused images and containers:

```bash
cd ~/IOTstack
docker system prune -f
```

For more aggressive cleanup (removes all unused data):

```bash
docker system prune -a --volumes -f
```

**Warning:** This removes all unused volumes. Backup first.

### Check InfluxDB Database Size

```bash
docker exec influxdb du -sh /var/lib/influxdb
```

If size is concerning, consider:

- Implementing retention policy
- Downsampling old data
- Archiving to external storage

### Refresh E-Ink Display

Prevent ghosting:

1. Access display menu
2. Settings → Display → Screen Refresh
3. Run full refresh cycle

### Test Backups

Periodically test backup restoration:

1. Restore a Proxmox VM backup to test VM
2. Verify services start correctly
3. Delete test VM after verification

## Quarterly Tasks

### Review Security

**Update Passwords:**

- Grafana admin password
- Node-RED password
- SSH keys (rotate if policy requires)

**Review Access Logs:**

```bash
# On VM
sudo grep 'Failed password' /var/log/auth.log

# On Raspberry Pi
ssh <pi_username>@<pi_tailscale_ip>
sudo grep 'Failed password' /var/log/auth.log
```

**Check Tailscale Admin Console:**

1. Log in to [Tailscale admin console](https://login.tailscale.com/admin/machines)
2. Review connected devices
3. Check for unauthorized devices
4. Review access logs and activity
5. Verify authentication keys are current

**Update Firewall Rules:**

Review and update as needed:

```bash
sudo ufw status numbered
```

### Performance Review

**InfluxDB Queries:**

Check query performance in Grafana. Slow queries may need optimization.

**Node-RED Flow:**

Review flow for inefficiencies:

- Unnecessary nodes
- Debug nodes left active
- Excessive logging

**Resource Usage:**

```bash
docker stats --no-stream
```

Note high CPU or memory usage.

### Documentation Review

Update this documentation with:

- Configuration changes
- New issues discovered
- Workarounds implemented
- Contact information updates

### Hardware Inspection

**Physical Checks:**

- Inspect cables for damage
- Check PoE switch ventilation
- Clean dust from equipment
- Verify all LEDs indicate normal operation

**Network:**

- Test network speed: `speedtest-cli`
- Check for packet loss: `ping -c 100 <local_gateway_ip>`
- Review switch logs (if managed)

## Yearly Tasks

### Full System Backup

**Proxmox:**

1. Backup all VMs
2. Export configuration
3. Document any custom settings

**Node-RED:**

1. Export all flows
2. Backup settings.js
3. Document installed nodes

**Grafana:**

1. Export all dashboards
2. Backup data sources configuration
3. Document plugins

**Documentation:**

Create archive of all documentation and configuration files.

### Disaster Recovery Test

Simulate failure and recovery:

1. **Test VM restore** from backup
2. **Test network failure** recovery
3. **Test power failure** recovery
4. **Document recovery time** and issues

### Capacity Planning

Review growth and plan for future:

**Data Growth:**

```bash
# InfluxDB size over time
docker exec influxdb influx -execute "SHOW STATS FOR 'database'"
```

**Resource Usage Trends:**

Review historical:

- CPU usage
- Memory usage
- Disk usage
- Network bandwidth

**Plan for:**

- Storage expansion
- Additional services
- Performance upgrades

### Firmware and Software Updates

**Proxmox:**

Check for major version updates: [https://www.proxmox.com/](https://www.proxmox.com/)

**IOTstack:**

Check for breaking changes: [https://github.com/SensorsIot/IOTstack](https://github.com/SensorsIot/IOTstack)

**Samsung Display:**

Check for firmware updates from manufacturer.

### Review SLA/Uptime

Calculate system uptime and reliability:

```bash
uptime
```

Document:

- Total uptime %
- Number of incidents
- Mean time to recovery
- Lessons learned

## Maintenance Calendar

| Task | Frequency | Estimated Time |
| ---- | --------- | -------------- |
| Monitor status | Daily | 5 min |
| Review logs | Weekly | 15 min |
| Check backups | Weekly | 10 min |
| Update Docker | Monthly | 30 min |
| Update OS | Monthly | 30 min |
| Security review | Quarterly | 1 hour |
| Full backup | Yearly | 2 hours |
| DR test | Yearly | 4 hours |

## Automation Opportunities

### Automated Monitoring

Set up alerts for:

- Container failures
- Disk space <20%
- High CPU/memory
- Network issues

Use tools like:

- Grafana alerts (email/Slack)
- Prometheus + Alertmanager
- Uptime Kuma

### Automated Backups

Schedule backups via cron:

```bash
crontab -e
```

Add:

```cron
# Daily backup at 2 AM
0 2 * * * cd ~/IOTstack && ./scripts/backup.sh

# Weekly cleanup at 3 AM Sunday
0 3 * * 0 docker system prune -f
```

### Automated Updates

Consider using:

- Watchtower (auto-update Docker containers)
- Unattended-upgrades (auto-update OS)

**Warning:** Test in non-production first.

## Maintenance Log

Keep a log of all maintenance activities:

```
Date: 2026-01-07
Task: Monthly Docker updates
Result: All containers updated successfully
Issues: None
Time: 25 minutes
```

Store in:

- Text file
- Spreadsheet
- Ticketing system
- Wiki

## Contact Information

Document key contacts:

| Role | Contact | Notes |
| ---- | ------- | ----- |
| System Administrator | Kieran Julien | Primary contact |
| System Administrator | CoE HRTech | Secondary contact |
| Network (Modem) | Montreal Solutions | Modem management, see their guide |
| Proxmox Support | (Community/Commercial) | For Proxmox issues |
| Hardware Vendor | Samsung | Display support |

## Escalation Procedures

For critical issues:

1. **Level 1**: Check troubleshooting guide
2. **Level 2**: Review logs and recent changes
3. **Level 3**: Contact system administrator
4. **Level 4**: Engage vendor support

Document resolution for future reference.
