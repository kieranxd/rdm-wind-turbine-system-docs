# USB Gadget Mode Setup

This guide configures a Raspberry Pi as a USB mass storage device to deliver images to the Samsung E-Ink display.

## Prerequisites

- Raspberry Pi 4 (or Pi Zero W/2W)
- Raspberry Pi OS Lite installed
- PoE+ HAT for power and network
- USB-C cable to display

## Enable USB Gadget Mode

This setup uses ConfigFS/usb_gadget, which is the modern way to configure USB OTG on Raspberry Pi. It's more flexible than the legacy `g_mass_storage` method and allows composing multiple USB functions.

### 1. Enable dwc2 Overlay

Edit `/boot/config.txt`:

```bash
sudo nano /boot/config.txt
```

Add at the end:

```
dtoverlay=dwc2,dr_mode=peripheral
```

The `dr_mode=peripheral` parameter sets the USB controller to peripheral (device) mode.

### 2. Load dwc2 Module on Boot

Edit `/etc/modules`:

```bash
sudo nano /etc/modules
```

Add to the end:

```
dwc2
```

This ensures the USB controller driver loads at startup.

### 3. Verify ConfigFS is Mounted

ConfigFS should be mounted by default on Raspberry Pi OS:

```bash
mount | grep configfs
```

Should show: `configfs on /sys/kernel/config type configfs`

If not mounted, mount it:

```bash
sudo mount -t configfs none /sys/kernel/config
```

To make it permanent, add to `/etc/fstab`:

```
configfs  /sys/kernel/config  configfs  defaults  0  0
```

### 4. Reboot

```bash
sudo reboot
```

## Create Mass Storage File

### Create Backing File

Create a 100MB disk image:

```bash
sudo dd if=/dev/zero of=/home/<pi_username>/drive.bin bs=1M count=100
```

### Format as FAT32

```bash
sudo mkfs.vfat /home/<pi_username>/drive.bin
```

### Mount and Create Folder

```bash
sudo mkdir -p /mnt/usb_drive
sudo mount -o loop /home/<pi_username>/drive.bin /mnt/usb_drive
sudo mkdir -p /mnt/usb_drive/SAMSUNG_E-Paper
sudo umount /mnt/usb_drive
```

The folder `SAMSUNG_E-Paper` is required by the Samsung display.

## Configure USB Gadget

The gadget configuration uses ConfigFS to create a mass storage device. This is done via the `gadget.sh` script.

### Understanding ConfigFS Structure

ConfigFS creates a virtual file system under `/sys/kernel/config/usb_gadget/` where:

- Each gadget is a directory (e.g., `EInkGadget`)
- Configuration is done by writing to files
- Functions (mass storage, ethernet, serial) are in `functions/`
- Configurations link functions together
- The gadget is activated by writing to the `UDC` file

### Install gadget.sh

Create the script at `/usr/local/bin/gadget.sh`:

```bash
sudo nano /usr/local/bin/gadget.sh
```

Paste the contents of `gadget.sh` (see [Shell Scripts](scripts.md) for complete code).

Key points:

- Loads `libcomposite` kernel module
- Creates gadget under `/sys/kernel/config/usb_gadget/EInkGadget`
- Configures USB IDs, strings, and mass storage function
- Binds to USB Device Controller (UDC)

Make executable:

```bash
sudo chmod +x /usr/local/bin/gadget.sh
```

### Run on Boot

Create systemd service at `/lib/systemd/system/usb-gadget.service`:

```bash
sudo nano /lib/systemd/system/usb-gadget.service
```

Content:

```ini
[Unit]
Description=USB Gadget Mode Configuration
After=systemd-user-sessions.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/gadget.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**Notes:**

- `After=systemd-user-sessions.service` ensures user sessions are ready
- `Type=oneshot` means script runs once and exits
- `RemainAfterExit=yes` keeps service marked as active

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable usb-gadget.service
sudo systemctl start usb-gadget.service
```

Verify:

```bash
sudo systemctl status usb-gadget.service
```

Should show "active (exited)" with no errors.

## Verify Gadget Mode

