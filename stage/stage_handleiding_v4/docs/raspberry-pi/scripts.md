# Shell Scripts

The Raspberry Pi uses shell scripts to manage image processing and USB mass storage updates.

## Script Overview

| Script | Purpose | Triggered By |
| ------ | ------- | ------------ |
| `gadget.sh` | Configure USB gadget mode | System boot (systemd) |
| `refresh_script.sh` | Resize, update, and refresh display images | Node-RED via SSH |

!!! warning "Critical: Replace Placeholders in Scripts"
    All scripts below contain `<pi_username>` as a placeholder. You **MUST** replace this with your actual Raspberry Pi username (e.g., `hrtech`, `pi`, etc.) when creating the scripts, otherwise they will not work.

## gadget.sh

**Purpose:** Configure the Raspberry Pi as a USB mass storage device.

**Location:** `/usr/local/bin/gadget.sh`

**Run by:** systemd on boot

### Complete Script

```bash
#!/bin/bash
modprobe libcomposite
cd /sys/kernel/config/usb_gadget || exit

mkdir -p EInkGadget
cd EInkGadget || exit

# Vendor/Product IDs
echo 0x1d6b > idVendor
echo 0x0104 > idProduct
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB

# Strings
mkdir -p strings/0x409
echo "RaspberryPi" > strings/0x409/manufacturer
echo "EInkUSB" > strings/0x409/product
SERIAL=$(awk '/Serial/ {print $3}' /proc/cpuinfo)
echo "$SERIAL" > strings/0x409/serialnumber

# Configuration
mkdir -p configs/c.1/strings/0x409
echo 120 > configs/c.1/MaxPower
echo "Config 1" > configs/c.1/strings/0x409/configuration

# Mass storage function
mkdir -p functions/mass_storage.usb0
echo "/home/<pi_username>/drive.bin" > functions/mass_storage.usb0/lun.0/file
echo 1 > functions/mass_storage.usb0/lun.0/removable
echo 0 > functions/mass_storage.usb0/lun.0/cdrom

# Link function
ln -s functions/mass_storage.usb0 configs/c.1/

UDC=$(ls /sys/class/udc | head -n 1)
echo "$UDC" > UDC
```

### Installation

```bash
sudo nano /usr/local/bin/gadget.sh
# Paste script content
# IMPORTANT: Replace <pi_username> with your actual username!
sudo chmod +x /usr/local/bin/gadget.sh
```

### Systemd Service

Create `/etc/systemd/system/usb-gadget.service`:

```ini
[Unit]
Description=USB Gadget Mode Configuration
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/gadget.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl enable usb-gadget.service
sudo systemctl start usb-gadget.service
```

## refresh_script.sh

**Purpose:** Unified script that unbinds USB, resizes images, updates the USB drive, and rebinds USB. Replaces the previous separate scripts (prepare_images.sh, update_usb_images.sh, publish_to_screen.sh).

**Location:** `/home/<pi_username>/refresh_script.sh`

**Run by:** Node-RED via SSH

### Complete Script

```bash
#!/bin/bash
set -e

# --- Paths ---
GADGET=/sys/kernel/config/usb_gadget/EInkGadget
BACKING=/home/<pi_username>/drive.bin
MOUNT=/mnt
SRC=/home/<pi_username>/incoming_images
DST_FOLDER="SAMSUNG_E-Paper"

# --- 1. Unbind gadget if active ---
UDC=$(ls /sys/class/udc | head -n 1)
if [ -n "$(cat $GADGET/UDC 2>/dev/null || true)" ]; then
    echo "Unbinding USB gadget..."
    echo "" | sudo tee "$GADGET/UDC"
    sleep 1
fi

# --- 2. Create backing file if missing ---
if [ ! -f "$BACKING" ]; then
    echo "Creating 8GB FAT32 backing file..."
    dd if=/dev/zero of="$BACKING" bs=1M count=8192 status=progress
    mkfs.vfat -F 32 -n EINKUSB "$BACKING"
fi

# --- 3. Mount .bin ---
sudo mkdir -p "$MOUNT"
sudo umount "$MOUNT" 2>/dev/null || true
sudo mount -o loop "$BACKING" "$MOUNT"

# Ensure the folder exists
mkdir -p "$MOUNT/$DST_FOLDER"

# --- 4. Prepare image with alternating filename ---
LAST_NUM_FILE="$MOUNT/.last_num"

# Determine next filename: 001.png <-> 002.png
if [ -f "$LAST_NUM_FILE" ]; then
    LAST_NUM=$(cat "$LAST_NUM_FILE")
    NUM=$(( (LAST_NUM % 2) + 1 ))   # 1 -> 2, 2 -> 1
else
    NUM=1
fi

printf -v FILENAME "%03d.png" "$NUM"

# Pick the first image in incoming_images
IMG=$(ls "$SRC"/*.{png,jpg,jpeg} 2>/dev/null | head -n1)
if [ -z "$IMG" ]; then
    echo "No images found in $SRC"
    sudo umount "$MOUNT"
    exit 1
fi

# Remove old files in folder
rm -f "$MOUNT/$DST_FOLDER"/*

# Convert and copy to .bin (landscape mode: 2560x1440)
convert "$IMG" \
    -resize 2560x1440 \
    -background white \
    -gravity center \
    -extent 2560x1440 \
    "$MOUNT/$DST_FOLDER/$FILENAME"

# Remember last number
echo "$NUM" > "$LAST_NUM_FILE"

# --- 5. Flush and unmount ---
sync
sudo umount "$MOUNT"

# --- 6. Rebind gadget ---
if [ -n "$UDC" ]; then
    echo "$UDC" | sudo tee "$GADGET/UDC"
    sleep 2
fi

echo "E-Ink USB refresh complete. Current file: $FILENAME"
```

