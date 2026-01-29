# PoE Setup

Power over Ethernet (PoE+) is used to power both the Raspberry Pi and Samsung E-Ink Display.

## PoE Standards

### PoE (802.3af)

- **Power**: Up to 15.4W at switch, 12.95W at device
- **Voltage**: 48V DC
- **Use**: Low-power devices

### PoE+ (802.3at)

- **Power**: Up to 30W at switch, 25.5W at device
- **Voltage**: 48V DC
- **Use**: Higher-power devices (our setup)

### PoE++ (802.3bt)

- **Power**: Up to 60-100W
- **Not required** for this project

## Devices Using PoE+

### Samsung EM32DX E-Ink Display

- **Standard**: PoE+ (802.3at)
- **Power Draw**: Approximately 20-25W
- **Voltage**: 48V DC (stepped down internally)
- **Connector**: RJ45 Ethernet

### Raspberry Pi with PoE+ HAT

- **HAT Model**: Official Raspberry Pi PoE+ HAT
- **Standard**: PoE+ (802.3at)
- **Power Draw**: 5-8W (depending on load)
- **Max Power**: 15W
- **Voltage**: 48V DC → 5V DC (via HAT)
- **Cooling**: Fan included on HAT (temperature-controlled)

## PoE+ Switch Requirements

### Specifications

- **Standard**: 802.3at (PoE+) or 802.3bt (PoE++)
- **Power Budget**: Minimum 60W total
  - Display: 25W
  - Raspberry Pi: 8W
  - Overhead: 10W
  - Reserve: 17W
- **Ports**: Minimum 4 (uplink, VM, Pi, Display)

### Recommended Switches

- **Unmanaged**: TP-Link TL-SG1005P (4 PoE+ ports, 65W budget)
- **Managed**: Netgear GS305EP (4 PoE+ ports, 63W budget)
- **Enterprise**: Cisco SG350-10P (8 PoE+ ports, 124W budget)

## Cable Requirements

### Category

- **Minimum**: Cat5e
- **Recommended**: Cat6 or Cat6a
- **Maximum Length**: 100 meters (328 feet)

### Wiring Standard

- **T568A or T568B** (both work, must be consistent)
- All **8 wires** must be connected for PoE

### Testing

Verify cable quality:

```bash
# On Linux device
sudo ethtool eth0
```

Look for:

- Speed: 1000Mb/s (Gigabit)
- Duplex: Full

## Installation

### Raspberry Pi PoE+ HAT

#### 1. Assemble HAT

1. Attach 40-pin header to Raspberry Pi GPIO
2. Align HAT with GPIO pins
3. Press down gently until seated
4. Secure with provided spacers and screws

#### 2. Connect Ethernet

1. Connect Ethernet cable to Pi
2. Other end to PoE+ switch port
3. Pi should power on automatically (red LED)

#### 3. Verify Power

Check Pi is receiving power:

```bash
vcgencmd get_throttled
```

Output `0x0` means no power issues.

### Samsung Display PoE+ Setup

1. Connect Ethernet cable to display's PoE+ port
2. Other end to PoE+ switch port
3. Display should power on automatically
4. Check network connection status via Samsung app (network configuration, content management)

## Power Budget Management

### Calculate Total Draw

| Device | Power Draw | PoE Port |
| ------ | ---------- | -------- |
| Samsung Display | 25W | Port 4 |
| Raspberry Pi | 8W | Port 3 |
| **Total** | **33W** | - |

Ensure switch power budget ≥ 33W (preferably 60W+ for headroom).

### Check Switch Power Status

If using managed switch:

1. Log into switch web interface
2. Navigate to PoE status
3. Verify:
   - Each port shows power delivered
   - Total budget not exceeded
   - No "overload" warnings

## Troubleshooting

### Device Not Powering On

#### Check 1: PoE Capability

Verify switch supports PoE+:

- Check label on switch for "802.3at" or "PoE+"
- Check switch specifications

#### Check 2: Cable

- Use Cat5e or better
- Ensure all 8 wires connected
- Test with known-good cable

#### Check 3: Port Configuration

If managed switch:

- Log into switch interface
- Enable PoE on specific port
- Set to "Auto" or "Force" mode

