# InfluxDB Configuration

InfluxDB 1.x is used as the time-series database for storing weather and turbine sensor data.

## Initial Setup

### Access InfluxDB CLI

```bash
docker exec -it influxdb influx
```

### Create Database

```sql
CREATE DATABASE iot_data
USE iot_data
SHOW MEASUREMENTS
EXIT
```

### Verify in Container Logs

```bash
docker compose logs influxdb
```

## Database Structure

### Measurements

Two main measurements are used:

#### 1. weather

Stores weather station data:

```sql
SELECT * FROM weather ORDER BY time DESC LIMIT 5
```

**Fields:**

- temperature_C
- humidity
- wind_speed_kmh, wind_speed_ms, wind_speed_kn
- wind_bft, wind_direction, wind_direction_deg
- air_pressure_hPa
- dew_point_C
- air_density
- visibility_m
- summary

**Tags:**

- location

#### 2. turbine

Stores wind turbine sensor data:

```sql
SELECT * FROM turbine ORDER BY time DESC LIMIT 5
```

**Fields:** (varies based on eGauge configuration)

- grid_* (power readings)
- orientation_* (orientation sensors)
- vibration_* (vibration sensors)

**Tags:**

- source: "rdm_windturbine"

## Configuration

### Data Retention

Check retention policies:

```sql
SHOW RETENTION POLICIES ON iot_data
```

Default is infinite. To set a retention policy (e.g., 90 days):

```sql
CREATE RETENTION POLICY "90_days" ON "iot_data" DURATION 90d REPLICATION 1 DEFAULT
```

### Continuous Queries

For downsampling high-frequency data (optional):

```sql
CREATE CONTINUOUS QUERY "downsample_turbine" ON "iot_data"
BEGIN
  SELECT mean(*) INTO "iot_data"."90_days"."turbine_1h"
  FROM "turbine"
  GROUP BY time(1h), *
END
```

This creates hourly averages from 30-second data.

## Backup & Restore

### Backup Database

```bash
docker exec influxdb influxd backup -portable -database iot_data /tmp/backup
docker cp influxdb:/tmp/backup ./influxdb_backup_$(date +%Y%m%d)
```

### Restore Database

```bash
docker cp ./influxdb_backup_20260107 influxdb:/tmp/restore
docker exec influxdb influxd restore -portable -db iot_data /tmp/restore
```

## Connecting from Grafana

To connect Grafana to InfluxDB:

1. In Grafana, add InfluxDB as a data source
2. Use URL: `http://influxdb:8086` (Docker DNS name)
3. Database: `iot_data`
4. Leave authentication blank if not enabled

For detailed steps, see [Grafana Configuration - Add InfluxDB Data Source](grafana.md#add-influxdb-data-source).

## Connecting from Node-RED

Install InfluxDB node:

1. In Node-RED UI, go to Menu → Manage palette
2. Install tab → Search "node-red-contrib-influxdb"
3. Click Install

Configure in flow:

- **Server**: `influxdb` (use Docker DNS name) or `<vm_ip>` if Node-RED runs outside Docker
- **Port**: `8086`
- **Database**: `iot_data`
- **Username**: (leave blank if no auth)
- **Password**: (leave blank if no auth)

## Monitoring

### Check Database Size

```sql
SHOW STATS FOR 'database'
```

### Check Series Cardinality

```sql
SHOW SERIES CARDINALITY
```

High cardinality can impact performance.

## Troubleshooting

### Connection Refused

Verify InfluxDB is running:

```bash
docker compose ps influxdb
```

Check if port 8086 is accessible:

```bash
curl -I http://<influxdb_ip>:8086/ping
```

Should return `HTTP/1.1 204 No Content` in the headers.

!!! info "IP Address Configuration"
    Replace `<influxdb_ip>` with your VM's IP address (e.g., `192.168.178.164`) or use `localhost` if testing locally on the VM.

### Out of Memory

Increase InfluxDB memory in `docker-compose.yml`:

```yaml
services:
  influxdb:
    mem_limit: 2g
```

### Slow Queries

Check query execution:

```sql
SHOW QUERIES
```

Kill slow query:

```sql
KILL QUERY <query-id>
```

## Useful Queries

### Count Records

```sql
SELECT COUNT(*) FROM weather
SELECT COUNT(*) FROM turbine
```

### Time Range

```sql
SELECT * FROM turbine WHERE time > now() - 1h
```

### Aggregate

```sql
SELECT MEAN(temperature_C) FROM weather WHERE time > now() - 24h GROUP BY time(1h)
```

### Delete Data

```sql
DELETE FROM weather WHERE time < '2025-01-01'
```

## Security (Optional)

### Enable Authentication

Edit InfluxDB config or environment variables in `docker-compose.yml`:

```yaml
environment:
  - INFLUXDB_HTTP_AUTH_ENABLED=true
  - INFLUXDB_ADMIN_USER=admin
  - INFLUXDB_ADMIN_PASSWORD=your_secure_password
```

Recreate container:

```bash
docker compose up -d influxdb
```

Update Grafana and Node-RED with credentials.
