# Grafana Image Renderer

The Grafana Image Renderer is a separate service that runs a headless Chromium browser to render dashboard screenshots.

## Why Separate Service?

- Not included in IOTstack by default
- Requires more resources (runs Chromium)
- Needs specific configuration for authentication
- Can be scaled independently

## Prerequisites

Before deploying the renderer:

1. Have Grafana already running (see [Grafana Configuration](grafana.md))
2. Generate a service account token in Grafana:
   - Go to **Administration** → **Users and access** → **Service accounts**
   - Create account named `image-renderer` with `Viewer` role
   - Generate and save the token (starts with `glsa_`)
   - See [Grafana Configuration](grafana.md#generate-service-account-token) for detailed steps
3. Note your VM's IP address

## Deploy the Renderer

You can add the Grafana Image Renderer service either via docker-compose.yml or Portainer.

### Option 1: Via docker-compose.yml

Edit `~/IOTstack/docker-compose.yml` and add this service:

```yaml
services:
  # ... existing services ...

  grafana-image-renderer:
    image: grafana/grafana-image-renderer:latest
    container_name: grafana-image-renderer
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      - ENABLE_METRICS=true
      - AUTH_TOKEN=<your_service_account_token>
      - HTTP_HOST=0.0.0.0
      - HTTP_PORT=8081
      - BROWSER_TZ=Europe/Amsterdam  # Your timezone
    networks:
      - default
```

Replace `<your_service_account_token>` with the token from Prerequisites step 2.

**Important**: The `AUTH_TOKEN` must match the `GF_RENDERING_RENDERER_TOKEN` in Grafana (configured in [Grafana Configuration](grafana.md#configure-grafana-environment-variables)).

Then start the service:

```bash
cd ~/IOTstack
docker compose up -d grafana-image-renderer
```

### Option 2: Via Portainer

1. Navigate to `http://<vm_ip>:9000`
2. Go to **Containers**
3. Click **+ Add container**
4. Configure:
   - **Name**: `grafana-image-renderer`
   - **Image**: `grafana/grafana-image-renderer:latest`
   - **Manual network port publishing**:
     - Host: `8081`, Container: `8081`
   - **Restart policy**: `Unless stopped`
5. Scroll to **Env** section
6. Click **+ add environment variable** for each:
   - `ENABLE_METRICS` = `true`
   - `AUTH_TOKEN` = `<your_service_account_token>`
   - `HTTP_HOST` = `0.0.0.0`
   - `HTTP_PORT` = `8081`
   - `BROWSER_TZ` = `Europe/Amsterdam` (or your timezone)
7. Scroll to **Network** section
8. Select the same network as your Grafana container (usually `iotstack_default` or similar)
9. Click **Deploy the container**

!!! tip "Service Account Token"
    Use the token from Prerequisites step 2

## Verify Installation

```bash
cd ~/IOTstack
docker compose up -d grafana-image-renderer
```

Verify:

```bash
docker compose ps grafana-image-renderer
docker compose logs grafana-image-renderer
```

## Configuration Options

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| AUTH_TOKEN | Bearer token for authentication | (none) |
| HTTP_HOST | Listen address | 0.0.0.0 |
| HTTP_PORT | Listen port | 8081 |
| BROWSER_TZ | Timezone for rendering | UTC |
| ENABLE_METRICS | Enable Prometheus metrics | false |
| LOG_LEVEL | Logging level (debug, info, warn, error) | info |

### Rendering Options

Additional environment variables for fine-tuning:

```yaml
environment:
  - RENDERING_MODE=default  # or 'clustered' for high load
  - RENDERING_CLUSTERING_MODE=browser  # or 'context'
  - RENDERING_CLUSTERING_MAX_CONCURRENCY=5
  - RENDERING_VIEWPORT_MAX_WIDTH=3000
  - RENDERING_VIEWPORT_MAX_HEIGHT=3000
```

## Test the Renderer

### Direct Health Check

```bash
curl http://<vm_ip>:8081/
```

Should return:

```
Grafana Image Renderer
```

### Test Render with Token

```bash
curl -H "Authorization: Bearer <your_service_account_token>" \
  -d 'url=http://<vm_ip>:3000/d/<dashboard_uid>?orgId=1&from=now-3h&to=now&kiosk=true' \
  -d 'width=1536' \
  -d 'height=2048' \
  http://<vm_ip>:8081/render \
  --output test.png
```

Replace:
- `<your_service_account_token>` with your Grafana service account token
- `<vm_ip>` with your VM's IP address
- `<dashboard_uid>` with your dashboard UID (e.g., `ad5v9hp`)

Check `test.png` is a valid image.

## Configure Grafana Integration

After deploying the renderer, configure Grafana to use it. See [Grafana Configuration - Configure Grafana Environment Variables](grafana.md#configure-grafana-environment-variables) for detailed steps.

Key points:
- Use `GF_RENDERING_SERVER_URL=http://grafana-image-renderer:8081/render`
- Token must match the `AUTH_TOKEN` set above
- Restart Grafana after configuration

## Verify Integration

In Grafana:

1. Go to **Administration** → **General** → **Settings**
2. Find the "rendering" section
3. Ignore the legacy "plugin.grafana-image-renderer"
4. Verify the external renderer is configured correctly

Or check Grafana logs:

```bash
docker compose logs grafana | grep -i render
```

Should show successful connection to renderer.

## Performance Tuning

### Increase Memory Limit

For larger dashboards:

```yaml
grafana-image-renderer:
  mem_limit: 1g
  mem_reservation: 512m
```

### Adjust Concurrency

If rendering multiple images simultaneously:

```yaml
environment:
  - RENDERING_CLUSTERING_MODE=browser
  - RENDERING_CLUSTERING_MAX_CONCURRENCY=10
```

## Troubleshooting

### Renderer Not Responding

Check if service is running:

```bash
docker compose ps grafana-image-renderer
```

Check logs for errors:

```bash
docker compose logs -f grafana-image-renderer
```

Common issues:

- Out of memory: Increase `mem_limit`
- Port conflict: Change `8081` to another port
- Network issues: Verify Docker network connectivity

### Authentication Errors

If you see "unauthorized" errors:

1. Verify token matches in both services
2. Restart both Grafana and renderer:

```bash
docker compose restart grafana grafana-image-renderer
```

### Timeout Errors

Increase timeout in Grafana:

```yaml
grafana:
  environment:
    - GF_RENDERING_TIMEOUT=60  # seconds
```

### Image Quality Issues

Adjust rendering parameters:

```yaml
environment:
  - RENDERING_VIEWPORT_DEVICE_SCALE_FACTOR=2  # Higher DPI
  - IGNORE_HTTPS_ERRORS=true  # If using self-signed certs
```

## Monitoring

### Check Metrics

If `ENABLE_METRICS=true`:

```bash
curl http://<vm_ip>:8081/metrics
```

Returns Prometheus-format metrics.

### View Logs

```bash
docker compose logs -f grafana-image-renderer
```

Look for:

- Render requests
- Errors
- Performance metrics

## Security Notes

### Token Security

The bearer token should be:

- Unique and complex
- Same in both Grafana and renderer
- Kept secret (not in version control)

To generate a new token, use Grafana's Service Account feature (see [Grafana Configuration](grafana.md)).

### Network Isolation

For production, consider:

- Not exposing port 8081 externally
- Using Docker internal network only
- Firewall rules to restrict access

## Resource Usage

Typical resource usage:

- **CPU**: Low when idle, spikes during rendering
- **Memory**: 200-500 MB baseline, up to 1 GB during rendering
- **Disk**: Minimal (stores temporary files)

Monitor with:

```bash
docker stats grafana-image-renderer
```

## Updates

Update to latest version:

```bash
cd ~/IOTstack
docker compose pull grafana-image-renderer
docker compose up -d grafana-image-renderer
```

Check for breaking changes in [release notes](https://github.com/grafana/grafana-image-renderer/releases).