### Dependencies

Requires ImageMagick:

```bash
sudo apt install imagemagick -y
```

### How It Works

1. **Unbind USB:** Disconnects USB gadget if currently active
2. **Create backing file:** Creates 8GB FAT32 drive.bin if it doesn't exist
3. **Mount:** Mounts the backing file to `/mnt`
4. **Alternating filenames:** Uses 001.png and 002.png alternately to force display refresh
5. **Resize image:** Converts first image from incoming_images to 2560x1440 (landscape)
6. **Update:** Copies processed image to USB drive
7. **Unmount:** Cleanly unmounts the drive
8. **Rebind USB:** Reconnects USB gadget to display

### Installation

```bash
nano /home/<pi_username>/refresh_script.sh
# Paste script content
# IMPORTANT: Replace ALL instances of <pi_username> with your actual username!
chmod +x /home/<pi_username>/refresh_script.sh
```

### Permissions

Requires sudo for USB unbind/bind and mount operations. Configure sudoers:

```bash
sudo visudo
```

Add (replace `<pi_username>` with your username):

```
<pi_username> ALL=(ALL) NOPASSWD: /usr/bin/tee, /usr/bin/mount, /usr/bin/umount, /usr/bin/mkdir
```

## Trigger from Node-RED

Node-RED uses SSH to trigger `refresh_script.sh`:

```bash
ssh <pi_username>@<pi_tailscale_ip> '/home/<pi_username>/refresh_script.sh'
```

Example:

```bash
ssh hrtech@100.64.1.5 '/home/hrtech/refresh_script.sh'
```

See [SSH Configuration](ssh-setup.md) for key-based auth setup.

## Directory Structure

```
/home/<pi_username>/
├── drive.bin                    # USB backing file (8GB FAT32)
├── incoming_images/             # Images from Node-RED via SCP
│   └── 001.png
└── refresh_script.sh            # Main refresh script
```

## Testing Script

### Test refresh_script.sh

```bash
# Add a test image
cp /path/to/test.png /home/<pi_username>/incoming_images/test.png

# Run script
/home/<pi_username>/refresh_script.sh

# Check display for updated image
```

## Troubleshooting

### "No images found" Error

Ensure images exist in incoming_images folder:

```bash
ls -lh /home/<pi_username>/incoming_images/
```

Add a test image if needed:

```bash
cp /path/to/test.png /home/<pi_username>/incoming_images/test.png
```

### "convert: command not found"

Install ImageMagick:

```bash
sudo apt install imagemagick -y
```

### Permission Denied

Ensure script is executable:

```bash
chmod +x /home/<pi_username>/refresh_script.sh
```

And sudoers configured correctly (see Permissions section above).

### Mount Errors

Check if already mounted:

```bash
mount | grep drive.bin
```

If mounted, unmount:

```bash
sudo umount /mnt
```

### USB Gadget Already Bound

The script automatically unbinds the gadget. If manual unbind is needed:

```bash
echo "" | sudo tee /sys/kernel/config/usb_gadget/EInkGadget/UDC
```

## Maintenance

### Clean Old Images

Periodically clean incoming images:

```bash
rm -f /home/<pi_username>/incoming_images/*
```

### Check Disk Usage

```bash
du -sh /home/<pi_username>/drive.bin
```

### Verify Backing File

```bash
file /home/<pi_username>/drive.bin
```

Should show FAT filesystem.

## Script Logging

Add logging to `refresh_script.sh` by modifying the last echo line:

```bash
echo "E-Ink USB refresh complete. Current file: $FILENAME" | tee -a /home/<pi_username>/refresh.log
```

View logs:

```bash
tail -f /home/<pi_username>/refresh.log
```
