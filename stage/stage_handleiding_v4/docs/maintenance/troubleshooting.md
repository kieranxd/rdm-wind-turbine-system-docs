# Troubleshooting Guide

Common issues and their solutions.

## System-Wide Issues

### All Services Offline

**Symptoms:**

- Cannot access Grafana, Node-RED, InfluxDB
- Docker containers not running

**Diagnosis:**

```bash
ssh <vm_username>@<vm_ip>
docker compose ps
```

**Solution:**

```bash
cd ~/IOTstack
docker compose up -d
```

Check logs for errors:

```bash
docker compose logs -f
```

### Proxmox VM Won't Start

**Symptoms:**

- VM shows "stopped" status in Proxmox
- Cannot SSH to VM

**Solution:**

1. In Proxmox web interface, select VM
2. Click **Start**
3. Open **Console** to see boot messages
4. If boot fails, check:
   - Disk space on Proxmox host
   - VM configuration (RAM, CPU limits)
   - Proxmox logs: `journalctl -u pveproxy`

### Network Connectivity Issues

**Symptoms:**

- Services cannot reach internet
- Services cannot reach each other

**Diagnosis:**

```bash
# Test internet
ping 8.8.8.8

# Test gateway
ping <local_gateway_ip>

# Test DNS
nslookup google.com
```

**Solution:**

