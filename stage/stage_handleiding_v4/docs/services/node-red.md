# Node-RED Configuration

Node-RED is the automation and orchestration engine that connects all components.

## Initial Access

- URL: `http://<vm_ip>:1880`
  - Replace `<vm_ip>` with your VM's IP address (e.g., `192.168.178.164`)
- No default authentication (configure security first)

## Enable Security (Recommended)

### Generate Password Hash

```bash
docker exec -it nodered node-red-admin hash-pw
```

Enter your desired password. Copy the hash (starts with `$2b$`).

### Edit Settings

```bash
cd ~/IOTstack/volumes/nodered/data
nano settings.js
```

Find the `adminAuth` section and uncomment/edit:

```javascript
adminAuth: {
    type: "credentials",
    users: [{
        username: "admin",
        password: "$2b$08$...(your hash here)",
        permissions: "*"
    }]
},
```

Save and restart Node-RED:

```bash
docker compose restart nodered
```

Now login required at `http://<vm_ip>:1880`.

## Docker Volume Configuration

### Add SSH Keys Volume

To enable SSH/SCP functionality from Node-RED to the Raspberry Pi, you need to mount the SSH directory as a volume in the Node-RED container.

#### Option 1: Using Portainer (Recommended for Beginners)

1. **Access Portainer**:
   - Navigate to `http://<vm_ip>:9000`
   - Login with your credentials

2. **Navigate to Node-RED Container**:
   - Click **Containers** in the left menu
   - Find `nodered` in the list
   - Click **Duplicate/Edit**

3. **Stop the Container**:
   - Click **Stop** on the container
   - Wait for it to fully stop

4. **Add Volume Mapping**:
   - In the container details, scroll to **Volumes** section
   - Click **+ map additional volume**
   - **Container path**: `/root/.ssh`
   - **Host path**: `/home/<user>/.ssh` (replace `<user>` with your username, e.g., `/home/kieran/.ssh`)
   - Toggle **Read-only** to ON (recommended for security)
   - Click **Deploy the container** at the bottom

5. **Start the Container**:
   - The container should start automatically after deployment
   - Verify it's running with a green "running" status

!!! tip "Alternative: Using Stacks"
    You can also use the **Stacks** feature in Portainer to edit the entire docker-compose.yml and redeploy the stack.

#### Option 2: Using Command Line

Edit the `docker-compose.yml` file in `~/IOTstack`:

```bash
cd ~/IOTstack
nano docker-compose.yml
```

Find the `nodered` service section and add the SSH volume to the volumes list:

```yaml
nodered:
  container_name: nodered
  image: nodered/node-red:latest
  restart: unless-stopped
  ports:
    - "1880:1880"
  volumes:
    - ./volumes/nodered/data:/data
    - /home/<user>/.ssh:/root/.ssh:ro  # Add this line
  environment:
    - TZ=Europe/Amsterdam
```

Replace `<user>` with your actual username (e.g., `/home/kieran/.ssh:/root/.ssh:ro`).

!!! note "Read-Only Mount"
    The `:ro` flag mounts the volume as read-only for security. This allows Node-RED to read SSH keys but not modify them.

After making changes, restart Node-RED:

```bash
docker compose down nodered
docker compose up -d nodered
```

#### Verify Volume Mount

To confirm the SSH volume is properly mounted, check the container configuration:

**Via Portainer**:
- Go to **Containers** → **nodered** → **Inspect**
- Scroll to **Mounts** section
- Verify `/root/.ssh` is listed with source `/home/<user>/.ssh`

**Via Command Line**:
```bash
docker inspect nodered | grep -A 5 Mounts
```

You should see the SSH directory mounted.

## Install Required Nodes

### Via UI

1. Menu (☰) → **Manage palette**
2. **Install** tab
3. Search and install:
   - `node-red-contrib-influxdb`
   - (Any other custom nodes you need)

### Via Command Line

```bash
docker exec -it nodered npm install --prefix /usr/src/node-red node-red-contrib-influxdb
docker compose restart nodered
```

## Prerequisites Before Configuring Flow

Before importing and configuring your flow, ensure:

