# Samsung E-Ink Display

Configuration and usage of the Samsung EM32DX 32" E-Paper display.

## Display Specifications

- **Model**: Samsung EM32DX
- **Size**: 32 inches diagonal
- **Technology**: E-Paper (E-Ink)
- **Resolution**: 2560 x 1440 pixels
- **Orientation**: Landscape
- **Power**: PoE+ (802.3at)
- **Connectivity**:
  - Ethernet (PoE+)
  - USB-C (data/mass storage)
  - Wi-Fi (optional)

## Display Modes

The display supports two methods for showing content:

### 1. USB Mass Storage Mode

**How it works:**

- Raspberry Pi in USB gadget mode appears as USB drive
- Display reads images from `SAMSUNG_E-Paper` folder
- Supports multiple images (001.png, 002.png, etc.)
- Display cycles through images

**Advantages:**

- Simple setup
- No network configuration needed
- Reliable (direct USB connection)

**Disadvantages:**

- Requires physical USB connection
- Manual refresh triggered by Pi scripts

**Configuration:**

See [Raspberry Pi Gadget Mode](../raspberry-pi/gadget-mode.md).

**Note:** This is the only display method used in this project. USB mass storage provides a reliable, simple solution without requiring custom app development.

## Initial Setup

### Power On

1. Connect PoE+ Ethernet cable
2. Display powers on automatically
3. Wait for boot (30-60 seconds)

### Access Settings

1. Press menu button on display (or remote)
2. Navigate to Settings
3. Configure network and USB settings

### Network Configuration

#### Wired (PoE+)

1. Settings → Network → Wired
2. Set static IP or DHCP
3. Test connection: ping display IP from VM

#### Wireless (Optional)

1. Settings → Network → Wi-Fi
2. Select SSID
3. Enter password
4. Save

### USB Settings

1. Settings → USB
2. Enable USB Mass Storage mode
3. Set auto-detect to On
4. Save

## Image Requirements

### Format

- **File Type**: PNG (preferred) or JPEG
- **Color**: Grayscale or RGB (will be converted to grayscale)
- **Bit Depth**: 8-bit or higher

### Resolution

- **Display Native**: 2560 x 1440 pixels
- **Aspect Ratio**: 16:9 (landscape)

**Recommendations:**

- Match native resolution for best quality
- Use `refresh_script.sh` to auto-resize (see [Scripts](../raspberry-pi/scripts.md))

### File Size

- **Recommended**: <2 MB per image
- **Maximum**: Display-dependent (check specs)

### Naming Convention

For USB mode, use sequential naming:

- `001.png`
- `002.png`
- `003.png`

Display reads in numerical order.

## Display Configuration

### Refresh Settings

E-Ink displays have different refresh modes:

#### Full Refresh

- **Speed**: Slow (1-2 seconds)
- **Quality**: High (no ghosting)
- **Use**: For detailed images, graphs

#### Partial Refresh

- **Speed**: Fast (0.5 seconds)
- **Quality**: May show ghosting
- **Use**: For frequent updates, text

**Configure:**

1. Settings → Display → Refresh Mode
2. Choose Full or Partial
3. Save

### Brightness

E-Ink uses ambient light (no backlight).

Optional front light:

1. Settings → Display → Front Light
2. Adjust intensity
3. Save

### Auto Rotate

1. Settings → Display → Orientation
2. Select Portrait or Landscape
3. Save

## Slideshow Settings (USB Mode)

### Configure Slideshow

1. Settings → USB → Slideshow
2. **Interval**: Set time between images (e.g., 300 seconds)
3. **Loop**: Enable to repeat
4. **Transition**: Select fade or instant
5. Save

### Manual Advance

Use remote or buttons to manually navigate images.

## Connecting to Raspberry Pi

### Physical Connection

1. **USB-C cable** from Raspberry Pi to display
2. Ensure Pi is powered and in gadget mode
3. Display should auto-detect USB drive

### Verify Connection

1. On display, navigate to USB source
2. Should show "EInkUSB" or similar device name
3. Open device to see `SAMSUNG_E-Paper` folder
4. Images should be listed

### Troubleshooting Connection

If display doesn't detect USB:

