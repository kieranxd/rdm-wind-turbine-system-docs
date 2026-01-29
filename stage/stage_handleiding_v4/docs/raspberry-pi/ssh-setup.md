# SSH Configuration

Configure SSH for Node-RED to trigger scripts on the Raspberry Pi.

## Goal

Allow Node-RED on the VM to execute commands on the Raspberry Pi without password prompts.

## Prerequisites

### Install Tailscale on Raspberry Pi

Tailscale provides secure networking between the VM and Raspberry Pi without requiring manual port forwarding.

**Install Tailscale:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the authentication link in your browser to connect your Pi to your Tailscale network.

**Get Pi's Tailscale IP:**

```bash
tailscale ip -4
```

Note this IP address (typically in the `100.x.x.x` range) - you'll use it for SSH connections and in Node-RED.

!!! note "VM Tailscale Setup Required"
    Ensure Tailscale is also installed on your VM. See [Node-RED Configuration - Tailscale Configuration](../services/node-red.md#tailscale-configuration) for VM setup.

## On the VM (Node-RED Host)

### Generate SSH Key

As the user running Node-RED (or Docker user):

```bash
ssh-keygen -t ed25519 -C "node-red@iotstack"
```

Press Enter for all prompts (no passphrase for automation).

Keys created:

- Private: `~/.ssh/id_ed25519`
- Public: `~/.ssh/id_ed25519.pub`

### Copy Public Key to Raspberry Pi

```bash
ssh-copy-id <pi_username>@<pi_tailscale_ip>
```

Replace:
- `<pi_username>` with your Raspberry Pi username (e.g., `hrtech`, `pi`)
- `<pi_tailscale_ip>` with your Raspberry Pi's Tailscale IP address (e.g., `100.64.1.5`)

Enter Raspberry Pi password when prompted.

!!! tip "Use Tailscale IP Address"
    Use the Pi's Tailscale IP address, not its local network IP. Get this with `tailscale ip -4` on the Pi.

Alternatively, manual copy:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the output, then on Raspberry Pi:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
# Paste the public key
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Test SSH Connection

```bash
ssh <pi_username>@<pi_tailscale_ip> 'echo Connected successfully'
```

Should connect without password using the Tailscale IP.

## On the Raspberry Pi

### Disable Password Authentication (Recommended)

For security, allow only key-based auth:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set:

```
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

### Configure Sudo for Scripts

Allow running specific commands without password:

```bash
sudo visudo
```

Add these lines (replace `<pi_username>` with your actual Raspberry Pi username):

```
<pi_username> ALL=(ALL) NOPASSWD: /usr/bin/tee
<pi_username> ALL=(ALL) NOPASSWD: /usr/bin/mount
<pi_username> ALL=(ALL) NOPASSWD: /usr/bin/umount
<pi_username> ALL=(ALL) NOPASSWD: /usr/bin/mkdir
<pi_username> ALL=(ALL) NOPASSWD: /bin/rm
<pi_username> ALL=(ALL) NOPASSWD: /bin/cp
```

This allows passwordless sudo for the scripts.

### Test Sudo Commands

```bash
ssh <pi_username>@<pi_tailscale_ip> 'sudo mount -o loop /home/<pi_username>/drive.bin /mnt'
```

Should not prompt for password.

## Node-RED Configuration

### For Docker Node-RED

If Node-RED runs in Docker, the SSH key must be accessible inside the container.

#### Option 1: Mount SSH Directory

Edit `docker-compose.yml`:

```yaml
services:
  nodered:
    volumes:
      - ./volumes/nodered/data:/data
      - /home/<your-user>/.ssh:/root/.ssh:ro  # Add this line
```

Restart Node-RED:

```bash
docker compose restart nodered
```

#### Option 2: Copy Key into Container

```bash
docker cp ~/.ssh/id_ed25519 nodered:/root/.ssh/
docker cp ~/.ssh/id_ed25519.pub nodered:/root/.ssh/
docker exec nodered chmod 600 /root/.ssh/id_ed25519
```

### Test from Node-RED Container

```bash
docker exec nodered ssh <pi_username>@<pi_tailscale_ip> 'echo Test from container'
```

### SSH Known Hosts

First connection may prompt to accept host key:

```bash
docker exec -it nodered ssh <pi_username>@<pi_tailscale_ip>
```

Type `yes` to accept.

Or add to known_hosts automatically:

```bash
docker exec nodered ssh-keyscan -H <pi_tailscale_ip> >> /root/.ssh/known_hosts
```

## Node-RED Exec Node Configuration

### SCP Command (Copy Image)

**Command:**

```bash
scp /tmp/001.png <pi_username>@<pi_tailscale_ip>:/home/<pi_username>/incoming_images/incoming.png
```

Example:
```bash
scp /tmp/001.png hrtech@100.64.1.5:/home/hrtech/incoming_images/incoming.png
```

**Options:**

- **Append msg.payload**: Unchecked
- **Use spawn**: Unchecked
- **Timeout**: 60 seconds

!!! tip "Use Tailscale IP Address"
    Use the Raspberry Pi's Tailscale IP address (100.x.x.x range), not its local network IP.

### SSH Command (Trigger Script)

Add an exec node after SCP:

**Command:**

```bash
ssh <pi_username>@<pi_tailscale_ip> '/home/<pi_username>/refresh_script.sh'
```

Example:
```bash
ssh hrtech@100.64.1.5 '/home/hrtech/refresh_script.sh'
```

**Options:**

- **Append msg.payload**: Unchecked
- **Use spawn**: Unchecked
- **Timeout**: 120 seconds (script may take time)

### Error Handling

Connect the error output (pin 2) of exec nodes to debug nodes to catch failures.

## Troubleshooting

### Permission Denied (publickey)

Check:

1. Public key copied to Pi: `cat ~/.ssh/authorized_keys` on Pi
2. Private key accessible in Node-RED container
3. Permissions correct:
   - Private key: `chmod 600`
   - `~/.ssh` directory: `chmod 700`

### Host Key Verification Failed

Add Pi to known_hosts:

```bash
ssh-keyscan -H <pi_tailscale_ip> >> ~/.ssh/known_hosts
```

Inside container:

```bash
docker exec nodered ssh-keyscan -H <pi_tailscale_ip> >> /root/.ssh/known_hosts
```

### Connection Timeout

Check:

1. Tailscale is connected on both VM and Pi:
   - VM: `docker exec tailscale tailscale status` (if Docker) or `tailscale status` (if native)
   - Pi: `sudo tailscale status`
2. Pi is reachable via Tailscale: `ping <pi_tailscale_ip>`
3. SSH service running on Pi: `sudo systemctl status ssh`
4. Pi's Tailscale IP hasn't changed (rare, but check with `tailscale ip -4` on Pi)

### SCP Fails

Verify:

1. Source file exists: `ls /tmp/001.png`
2. Destination directory exists on Pi: `ls /home/<pi_username>/incoming_images/`
3. Permissions allow write to destination

Create directory if missing:

```bash
ssh <pi_username>@<rpi_ip> 'mkdir -p /home/<pi_username>/incoming_images'
```

## Security Best Practices

### Use SSH Keys Only

Never use password authentication for automation.

### Limit Sudo Commands

Only grant passwordless sudo to specific commands needed by scripts.

### Firewall Rules

On the Pi, you can optionally restrict SSH access to Tailscale network only:

```bash
# Allow SSH only from Tailscale interface
sudo ufw allow in on tailscale0 to any port 22
sudo ufw deny 22
sudo ufw enable
```

This ensures SSH is only accessible through Tailscale, not from the local network.

### Regular Key Rotation

Periodically regenerate SSH keys:

1. Generate new key
2. Copy to Pi
3. Test
4. Remove old key from `authorized_keys`

### Monitor SSH Logs

On Pi, check for suspicious activity:

```bash
sudo tail -f /var/log/auth.log
```

## Testing Full Flow

### From VM Terminal

```bash
# Copy image
scp /tmp/001.png <pi_username>@<pi_tailscale_ip>:/home/<pi_username>/incoming_images/incoming.png

# Trigger script
ssh <pi_username>@<pi_tailscale_ip> '/home/<pi_username>/refresh_script.sh'
```

Example:
```bash
scp /tmp/001.png hrtech@100.64.1.5:/home/hrtech/incoming_images/incoming.png
ssh hrtech@100.64.1.5 '/home/hrtech/refresh_script.sh'
```

Both should complete without password prompts.

### From Node-RED

1. Manually inject the "loop_image_renderer" flow
2. Check debug output for:
   - Successful SCP
   - Successful SSH script execution
3. Verify image appears on display

## Backup SSH Keys

Save SSH keys securely:

```bash
# On VM
cp ~/.ssh/id_ed25519 /path/to/secure/backup/
cp ~/.ssh/id_ed25519.pub /path/to/secure/backup/
```

Store in password manager or encrypted backup.

## Alternative: SCP Without SSH

If SSH setup is problematic, alternatives:

1. **HTTP Upload**: Run web server on Pi, Node-RED POSTs image
2. **Shared Network Storage**: NFS or SAMBA mount
3. **MQTT**: Publish image as base64 message

However, SSH/SCP is the most straightforward for this use case.
