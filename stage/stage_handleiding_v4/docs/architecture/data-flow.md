# Data Flow

This document details how data moves through the system from sensors to display.

## Flow 1: Weather Data Collection

```mermaid
flowchart TD
    Weather["Weather Station API<br/>weerlive.nl"]
    HTTPReq["Node-RED<br/>HTTP Request Node"]
    Parse["Node-RED<br/>Parse Function<br/>• Extract temp<br/>• Extract humidity<br/>• Calculate air density"]
    DB["InfluxDB<br/>weather measurement"]
    
    Weather -->|HTTPS GET<br/>Every 5 min| HTTPReq
    HTTPReq -->|JSON Response| Parse
    Parse -->|InfluxDB Point| DB
    
    style Weather fill:#81c784,stroke:#43a047,color:#fff
    style HTTPReq fill:#66bb6a,stroke:#388e3c,color:#fff
    style Parse fill:#aed581,stroke:#7cb342,color:#fff
    style DB fill:#42a5f5,stroke:#1e88e5,color:#fff
```

### Weather Data Fields Stored

| Field | Description | Unit |
|-------|-------------|------|
| temperature_C | Air temperature | °C |
| humidity | Relative humidity | % |
| wind_speed_kmh | Wind speed | km/h |
| wind_speed_ms | Wind speed | m/s |
| wind_speed_kn | Wind speed | knots |
| wind_bft | Beaufort scale | - |
| wind_direction | Cardinal direction | text |
| wind_direction_deg | Wind direction | degrees |
| air_pressure_hPa | Atmospheric pressure | hPa |
| dew_point_C | Dew point | °C |
| air_density | Calculated air density | kg/m³ |
| visibility_m | Visibility | meters |
| summary | Weather description | text |

### Air Density Calculation

```javascript
let P = weather.luchtd * 100;  // hPa → Pa
let R = 287;                    // J/(kg·K)
let T_K = T + 273.15;           // °C → K
let air_density = P / (R * T_K);
```

## Flow 2: Turbine Data Collection

```mermaid
flowchart TD
    eGauge["Wind Turbine eGauge<br/>egauge84432.egaug.es"]
    HTTPReq["Node-RED<br/>HTTP Request Node"]
    XMLParse["Node-RED<br/>XML Parser Node"]
    Parse["Node-RED<br/>Parse Function<br/>• Sanitize field names<br/>• Convert values<br/>• Add prefixes"]
    DB["InfluxDB<br/>turbine measurement"]
    
    eGauge -->|HTTPS GET<br/>Every 30 sec| HTTPReq
    HTTPReq -->|XML Response| XMLParse
    XMLParse -->|Parsed Object| Parse
    Parse -->|InfluxDB Point| DB
    
    style eGauge fill:#ef5350,stroke:#c62828,color:#fff
    style HTTPReq fill:#66bb6a,stroke:#388e3c,color:#fff
    style XMLParse fill:#ba68c8,stroke:#8e24aa,color:#fff
    style Parse fill:#aed581,stroke:#7cb342,color:#fff
    style DB fill:#42a5f5,stroke:#1e88e5,color:#fff
```

### Field Name Sanitization

The parse function converts eGauge field names to snake_case and adds prefixes:

| Original Prefix | New Prefix | Example |
|----------------|------------|---------|
| S6_, S5_, S2_ | grid_ | `S6_Power` → `grid_s6_power` |
| IM_ | orientation_ | `IM_X` → `orientation_x` |
| VM_ | vibration_ | `VM_Accel` → `vibration_accel` |

## Flow 3: Image Rendering & Delivery

```mermaid
flowchart TD
    Inject["Node-RED<br/>Inject Node<br/>(Every 5 min)"]
    HTTPReq["Node-RED<br/>HTTP Request to Grafana<br/>Bearer Auth"]
    Renderer["Grafana Image Renderer<br/>(Headless Chromium)"]
    WriteFile["Node-RED<br/>Write File<br/>/tmp/001.png"]
    
    subgraph PathA["Path A: USB Mode"]
        SCP["Node-RED<br/>SCP to RPi<br/>via SSH"]
        Incoming["Raspberry Pi<br/>/home/hrtech/incoming_images/"]
        RefreshScript["Shell Script<br/>refresh_script.sh<br/>• Unbind USB<br/>• Resize to 2560x1440<br/>• Mount drive.bin<br/>• Copy image<br/>• Unmount<br/>• Rebind USB"]
        USBStorage["USB Mass Storage<br/>(SAMSUNG_E-Paper/)"]
        DisplayUSB["Samsung E-Ink Display<br/>(USB Mode)"]
    end
    
    Inject --> HTTPReq
    HTTPReq --> Renderer
    Renderer -->|PNG Binary| WriteFile
    WriteFile --> SCP
    
    SCP --> Incoming
    Incoming --> RefreshScript
    RefreshScript --> USBStorage
    USBStorage -->|USB-C Cable| DisplayUSB
    
    style Inject fill:#66bb6a,stroke:#388e3c,color:#fff
    style HTTPReq fill:#66bb6a,stroke:#388e3c,color:#fff
    style Renderer fill:#ffb74d,stroke:#fb8c00,color:#fff
    style WriteFile fill:#aed581,stroke:#7cb342,color:#fff
    style SCP fill:#4dd0e1,stroke:#00acc1,color:#fff
    style PrepareScript fill:#ba68c8,stroke:#8e24aa,color:#fff
    style UpdateScript fill:#ba68c8,stroke:#8e24aa,color:#fff
    style PublishScript fill:#ba68c8,stroke:#8e24aa,color:#fff
    style DisplayUSB fill:#9575cd,stroke:#5e35b1,color:#fff
```

## Timing & Frequencies

| Activity | Frequency | Node-RED Node |
|----------|-----------|---------------|
| Weather data poll | Every 5 minutes (300s) | loop_weather_data |
| Turbine data poll | Every 30 seconds | loop_turbine_data |
| Image render & delivery | Every 5 minutes (300s) | loop_image_renderer |

## Authentication & Tokens

### Grafana to Image Renderer
- **Method**: Bearer token authentication
- **Token**: `glsa_AzyNQ7uhzHzg4UCe5KC8fHUAiJPGjBaz_0d0c9ffb`
- **Environment Variables**:
    - Grafana: `GF_RENDERING_RENDERER_TOKEN`
    - Renderer: `AUTH_TOKEN`

!!! warning "Security Note"
    Tokens shown here are examples from your development environment. In production, regenerate and secure these tokens.

### Node-RED to Raspberry Pi
- **Method**: SSH with key-based authentication
- **Network**: Tailscale VPN (encrypted mesh network)
- **Command**: `scp /tmp/001.png <pi-user>@<pi-tailscale-ip>:/home/<pi-user>/incoming_images/incoming.png`
  - Example: `scp /tmp/001.png hrtech@100.64.1.5:/home/hrtech/incoming_images/incoming.png`
  - Uses Tailscale IP (typically `100.x.x.x` range)

## Error Handling

### Node-RED Flow Logic
- Each loop operates independently
- Failed HTTP requests logged to debug nodes
- Continue on error to prevent blocking other flows

### Raspberry Pi Scripts
- `set -e` in scripts: Exit on any error
- Check if USB gadget is bound before updating
- Verify mount points exist before operations

## Data Retention

- **InfluxDB**: Default retention policy (check IOTstack settings)
- **Grafana**: Dashboard configured for last 3 hours (`from=now-3h&to=now`)
- **Images**: Only latest image kept on Raspberry Pi and display