### Check USB Controller

```bash
ls /sys/class/udc
```

Should show the UDC (e.g., `fe980000.usb`).

### Check Gadget Directory

```bash
ls /sys/kernel/config/usb_gadget/EInkGadget/
```

Should show configuration directories.

### Check USB Binding

```bash
cat /sys/kernel/config/usb_gadget/EInkGadget/UDC
```

Should show the UDC name (not empty).

## Connect to Display

1. Connect USB-C cable from Pi to display
2. Display should detect mass storage device
3. Check display's USB mode settings
4. Should show "SAMSUNG_E-Paper" folder

## Testing

### Manually Add Image

Mount the drive:

```bash
sudo mount -o loop /home/<pi_username>/drive.bin /mnt/usb_drive
```

Copy a test image:

```bash
sudo cp /path/to/test.png /mnt/usb_drive/SAMSUNG_E-Paper/001.png
```

Unmount:

```bash
sudo umount /mnt/usb_drive
```

Refresh display to see image.

## Troubleshooting

### Gadget Not Created

Check kernel modules:

```bash
lsmod | grep -E 'dwc2|libcomposite'
```

Should show both modules loaded.

If `dwc2` missing:

```bash
sudo modprobe dwc2
```

Check `/etc/modules` contains `dwc2`.

If `libcomposite` missing, the gadget script loads it automatically.

### UDC Not Found

Verify `/boot/config.txt` has `dtoverlay=dwc2,dr_mode=peripheral`.

Check:

```bash
ls /sys/class/udc
```

Should show device like `fe980000.usb` (Pi 4) or similar.

If empty:

- Reboot: `sudo reboot`
- Check Pi model (Pi 4, Pi Zero W/2W have OTG support)
- Verify using correct USB port (USB-C port on Pi 4)

### ConfigFS Not Mounted

Check if mounted:

```bash
mount | grep configfs
```

If not mounted:

```bash
sudo mount -t configfs none /sys/kernel/config
```

Add to `/etc/fstab` for persistence.

### Display Not Detecting

1. Check USB cable (must support data, not just power)
2. Try different USB-C port on Pi (use the peripheral port, not power)
3. Check display USB mode settings
4. Verify gadget is bound: `cat /sys/kernel/config/usb_gadget/EInkGadget/UDC`

### Permission Errors

Ensure scripts run as root or with sudo:

```bash
sudo /usr/local/bin/gadget.sh
```

## Advanced: Multiple USB Functions

ConfigFS allows combining multiple USB functions. For example, to add both mass storage and serial console:

```bash
# In gadget.sh, after creating mass_storage.usb0:

# Add serial function
mkdir functions/acm.usb0

# Link both functions to configuration
ln -s functions/mass_storage.usb0 configs/c.1/
ln -s functions/acm.usb0 configs/c.1/
```

The Pi will appear as both a USB drive and a serial device.

**Other available functions:**

- `rndis.usb0` - Ethernet over USB (Windows)
- `ecm.usb0` - Ethernet over USB (Linux/Mac)
- `acm.usb0` - Serial console
- `hid.usb0` - Keyboard/Mouse emulation

See [Linux USB Gadget documentation](https://www.kernel.org/doc/Documentation/usb/gadget_configfs.txt) for more.

## Security Notes

- Scripts require root access
- Secure SSH access to Pi
- Use key-based authentication only
- Disable password authentication

See [SSH Configuration](ssh-setup.md).

## Resources

- Official ConfigFS docs: [https://docs.kernel.org/driver-api/usb/gadget.html](https://docs.kernel.org/driver-api/usb/gadget.html)
- Kernel ConfigFS USB Gadget: [https://www.kernel.org/doc/Documentation/ABI/testing/configfs-usb-gadget](https://www.kernel.org/doc/Documentation/ABI/testing/configfs-usb-gadget)
- Raspberry Pi USB OTG: [https://www.raspberrypi.com/documentation/computers/raspberry-pi.html](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html)

## Next Steps

- [Configure shell scripts](scripts.md) for automated image handling
- [Set up SSH access](ssh-setup.md) from Node-RED
