# Grafana Configuration

Grafana provides the visualization dashboard for turbine and weather data.

## Initial Access

- URL: `http://<vm_ip>:3000`
  - Replace `<vm_ip>` with your VM's IP address (e.g., `192.168.178.164`)
- Default credentials: `admin` / `admin`
- Change password on first login

## Add InfluxDB Data Source

1. Go to **Connections** → **Data sources**
2. Click **Add new data source**
3. Select **InfluxDB**
4. Configure:
   - **Name**: `InfluxDB-IoT`
   - **Query Language**: `InfluxQL` (for InfluxDB 1.x)
   - **URL**: `http://influxdb:8086` (Docker DNS name)
   - **Database**: `iot_data`
   - Leave authentication fields blank if no auth is enabled
5. Click **Save & Test**

Should show: "Data source is working".

For detailed InfluxDB setup and configuration, see [InfluxDB Configuration](influxdb.md).

## Image Renderer Integration

For dashboard rendering to images (needed for e-ink display), you need the Grafana Image Renderer service.

### Prerequisites

1. First, deploy the Grafana Image Renderer service - see [Grafana Image Renderer](grafana-renderer.md) documentation
2. Generate a service account token (see below)
3. Configure Grafana with renderer settings

### Generate Service Account Token

1. In Grafana UI, go to **Administration** → **Users and access** → **Service accounts**
2. Click **Add service account**
3. Name: `image-renderer`
4. Role: `Viewer`
5. Click **Create**
6. Click **Add service account token**
7. Copy the token (starts with `glsa_`)
8. Save this token - you'll need it for both Grafana and the renderer service

### Configure Grafana Environment Variables

Add these environment variables to your Grafana container:

```yaml
services:
  grafana:
    environment:
      - GF_SERVER_DOMAIN=<vm_ip>
      - GF_SERVER_ROOT_URL=http://<vm_ip>:3000/
      - GF_RENDERING_SERVER_URL=http://grafana-image-renderer:8081/render
      - GF_RENDERING_CALLBACK_URL=http://<vm_ip>:3000/
      - GF_RENDERING_RENDERER_TOKEN=<your_service_account_token>
```

!!! note "Docker Network Configuration"
    Use Docker DNS name `grafana-image-renderer` if both services are in the same Docker network. Otherwise use `http://<vm_ip>:8081/render`.

Replace:
- `<vm_ip>` with your VM's IP address
- `<your_service_account_token>` with the token from above

Restart Grafana after changes:

```bash
docker compose restart grafana
```

## Create Dashboard

### Option 1: Import Pre-configured Dashboard

This repository includes a pre-configured dashboard designed for the wind turbine monitoring system with:

- Environmental impact metrics (CO₂ avoided, households powered, trees equivalent)
- Power generation gauges and time series
- Weather conditions (wind speed, direction, temperature, humidity)
- All panels pre-configured with InfluxQL queries

**Download**: [dashboard.json](../files/dashboard.json) (right-click → Save link as...)

To import:

1. Go to **Dashboards** → **New** → **Import**
2. Upload the downloaded `dashboard.json` file
3. Click **Load**
4. Select data source: **InfluxDB-IoT**
5. Click **Import**

!!! warning "Verify Data Source UID"
    After importing, verify that the InfluxDB data source UID matches. If panels show "No data", edit each panel and reselect the data source.

### Option 2: Import Other Dashboard

If you have a different dashboard JSON:

1. Go to **Dashboards** → **New** → **Import**
2. Upload JSON file or paste JSON
3. Click **Load**
4. Select data source
5. Click **Import**

### Option 3: Create New Dashboard

1. Go to **Dashboards** → **New** → **New Dashboard**
2. Click **Add visualization**
3. Select **InfluxDB-IoT** data source
4. Write query:

```sql
SELECT mean("temperature_C") FROM "weather" 
WHERE $timeFilter 
GROUP BY time($__interval)
```

5. Customize panel (title, axes, legend)
6. Click **Apply**
7. Add more panels as needed
8. Click **Save dashboard** (disk icon)

### Example Panels

#### Turbine Power Output

```sql
SELECT mean("grid_s6_power") FROM "turbine" 
WHERE $timeFilter 
GROUP BY time($__interval)
```

#### Wind Speed

```sql
SELECT mean("wind_speed_ms") FROM "weather" 
WHERE $timeFilter 
GROUP BY time($__interval)
```

#### Temperature & Humidity

```sql
SELECT mean("temperature_C") AS "Temperature", 
       mean("humidity") AS "Humidity" 
FROM "weather" 
WHERE $timeFilter 
GROUP BY time($__interval)
```

## Get Dashboard UID

For Node-RED image requests, you need the dashboard UID:

1. Open the dashboard
2. Check the URL: `http://.../d/<UID>/dashboard-name`
3. Copy the UID (e.g., `ad5v9hp`)

Or via API:

```bash
curl -s -H "Authorization: Bearer <your_service_account_token>" \
  http://<vm_ip>:3000/api/search | jq
```

## Image Rendering Test

Test the renderer directly:

```bash
curl -H "Authorization: Bearer <your_service_account_token>" \
  "http://<vm_ip>:3000/render/d/<dashboard_uid>?orgId=1&from=now-3h&to=now&width=1536&height=-1&kiosk=true" \
  --output test_render.png
```

Replace:
- `<your_service_account_token>` with the token generated in the previous section
- `<dashboard_uid>` with your dashboard UID (e.g., `ad5v9hp`)

If successful, `test_render.png` should contain the dashboard image.

## Additional Configuration

### Anonymous Access (Optional)

To allow viewing without login:

```yaml
environment:
  - GF_AUTH_ANONYMOUS_ENABLED=true
  - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
```

Restart Grafana after changes.

### Email Alerts (Optional)

Configure SMTP for alerts:

```yaml
environment:
  - GF_SMTP_ENABLED=true
  - GF_SMTP_HOST=smtp.gmail.com:587
  - GF_SMTP_USER=your-email@gmail.com
  - GF_SMTP_PASSWORD=your-app-password
  - GF_SMTP_FROM_ADDRESS=your-email@gmail.com
```

## Backup Dashboard

### Export Dashboard JSON

1. Open dashboard
2. Click **Export**
3. Click **Export as code**
4. Click **Download file**

Store in safe location.

### Programmatic Backup

```bash
# Get dashboard JSON by UID
curl -H "Authorization: Bearer <your_service_account_token>" \
  http://<vm_ip>:3000/api/dashboards/uid/<dashboard_uid> > dashboard_backup.json
```

Replace `<dashboard_uid>` with your dashboard UID (e.g., `ad5v9hp`).

## Restore Dashboard

1. **Dashboards** → **New** → **Import**
2. Upload saved JSON
3. Select data source
4. Import

## Useful Links

- Grafana Docs: [https://grafana.com/docs/](https://grafana.com/docs/)
- Query Language: [InfluxQL Guide](https://docs.influxdata.com/influxdb/v1.8/query_language/)

## Troubleshooting

### Can't Connect to InfluxDB

Check Docker network:

```bash
docker exec grafana ping influxdb
```

Should resolve to InfluxDB container IP.

### Image Rendering Not Working

Check renderer service:

```bash
docker compose ps grafana-image-renderer
docker compose logs grafana-image-renderer
```

Verify token matches in both services.

### Dashboard Shows No Data

1. Verify data exists in InfluxDB
2. Check time range in dashboard
3. Review query syntax
4. Check data source configuration
