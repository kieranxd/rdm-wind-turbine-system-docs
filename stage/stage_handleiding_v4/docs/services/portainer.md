# Portainer

Portainer provides a web UI for managing Docker containers, images, and stacks.

## Access

- URL: `http://<vm_ip>:9000`
  - Replace `<vm_ip>` with your VM's IP address (e.g., `192.168.178.164`)
- Create admin user on first visit

## Initial Setup

1. Navigate to `http://<vm_ip>:9000`
2. **Create admin user**:
   - Username: `admin`
   - Password: (set strong password)
3. Click **Create user**
4. **Connect to local Docker**:
   - Select "Get started" → "Local"
   - Portainer auto-connects to Docker socket

## Main Features

### Dashboard

Overview of:

- Running containers
- Stopped containers
- Images
- Volumes
- Networks

### Containers

View all containers with:

- Status (running, stopped, paused)
- CPU and memory usage
- Port mappings

**Actions:**

- Start/Stop/Restart containers
- View logs
- Inspect container details
- Attach console
- View stats

### Stacks

Manage Docker Compose stacks:

- View IOTstack
- Edit docker-compose.yml
- Restart entire stack
- View stack containers

### Images

- View all Docker images
- Pull new images
- Remove unused images
- Build images from Dockerfile

### Volumes

- View persistent data volumes
- Browse volume contents
- Remove unused volumes

### Networks

- View Docker networks
- Create custom networks
- Inspect network details

## Common Tasks

### View Container Logs

1. Go to **Containers**
2. Click container name
3. **Logs** tab
4. Adjust:
   - Lines to display
   - Auto-refresh
   - Wrap lines

### Restart a Service

1. Go to **Containers**
2. Select container checkbox
3. Click **Restart**

Or:

1. Click container name
2. Click **Restart** button at top

### Update Container Image

1. Go to **Containers**
2. Click container name
3. **Image** section shows current tag
4. Go to **Images** → find image → **Pull latest**
5. Return to **Containers** → **Recreate** container

### Execute Commands in Container

1. Go to **Containers**
2. Click container name
3. **Console** tab
4. Choose shell (usually `/bin/bash` or `/bin/sh`)
5. Click **Connect**

### Monitor Resource Usage

1. Go to **Containers**
2. Click container name
3. **Stats** tab
4. View:
   - CPU usage %
   - Memory usage
   - Network I/O
   - Block I/O

## Stack Management

### View IOTstack

1. Go to **Stacks**
2. Click `iotstack` (or your stack name)
3. View:
   - Containers in stack
   - Stack file (docker-compose.yml)

### Edit Stack

1. **Stacks** → Select stack
2. **Editor** tab
3. Modify `docker-compose.yml`
4. Click **Update the stack**
5. Containers will be recreated with new config

**Warning:** Use with caution, prefer editing files directly via SSH.

### Restart Stack

1. **Stacks** → Select stack
2. Click **Stop this stack**
3. Wait for all containers to stop
4. Click **Start this stack**

## Security

### Change Admin Password

1. **Users** → Select admin user
2. **Change password**
3. Enter current and new password
4. Save

### Add Users

1. Go to **Users**
2. Click **Add user**
3. Set username and password
4. Assign role (admin or user)
5. Create

### Enable Two-Factor Authentication

Available in Portainer Business Edition (paid).

## Backup Portainer Data

Portainer data is stored in a Docker volume:

```bash
docker volume ls | grep portainer
```

Backup:

```bash
docker run --rm -v portainer_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/portainer_backup.tar.gz /data
```

Restore:

```bash
docker run --rm -v portainer_data:/data -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/portainer_backup.tar.gz --strip 1"
```

## Useful Features

### Image Cleanup

Remove unused images to free space:

1. **Images** → **Unused**
2. Select all unused images
3. Click **Remove**

### Volume Cleanup

Remove unused volumes:

1. **Volumes** → **Unused**
2. Select volumes
3. **Remove**

### Quick Actions

From dashboard:

- Click numbers to jump to containers/images/volumes
- Use search to find specific containers
- Filter by status

## Troubleshooting

### Can't Access Portainer

Check if running:

```bash
docker compose ps portainer
```

Restart:

```bash
docker compose restart portainer
```

### Forgot Admin Password

Reset by recreating Portainer:

```bash
cd ~/IOTstack
docker compose down portainer
docker volume rm portainer_data
docker compose up -d portainer
```

**Warning:** This deletes all Portainer settings. You'll need to reconfigure.

### Portainer Shows Wrong Status

Refresh browser or clear cache:

- Ctrl+Shift+R (hard refresh)
- Or clear browser cache

## Limitations

- Cannot replace direct command-line access
- Some advanced Docker features not available in UI
- Performance overhead (minimal)

## Alternatives

If Portainer doesn't meet your needs:

- **Yacht**: Lightweight alternative
- **Dockge**: Stack-focused manager
- **Lazydocker**: Terminal UI for Docker

## Updates

Portainer included in IOTstack. Update with:

```bash
cd ~/IOTstack
docker compose pull portainer
docker compose up -d portainer
```

Check [Portainer releases](https://github.com/portainer/portainer/releases) for changes.