#### Check 4: Power Budget

- Check if other PoE devices consuming budget
- Disconnect other PoE devices temporarily
- Verify total draw < switch capacity

### Intermittent Power

#### Symptoms

- Device reboots randomly
- Network disconnects
- Underpowered warnings

#### Solutions

1. **Check cable length**: Must be <100m
2. **Verify cable quality**: Use Cat6 for longer runs
3. **Check crimps**: Poorly crimped connectors cause resistance
4. **Monitor power draw**:

```bash
# On Raspberry Pi
vcgencmd get_throttled
```

If not `0x0`, power issues detected.

5. **Check switch temperature**: Overheating reduces power output

### PoE HAT Fan Noise

#### Adjust Fan Settings

Edit Pi config:

```bash
sudo nano /boot/config.txt
```

Add:

```
dtparam=poe_fan_temp0=60000
dtparam=poe_fan_temp1=65000
dtparam=poe_fan_temp2=70000
dtparam=poe_fan_temp3=75000
```

This sets fan thresholds (in millidegrees Celsius):

- Below 60°C: Fan off
- 60-65°C: Low speed
- 65-70°C: Medium speed
- 70-75°C: High speed
- Above 75°C: Full speed

Reboot:

```bash
sudo reboot
```

## Safety Considerations

### Voltage

- PoE uses 48V DC
- Safe for humans (low current)
- Still, avoid touching exposed conductors

### Heat

- PoE switches generate heat
- Ensure adequate ventilation
- Don't stack devices on top of switch

### Certification

- Use only certified PoE equipment
- Avoid cheap/uncertified PoE injectors
- Check for CE/FCC/UL marks

## Alternative Power Options

If PoE is not available:

### Raspberry Pi

- **USB-C Power Supply**: Official 5V 3A adapter
- **PoE Injector**: Passive or active injector

### Samsung Display

- **DC Power Adapter**: Check display specs
- **PoE Injector**: 802.3at compatible injector

## Power Consumption Monitoring

### Raspberry Pi

Check current draw:

```bash
vcgencmd measure_volts
vcgencmd measure_current
```

### Display

Power consumption monitoring is limited on the Samsung E-Ink display. Configuration and monitoring available through the Samsung app includes:

- Network settings
- Content management
- Screen protection settings

**Note**: Real-time power consumption monitoring may not be available. Check switch-level monitoring instead.

### Switch

If managed switch, view per-port power consumption:

- Log into web interface
- PoE status page
- Shows watts per port

## Maintenance

### Monthly Checks

1. Verify all devices powered
2. Check switch status LEDs
3. Test network connectivity
4. Monitor temperatures
5. Check display status via Samsung app

### Annual Maintenance

1. Inspect cables for damage
2. Clean PoE switch vents
3. Verify power budget still adequate
4. Update switch firmware

## PoE Standards Comparison

| Standard | IEEE | Max Power (Switch) | Max Power (Device) | Voltage |
| -------- | ---- | ------------------ | ------------------ | ------- |
| PoE | 802.3af | 15.4W | 12.95W | 48V |
| PoE+ | 802.3at | 30W | 25.5W | 48V |
| PoE++ Type 3 | 802.3bt | 60W | 51W | 48V |
| PoE++ Type 4 | 802.3bt | 100W | 71W | 48V |

Our setup requires **PoE+** minimum.

## Cable Pinout

PoE uses pairs 2 and 3 (pins 1,2,3,6 and 4,5,7,8):

```
Pin 1: TX+ (Data & Power)
Pin 2: TX- (Data & Power)
Pin 3: RX+ (Data & Power)
Pin 4: Power+
Pin 5: Power-
Pin 6: RX- (Data & Power)
Pin 7: Power+
Pin 8: Power-
```

Standard Ethernet data uses pins 1,2,3,6.
PoE adds power on pins 4,5,7,8 or uses data pairs.

## Future Upgrades

### Higher Power Devices

If adding devices >25W:

- Upgrade to PoE++ (802.3bt) switch
- Verify cable supports higher power
- Recalculate total power budget

### Additional PoE Devices

When adding more:

1. Calculate total power draw
2. Ensure switch budget sufficient
3. Consider redundant power supply
4. Document new device power requirements