Check network configuration (see [VM Setup](../infrastructure/vm-setup.md#set-static-ip-recommended)).

If DHCP:

```bash
sudo dhclient -r  # Release
sudo dhclient     # Renew
```

If static IP, verify gateway and DNS in network config.

## Docker/Container Issues

### Container Keeps Restarting

**Symptoms:**

- Container status shows "Restarting"
- `docker compose ps` shows restart count increasing

**Diagnosis:**

```bash
docker compose logs <service-name>
```

**Common Causes:**

1. **Configuration error**: Check environment variables
2. **Port conflict**: Another service using same port
3. **Resource limit**: Container out of memory

**Solutions:**

**Port Conflict:**

```bash
sudo netstat -tlnp | grep <port>
```

Change port in `docker-compose.yml` or stop conflicting service.

**Memory Limit:**

Edit `docker-compose.yml`:

```yaml
services:
  <service>:
    mem_limit: 2g
```

Restart:

```bash
docker compose up -d <service>
```

### Container Not Starting

**Symptoms:**

- Container shows "Exited" status
- Won't stay running

**Diagnosis:**

```bash
docker compose logs --tail=50 <service>
```

**Solutions:**

1. **Remove and recreate:**

```bash
docker compose rm -f <service>
docker compose up -d <service>
```

2. **Check volume permissions:**

```bash
ls -la ~/IOTstack/volumes/<service>/
sudo chown -R 1000:1000 ~/IOTstack/volumes/<service>/
```

3. **Pull fresh image:**

```bash
docker compose pull <service>
docker compose up -d <service>
```

### Docker Disk Full

**Symptoms:**

- "no space left on device" errors
- Containers failing to start

**Diagnosis:**

```bash
df -h
docker system df
```

**Solution:**

```bash
# Remove unused containers, networks, images
docker system prune -a

# For aggressive cleanup (includes volumes)
docker system prune -a --volumes
```

**Warning:** Backup first if removing volumes.

## Node-RED Issues

### Flow Not Running

**Symptoms:**

- Inject nodes don't trigger
- No data flowing through

**Diagnosis:**

1. Check Node-RED is running:

```bash
docker compose ps nodered
```

2. Open Node-RED UI: `http://<vm_ip>:1880`
3. Check deploy status (should say "Deployed")

**Solution:**

1. Click **Deploy** button
2. Check for errors in debug panel
3. Verify inject nodes are enabled (blue badge under node)

### HTTP Request Timeout

**Symptoms:**

- Weather API or eGauge requests failing
- "timeout" errors in debug

**Diagnosis:**

Check debug output:

```
"Error: timeout of 60000ms exceeded"
```

**Solution:**

1. **Test URL manually:**

```bash
curl "https://weerlive.nl/api/weerlive_api_v2.php?key=3c2224ecab&locatie=51.89908,4.42348"
```

2. **Increase timeout** in HTTP request node:
   - Double-click node
   - Set timeout to 120 seconds
   - Deploy

3. **Check network connectivity** from VM

### InfluxDB Write Failures

**Symptoms:**

- Debug shows "Failed to write to InfluxDB"
- Data not appearing in Grafana

**Diagnosis:**

```bash
# Test InfluxDB reachable
curl http://influxdb:8086/ping
```

If running outside Node-RED:

```bash
docker exec nodered curl http://influxdb:8086/ping
```

**Solution:**

1. **Verify InfluxDB is running:**

```bash
docker compose ps influxdb
docker compose logs influxdb
```

2. **Check database exists:**

```bash
docker exec influxdb influx -execute "SHOW DATABASES"
```

If `iot_data` missing:

```bash
docker exec influxdb influx -execute "CREATE DATABASE iot_data"
```

3. **Verify InfluxDB node configuration:**
   - Server: `influxdb`
   - Port: `8086`
   - Database: `iot_data`

### SSH/SCP to Raspberry Pi Fails

**Symptoms:**

- "Permission denied" errors
- "Connection refused"
- SCP command fails in exec node

**Diagnosis:**

Test SSH manually:

```bash
docker exec nodered ssh <pi_username>@<pi_tailscale_ip> 'echo test'
```

**Solution:**

See [SSH Configuration](../raspberry-pi/ssh-setup.md) for complete setup.

Quick checks:

1. **Tailscale is connected:**

```bash
# On VM (if Docker)
docker exec tailscale tailscale status

# On VM (if native)
sudo tailscale status

# On Pi
sudo tailscale status
```

2. **Pi is reachable via Tailscale:**

```bash
ping <pi_tailscale_ip>  # e.g., ping 100.64.1.5
```

3. **SSH keys configured:**

```bash
docker exec nodered ls -la /root/.ssh/
```

Should show `id_ed25519` and `id_ed25519.pub`.

4. **Known hosts:**

```bash
docker exec nodered ssh-keyscan -H <pi_tailscale_ip> >> /root/.ssh/known_hosts
```

5. **SSH volume mounted:**

Check `docker-compose.yml` has:
```yaml
volumes:
  - /home/<user>/.ssh:/root/.ssh:ro
```

## Grafana Issues

### Can't Login

**Symptoms:**

- Forgot admin password
- Login page not loading

**Solution:**

**Reset Password:**

```bash
docker exec -it grafana grafana-cli admin reset-admin-password newpassword
```

Replace `newpassword` with your new password.

**Login Page Not Loading:**

Check Grafana is running:

```bash
docker compose ps grafana
docker compose logs grafana
```

Restart:

```bash
docker compose restart grafana
```

### Dashboard Shows "No Data"

**Symptoms:**

- Dashboard panels empty
- Shows "No data" message

**Diagnosis:**

1. **Check data source:** Configuration → Data Sources → InfluxDB-IoT → Test
2. **Verify data exists:**

```bash
docker exec influxdb influx -execute "SELECT * FROM weather LIMIT 5"
```

3. **Check time range:** Dashboard may be showing wrong time period

**Solution:**

1. **Adjust time range:** Top right corner, select "Last 3 hours"
2. **Check query:** Edit panel → View query → Fix any syntax errors
3. **Refresh:** Click refresh button or Ctrl+R

### Image Rendering Fails

**Symptoms:**

- Node-RED shows render errors
- Images not generated

**Diagnosis:**

```bash
docker compose ps grafana-image-renderer
docker compose logs grafana-image-renderer
```

**Solution:**

1. **Verify renderer is running:**

```bash
docker compose up -d grafana-image-renderer
```

2. **Check token matches:**

In `docker-compose.yml`, verify:

- Grafana: `GF_RENDERING_RENDERER_TOKEN`
- Renderer: `AUTH_TOKEN`

Are identical.

3. **Test renderer directly:**

```bash
curl -H "Authorization: Bearer <token>" \
  "http://<vm_ip>:8081/" 
```

Should return "Grafana Image Renderer".

## InfluxDB Issues

### Database Not Found

**Symptoms:**

- "database not found" errors
- Grafana can't connect

**Solution:**

```bash
docker exec influxdb influx -execute "CREATE DATABASE iot_data"
```

Verify:

```bash
docker exec influxdb influx -execute "SHOW DATABASES"
```

### High Memory Usage

**Symptoms:**

- InfluxDB container using >80% memory
- Slow queries

**Diagnosis:**

```bash
docker stats influxdb --no-stream
```

**Solution:**

1. **Increase memory limit:**

Edit `docker-compose.yml`:

```yaml
influxdb:
  mem_limit: 2g
```

Restart:

```bash
docker compose up -d influxdb
```

2. **Implement retention policy:**

```bash
docker exec influxdb influx -execute \
  "CREATE RETENTION POLICY 90_days ON iot_data DURATION 90d REPLICATION 1 DEFAULT"
```

3. **Downsample old data** (see [InfluxDB docs](../services/influxdb.md))

## Raspberry Pi Issues

### Pi Not Reachable

**Symptoms:**

- Cannot SSH to Pi
- Ping fails

**Diagnosis:**

```bash
# Test via Tailscale
ping <pi_tailscale_ip>  # e.g., ping 100.64.1.5

# Test via local network (if on same network)
ping <pi_local_ip>  # e.g., ping 192.168.178.148
```

**Solution:**

1. **Check Tailscale connectivity:**

```bash
# On VM (Docker)
docker exec tailscale tailscale status

# On VM (native)
sudo tailscale status
```

Verify Pi is shown as online in the status output.

2. **Check physical connection:** PoE+ cable connected?

3. **Check PoE+ power:** LEDs on Pi should be lit

4. **Check Tailscale on Pi:**

Connect via monitor/keyboard or local network:

```bash
sudo tailscale status
```

If not connected:

```bash
sudo tailscale up
```

5. **Check network configuration on Pi:**

```bash
ip a
```

Verify both local IP and Tailscale IP are assigned.

6. **Restart Pi:**

```bash
sudo reboot
```

Or power cycle (unplug PoE, wait, replug).

### USB Gadget Not Working

**Symptoms:**

- Display doesn't detect USB drive
- Files not accessible

**Diagnosis:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'cat /sys/kernel/config/usb_gadget/EInkGadget/UDC'
```

Should show UDC name, not empty.

**Solution:**

1. **Restart gadget service:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'sudo systemctl restart usb-gadget'
```

2. **Check gadget script:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'sudo /usr/local/bin/gadget.sh'
```

Check for errors.

3. **Verify module loaded:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'lsmod | grep -E "dwc2|libcomposite"'
```

If missing:

```bash
ssh <pi_username>@<pi_tailscale_ip> 'sudo modprobe dwc2 && sudo modprobe libcomposite'
```

4. **Check cable:** Try different USB-C cable

### Image Not Updating on Display

**Symptoms:**

- Old image still showing
- New image transferred but not displayed

**Diagnosis:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'ls -lh /home/<pi_username>/eink_usb/SAMSUNG_E-Paper/'
```

Verify new image exists.

**Solution:**

1. **Run refresh script:**

```bash
ssh <pi_username>@<pi_tailscale_ip> '/home/<pi_username>/refresh_script.sh'
```

2. **Check script output** for errors
3. **Manually refresh display:** Use display menu to refresh
4. **Verify USB is bound:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'cat /sys/kernel/config/usb_gadget/EInkGadget/UDC'
```

Should not be empty.

## Tailscale Issues

### Tailscale Not Connected

**Symptoms:**

- Cannot ping Tailscale IPs
- SSH fails over Tailscale
- `tailscale status` shows offline

**Diagnosis:**

**On VM (Docker):**

```bash
docker ps | grep tailscale
docker exec tailscale tailscale status
```

**On VM (native) or Pi:**

```bash
sudo tailscale status
```

**Solution:**

**If Docker container not running:**

```bash
cd ~/IOTstack
docker compose up -d tailscale
```

**If not authenticated:**

```bash
# Docker
docker exec tailscale tailscale up

# Native
sudo tailscale up
```

Follow the authentication link in your browser.

**If service stopped (native install):**

```bash
sudo systemctl start tailscaled
sudo systemctl enable tailscaled
```

### Tailscale IP Changed

**Symptoms:**

- SSH suddenly fails
- Node-RED can't connect to Pi

**Diagnosis:**

Get current IPs:

```bash
# On VM
docker exec tailscale tailscale ip -4  # or: tailscale ip -4

# On Pi
tailscale ip -4
```

**Solution:**

Tailscale IPs are normally persistent, but if changed:

1. **Update Node-RED flows** with new Pi Tailscale IP
2. **Update known_hosts:**

```bash
docker exec nodered ssh-keyscan -H <new_pi_tailscale_ip> >> /root/.ssh/known_hosts
```

3. **Test connection:**

```bash
ssh <pi_username>@<new_pi_tailscale_ip>
```

### Tailscale Firewall Blocked

**Symptoms:**

- Tailscale won't connect
- Shows "connecting" but never succeeds

**Diagnosis:**

Tailscale needs outbound HTTPS (443) and UDP (41641) access.

**Solution:**

1. **Check firewall rules** on VM and Pi
2. **Contact network administrator** if on restricted network
3. **Try different network** to isolate issue
4. **Check logs:**

```bash
# Docker
docker logs tailscale

# Native
sudo journalctl -u tailscaled -n 50
```

### Devices Can't See Each Other

**Symptoms:**

- Both devices online in Tailscale
- Can't ping each other

**Diagnosis:**

```bash
# From VM
ping <pi_tailscale_ip>

# From Pi
ping <vm_tailscale_ip>
```

**Solution:**

1. **Check ACLs** in Tailscale admin console:
   - Go to [https://login.tailscale.com/admin/acls](https://login.tailscale.com/admin/acls)
   - Ensure devices can communicate

2. **Verify both on same tailnet:**

```bash
tailscale status
```

Check the tailnet name matches.

3. **Restart Tailscale** on both devices:

```bash
# Docker (VM)
docker compose restart tailscale

# Native (Pi)
sudo systemctl restart tailscaled
```

## Display Issues

### Display Not Powering On

See [PoE Troubleshooting](../hardware/poe.md#device-not-powering-on).

### Display Showing Wrong Image

**Symptoms:**

- Old image displayed
- Image from wrong source

**Solution:**

1. **Check display source:** Ensure USB mode is selected
2. **Refresh display:** Manually trigger refresh
3. **Verify image on USB:**

```bash
ssh <pi_username>@<pi_tailscale_ip> 'sudo mount -o loop /home/<pi_username>/drive.bin /mnt && ls -lh /mnt/SAMSUNG_E-Paper/ && sudo umount /mnt'
```

### Ghosting on Display

**Symptoms:**

- Previous image faintly visible
- Image blurry or overlapping

**Solution:**

1. **Run full refresh:** Display menu → Screen Refresh
2. **Enable full refresh mode:** Display settings → Refresh Mode → Full
3. **Vary content:** Avoid static images for long periods

## Performance Issues

### Slow Response Times

**Symptoms:**

- Grafana slow to load
- Node-RED UI laggy
- API requests timing out

**Diagnosis:**

```bash
docker stats --no-stream
top
htop
```

Check CPU and memory usage.

**Solution:**

1. **Increase VM resources** in Proxmox:
   - More CPU cores
   - More RAM (8GB recommended)

2. **Optimize queries** in Grafana (use appropriate time ranges)

3. **Reduce Node-RED flow complexity**

4. **Clean Docker:**

```bash
docker system prune -a
```

### High Disk I/O

**Diagnosis:**

```bash
iotop
```

**Solution:**

- Implement InfluxDB retention policy
- Move Docker volumes to faster storage (SSD)
- Reduce write frequency in Node-RED

## Data Issues

### Missing Data in Grafana

**Symptoms:**

- Gaps in graphs
- Some fields missing

**Diagnosis:**

```bash
docker exec influxdb influx -execute \
  "SELECT * FROM turbine ORDER BY time DESC LIMIT 10"
```

**Solution:**

1. **Check Node-RED is running** and flows deployed
2. **Review Node-RED debug** for API errors
3. **Verify APIs are accessible:**

```bash
curl "https://weerlive.nl/api/weerlive_api_v2.php?key=3c2224ecab&locatie=51.89908,4.42348"
```

### Incorrect Data Values

**Symptoms:**

- Values don't make sense
- Null or NaN values

**Diagnosis:**

Check Node-RED function node for parsing errors:

- Enable debug on function output
- Verify field names match database

**Solution:**

Review and fix function node code (see [Node-RED Configuration](../services/node-red.md)).

## Getting Help

### Collect Information

Before seeking help, gather:

1. **Error messages** from logs
2. **Docker status:** `docker compose ps`
3. **System info:** `uname -a`, `df -h`
4. **Recent changes** made to system

### Resources

- **IOTstack GitHub:** [Issues](https://github.com/SensorsIot/IOTstack/issues)
- **Node-RED Forum:** [https://discourse.nodered.org/](https://discourse.nodered.org/)
- **Grafana Community:** [https://community.grafana.com/](https://community.grafana.com/)
- **InfluxDB Forums:** [https://community.influxdata.com/](https://community.influxdata.com/)

### Create Support Ticket

Include:

- Problem description
- Steps to reproduce
- Logs (relevant sections)
- System configuration
- What you've tried

## Emergency Procedures

### System Recovery

If system is completely broken:

1. **Restore from Proxmox backup**
2. **Verify services start**
3. **Test each component**
4. **Review what caused failure**
5. **Update procedures to prevent recurrence**

### Data Loss

If InfluxDB data is lost:

1. **Restore from backup** (if available)
2. **Accept data loss** and start fresh
3. **Implement better backup strategy** going forward

### Hardware Failure

1. **Identify failed component**
2. **Source replacement**
3. **Install and configure**
4. **Restore from backups**
5. **Document in maintenance log**