1. Check cable (must support data)
2. Verify Pi gadget mode active:

```bash
cat /sys/kernel/config/usb_gadget/EInkGadget/UDC
```

Should show UDC name, not empty.

3. Restart both devices
4. Check display USB settings (must be enabled)

## Image Quality Optimization

### For E-Ink Displays

E-Ink is different from LCD:

- **High Contrast**: Use bold text, clear graphics
- **Grayscale**: Color is converted to gray
- **No Animation**: Refresh rate is slow
- **Power Efficient**: Image persists without power

### Grafana Dashboard Design

For best results:

1. **High Contrast Theme**: Dark background, light text (or vice versa)
2. **Large Text**: Minimum 14pt font
3. **Bold Lines**: Thick graph lines (2-3px)
4. **Limit Colors**: Use grayscale-friendly colors
5. **Avoid Gradients**: Solid fills preferred

### Testing

Render and view on actual display to verify readability.

## Maintenance

### Cleaning

- **Screen**: Use soft, dry microfiber cloth
- **Avoid**: Liquid cleaners, abrasives
- **Frequency**: As needed (E-Ink accumulates less dust)

### Ghosting Prevention

E-Ink can show ghosting (image retention).

**Prevention:**

- Perform full refresh periodically
- Vary content to avoid static images
- Use display's "Screen Refresh" function monthly

**To Refresh:**

1. Settings → Display → Screen Refresh
2. Display will flash black/white several times
3. Clears ghosting

### Firmware Updates

Check for updates:

1. Settings → System → About
2. Check for updates
3. Download and install if available

Or visit Samsung support website.

## Power Management

### Sleep Mode

1. Settings → Power → Sleep Timer
2. Set inactivity timeout (e.g., Never for always-on)
3. Save

### Auto Power On/Off

1. Settings → Power → Schedule
2. Set on/off times
3. Save

For 24/7 monitoring, disable auto-off.

## Troubleshooting

### Display Not Powering On

1. Check PoE+ connection
2. Verify switch provides PoE+ (not just PoE)
3. Try different switch port
4. Check cable

See [PoE Setup](poe.md) for more.

### Image Not Updating

#### USB Mode

1. Verify Pi is connected (check USB cable)
2. Check images exist in `SAMSUNG_E-Paper` folder:

```bash
ssh <pi_username>@<rpi_ip> 'ls -lh /mnt/usb_drive/SAMSUNG_E-Paper/'
```

3. Trigger manual refresh on display
4. Check display slideshow settings

### Display Frozen

1. Unplug PoE cable
2. Wait 30 seconds
3. Reconnect
4. Display should reboot

### Poor Image Quality

1. Check image resolution matches display (2560x1440)
2. Use full refresh mode
3. Increase text size and line thickness
4. Verify PNG format (not compressed JPEG)

## Advanced Features

### Remote Management

If display supports:

- **Web Interface**: Access via `http://<display-ip>`
- **SNMP**: Monitor status via SNMP
- **Samsung MagicInfo**: Enterprise content management

Consult display manual for details.

### Content Management

For more complex content:

- Use Samsung MagicInfo software
- Schedule content playback
- Manage multiple displays

## Security

### Display Settings Password

1. Settings → Security → Set Password
2. Enter and confirm password
3. Save

Prevents unauthorized changes.

### Network Security

- Use VLAN to isolate display (if using managed switch)
- Firewall rules to limit access
- Disable unused services (Wi-Fi if not needed)

## Specifications Reference

| Feature | Specification |
| ------- | ------------- |
| Display Size | 32" diagonal |
| Resolution | 2560 x 1440 pixels |
| Aspect Ratio | 16:9 (landscape) |
| Technology | E-Paper (E-Ink) |
| Brightness | Ambient light reflective |
| Viewing Angle | 180° (E-Ink characteristic) |
| Power | PoE+ (802.3at), <25W |
| Refresh Rate | 1-2 seconds (full refresh) |
| Operating Temp | 0-40°C |
| Humidity | 10-80% RH (non-condensing) |
| MTBF | Check manufacturer specs |

## Support Resources

- **Samsung Support**: [https://www.samsung.com/support/](https://www.samsung.com/support/)
- **User Manual**: Consult included documentation