1. **InfluxDB** is running with database `iot_data` created - see [InfluxDB Configuration](influxdb.md)
2. **Grafana** is configured with dashboard and service account token - see [Grafana Configuration](grafana.md)
3. **Tailscale VPN** is configured and running between VM and Raspberry Pi - see [Tailscale Configuration](#tailscale-configuration) below
4. **Raspberry Pi** has SSH key-based authentication configured - see [Raspberry Pi SSH Setup](../raspberry-pi/ssh-setup.md)
5. **SSH known_hosts** is configured - see [SSH Known Hosts Setup](#ssh-known-hosts-setup) below

### SSH Known Hosts Setup

Before Node-RED can connect to the Raspberry Pi via SSH/SCP, you need to add the Pi to the known hosts file. This prevents SSH from prompting for host verification.

**On the VM, connect to the Pi via SSH as your regular user:**

```bash
ssh <pi-user>@<pi-tailscale-ip>
```

Example:
```bash
ssh hrtech@100.64.1.5
```

When prompted with "Are you sure you want to continue connecting (yes/no)?", type `yes` and press Enter. This adds the Pi to `~/.ssh/known_hosts`.

You can then exit the SSH session:

```bash
exit
```

!!! important "Required Before Node-RED Configuration"
    This step must be completed before Node-RED can use SSH/SCP commands to the Raspberry Pi. The known_hosts file is shared with the Node-RED container through the volume mount.

### Tailscale Configuration

Tailscale provides secure networking between the VM and Raspberry Pi without requiring manual port forwarding or complex VPN setup.

#### Install Tailscale on VM

Run Tailscale natively on the VM (simpler than Docker-based setup):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the authentication link to connect your VM to your Tailscale network.

**Get VM's Tailscale IP:**
```bash
tailscale ip -4
```

Note this IP address (typically in the `100.x.x.x` range).

#### Install Tailscale on Raspberry Pi

!!! warning "Prerequisite: Configure Raspberry Pi First"
    Before proceeding, ensure Tailscale is installed and configured on your Raspberry Pi. 
    
    See [Raspberry Pi SSH Setup](../raspberry-pi/ssh-setup.md) for complete Pi network configuration including Tailscale installation.

After Tailscale is installed on the Pi, get its Tailscale IP:

```bash
tailscale ip -4
```

Note this IP address - you'll use it in Node-RED.

#### Verify Connectivity

**From VM, ping the Raspberry Pi:**
```bash
ping <pi-tailscale-ip>
```

**Test SSH connection:**
```bash
ssh <pi-user>@<pi-tailscale-ip>
```

#### Configure Node-RED for Tailscale

Update the exec node's SCP command to use the Raspberry Pi's Tailscale IP:

```bash
scp /tmp/001.png <pi-user>@<pi-tailscale-ip>:/home/<pi-user>/incoming_images/incoming.png
```

Example:
```bash
scp /tmp/001.png hrtech@100.64.1.5:/home/hrtech/incoming_images/incoming.png
```

!!! success "Advantages of Tailscale"
    - No manual port forwarding required
    - Automatic NAT traversal
    - Easy multi-device management
    - Built-in access controls
    - Works across different networks
    - Zero-config mesh networking

## Import Flow

### Option 1: Use Provided Flow

This repository includes a pre-configured flow with all nodes already set up for the wind turbine monitoring system.

**Download**: [flows.json](../files/flows.json) (right-click → Save link as...)

1. In Node-RED UI, Menu → **Import**
2. **Select a file to import**
3. Choose the downloaded `flows.json` file
4. Click **Import**

### Option 2: Copy Directly

If you have access to the repository on the VM, copy the flow file directly:

```bash
cp /path/to/repository/docs/files/flows.json ~/IOTstack/volumes/nodered/data/flows.json
docker compose restart nodered
```

!!! note "Configuration Still Required"
    After importing, you'll still need to configure the nodes with your specific values (IPs, tokens, etc.) as described in the Flow Configuration section below.

## Flow Configuration

### Configure InfluxDB Nodes

For each InfluxDB node in the flow:

1. Double-click the node
2. Edit **Server** configuration:
   - **URL**: `http://influxdb:8086` (Docker DNS name)
   - **Database**: `iot_data`
3. Click **Done**
4. Deploy

> See [InfluxDB Configuration](influxdb.md) for database setup details.

### Configure HTTP Request Nodes

#### Weather API Node

- **Method**: GET
- **URL**: `https://weerlive.nl/api/weerlive_api_v2.php?key=3c2224ecab&locatie=51.89908,4.42348`
- **Return**: a parsed JSON object

#### eGauge Node

- **Method**: GET
- **URL**: `https://egauge84432.egaug.es/cgi-bin/egauge?inst`
- **Return**: a UTF-8 string
- **Authentication**: Digest Auth
  - **Username**: `HR_RDM`
  - **Password**: (provided separately)

!!! note "Authentication Required"
    The eGauge API requires digest authentication for access. Ensure you have the correct credentials before configuring this node.

!!! info "API Details"
    The eGauge XML API provides real-time instantaneous register values. See the eGauge XML API documentation for register types and data structures.

**Configuration Steps:**

1. Double-click the eGauge HTTP request node
2. Check **Use authentication**
3. **Type**: Digest authentication
4. Enter **Username**: `HR_RDM`
5. Enter **Password**: (obtain from supervisor/administrator)
6. Click **Done**

#### Grafana Render Node

- **Method**: GET
- **URL**: `http://<vm_ip>:3000/render/d/<dashboard_uid>?orgId=1&from=now-1h&to=now&timezone=browser&height=1440&width=2560&scale=1&kiosk=true&hideNav=true&fullPageImage=true`
- **Return**: a binary buffer
- **Authentication**: Bearer
- **Token**: Your Grafana service account token (starts with `glsa_`)

Replace:
- `<vm_ip>` with your VM's IP address
- `<dashboard_uid>` with your dashboard UID from Grafana

> See [Grafana Configuration - Generate Service Account Token](grafana.md#generate-service-account-token) and [Get Dashboard UID](grafana.md#get-dashboard-uid) for details.

### Configure Exec Nodes (SSH to Raspberry Pi)

The exec node for SCP uses Tailscale for secure connectivity:

- **Command**: `scp /tmp/001.png <pi-user>@<pi-tailscale-ip>:/home/<pi-user>/incoming_images/incoming.png`
  - Replace `<pi-tailscale-ip>` with your Pi's Tailscale IP (e.g., `100.64.1.5`)
  - Replace `<pi-user>` with your Pi username (e.g., `hrtech`)
  - All communication goes through Tailscale's encrypted mesh network

**Example:**
```bash
scp /tmp/001.png hrtech@100.64.1.5:/home/hrtech/incoming_images/incoming.png
```

**Settings:**
- **Append msg.payload**: unchecked
- **Use spawn**: unchecked

**Prerequisites:**

- Tailscale configured and active (see [Tailscale Configuration](#tailscale-configuration))
- SSH key-based authentication configured (see [Raspberry Pi SSH Setup](../raspberry-pi/ssh-setup.md))
- SSH known_hosts configured (see [SSH Known Hosts Setup](#ssh-known-hosts-setup))
- SSH volume mounted in Docker (see [Docker Volume Configuration](#docker-volume-configuration))

**Security Note**: By using Tailscale's private network IPs, all SSH/SCP traffic is encrypted through a secure tunnel. SSH port 22 is never exposed to the public internet, eliminating a major security risk.

### Configure Inject Nodes (Timing)

#### loop_turbine_data

- **Repeat**: interval
- **Every**: 30 seconds
- **Start automatically**: checked

#### loop_weather_data

- **Repeat**: interval
- **Every**: 300 seconds (5 minutes)
- **Start automatically**: checked

#### loop_image_renderer

- **Repeat**: interval
- **Every**: 300 seconds (5 minutes)
- **Start automatically**: checked
- **Inject once**: at start plus 0.1 seconds

## Function Node Code

The function nodes contain JavaScript for data processing. Key ones:

### parse_data (Weather)

Extracts weather data, calculates air density, formats for InfluxDB.

```javascript
// See flows.json for complete code
let data = msg.payload;
let weather = data.liveweer[0];
// ... processing ...
msg.payload = [{ measurement: "weather", tags: {...}, fields: {...} }];
return msg;
```

### parse_data (Turbine)

Parses XML from eGauge, sanitizes field names, formats for InfluxDB.

```javascript
// See flows.json for complete code
function sanitize(name) {
    // Prefix logic for S6_, IM_, VM_ etc.
}
// ... processing ...
msg.payload = [{ measurement: "turbine", tags: {...}, fields: {...} }];
return msg;
```

## Deploy Flow

After configuration, click **Deploy** (top right).

Monitor debug nodes:

- weather_data
- turbine_data
- Errors SCP
- Errors Refresh

## Flow Testing

### Test Individual Loops

Click the inject node's button to manually trigger:

1. **loop_turbine_data**: Should fetch and store turbine data
2. **loop_weather_data**: Should fetch and store weather data
3. **loop_image_renderer**: Should render image and send to Pi

### Monitor Debug Output

Open **Debug** panel (bug icon) to see:

- HTTP responses
- Parsed data
- InfluxDB write results
- Errors

## Backup Flow

### Export Flow

1. Select all nodes (Ctrl+A)
2. Menu → **Export**
3. **JSON** tab
4. Click **Download**

Save to secure location.

### Automatic Backup

Node-RED auto-saves flows to:

```
~/IOTstack/volumes/nodered/data/flows.json
```

Include in your regular backups.

## Useful Nodes

### Debug Node

Shows msg properties in debug panel:

- Set to `msg.payload` to see data
- Enable/disable with button

### Inject Node

Manually trigger flows:

- Click button to inject
- Configure repeat intervals

### Function Node

Write custom JavaScript:

- Access `msg`, `context`, `flow`, `global`
- Use `node.warn()` for logging

### Change Node

Modify message properties:

- Set, delete, move properties
- Use JSONata expressions

## Advanced Configuration

### Context Storage

Enable file-based context (persists across restarts):

Edit `settings.js`:

```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    },
},
```

### Custom Modules

Install npm packages:

```bash
docker exec -it nodered npm install --prefix /usr/src/node-red <package-name>
docker compose restart nodered
```

### Environment Variables

Pass variables to Node-RED via `docker-compose.yml`:

```yaml
nodered:
  environment:
    - TZ=Europe/Amsterdam
    - NODE_RED_CREDENTIAL_SECRET=your_secret_key
```

## Troubleshooting

### Flow Not Running

1. Check if Node-RED is running:

```bash
docker compose ps nodered
```

2. Check for errors in logs:

```bash
docker compose logs -f nodered
```

3. Verify flows are deployed (should see "Flows started")

### HTTP Request Timeouts

Increase timeout in HTTP request node:

- **Timeout**: 60 seconds (default is 120)

### InfluxDB Write Errors

Check:

1. InfluxDB is running
2. Database exists
3. Data format is correct (array of points)
4. Network connectivity

### SSH/SCP Fails

**Check Tailscale connectivity:**

1. **Tailscale is running**: `sudo tailscale status`
2. **Tailscale connectivity**: `ping <pi-tailscale-ip>`
3. **Pi is online in Tailscale**: Check `tailscale status` output shows the Pi

**Check SSH configuration:**

1. **SSH volume is mounted**: Check `docker-compose.yml` has `/home/<user>/.ssh:/root/.ssh:ro`
2. **Known_hosts is configured**: The Pi's host key should be in `~/.ssh/known_hosts`
3. **SSH keys have correct permissions**:
   ```bash
   ls -l ~/.ssh/
   # id_rsa should be 600, id_rsa.pub should be 644
   ```
4. **Test SSH manually from VM** (not from Node-RED container):
   ```bash
   ssh <pi-user>@<pi-tailscale-ip>
   ```
5. **Path exists on Pi**: `/home/<pi-user>/incoming_images/`
6. **SSH keys are configured** on the Pi (see [Raspberry Pi SSH Setup](../raspberry-pi/ssh-setup.md))

See:
- [Tailscale Configuration](#tailscale-configuration)
- [Raspberry Pi SSH Setup](../raspberry-pi/ssh-setup.md)
- [SSH Known Hosts Setup](#ssh-known-hosts-setup)

## Performance

### Limit Concurrent Requests

Use **Delay** nodes to throttle:

- **Action**: Rate Limit
- **All messages**: 1 per 5 seconds

### Monitor Memory

```bash
docker stats nodered
```

If high memory usage, reduce flow complexity or increase container limits.

## Security Best Practices

1. **Enable authentication** (adminAuth)
2. **Use HTTPS** (reverse proxy with nginx)
3. **Limit network exposure** (firewall rules)
4. **Secure credentials** (use environment variables)
5. **Regular updates**

```bash
docker compose pull nodered
docker compose up -d nodered
```
