# Raspberry Pi 5 Setup & Optimization for Klipper MK3S/+

## TESTED AND CONFIRMED 2026-04-01

## Why Pi 5?

The Raspberry Pi 5 has a **dedicated xHCI USB controller** separate from WiFi. On the Pi Zero 2W (and Pi 3B), USB and WiFi share the same `dwc_otg` bus — if the USB connection to the Einsy crashes (due to the 32u2 firmware bug or heavy traffic), WiFi dies too, requiring a full power cycle.

The Pi 5 eliminates this single point of failure. Even if the 32u2 has issues, the Pi stays online and accessible.

| Feature | Pi Zero 2W | Pi 5 4GB |
|---------|-----------|----------|
| USB controller | dwc_otg (shared with WiFi) | xHCI (dedicated) |
| WiFi crash on USB error | Yes | No |
| CPU cores | 4x 1GHz | 4x 2.4GHz |
| RAM | 512MB | 4GB |
| enable_object_processing | Not recommended | Works fine |
| Typical load during print | 0.3-0.5 | 0.03-0.07 |

## Installation

### Step 1: Flash MainsailOS
Use Raspberry Pi Imager — it lets you preconfigure WiFi, hostname, SSH, and user credentials before writing.

1. **Device**: Raspberry Pi 5
2. **OS**: Other specific-purpose OS → 3D printing → MainsailOS
3. **Storage**: Your SD card
4. **Settings** (gear icon before writing):
   - Hostname: `mainsailos-rpi5` (or your preference)
   - Enable SSH: Yes, password auth
   - Username/Password: your credentials
   - WiFi SSID + password
   - Locale: your timezone

### Step 2: First Boot
- Insert SD card, power on Pi 5
- Wait 1-2 minutes for first boot (filesystem resize + WiFi config)
- Find the Pi's IP via your router or `ping mainsailos-rpi5.local`

### Step 3: Configure Passwordless Sudo
```bash
ssh your_user@YOUR_PI_IP
echo 'your_user ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/your_user
sudo chmod 440 /etc/sudoers.d/your_user
```

### Step 4: Update Klipper to Match MCU Firmware
MainsailOS ships with an older Klipper version. Update to match your MCU firmware:
```bash
cd ~/klipper
git pull
```
Then restart Klipper: `sudo systemctl restart klipper`

### Step 5: Copy Config Files
If migrating from another Pi, copy your config files:
```bash
scp user@OLD_PI_IP:~/printer_data/config/*.cfg ~/printer_data/config/
scp user@OLD_PI_IP:~/printer_data/config/*.conf ~/printer_data/config/
```
Or set up fresh using the Primary Configuration Files from this repo.

## Linux Optimizations

These optimizations reduce latency, improve network responsiveness, and give Klipper more resources. All were tested on Pi 5 4GB running MainsailOS 2.2.2.

### Disable Unnecessary Services
```bash
sudo systemctl disable --now ModemManager triggerhappy triggerhappy.socket
sudo systemctl disable --now packagekit
sudo systemctl mask packagekit
```
- **ModemManager**: No cellular modems on a printer controller
- **PackageKit**: Unattended package management, wastes RAM/CPU
- **triggerhappy**: Hotkey daemon, no keyboard attached

### Kernel & Network Tuning
Create `/etc/sysctl.d/99-klipper.conf`:
```bash
sudo tee /etc/sysctl.d/99-klipper.conf > /dev/null << 'EOF'
# Minimize swap usage — 4GB RAM is plenty for Klipper stack
vm.swappiness = 10

# Increase network buffer sizes for Mainsail/Moonraker responsiveness
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.core.netdev_max_backlog = 5000

# Increase TCP buffer sizes
net.ipv4.tcp_rmem = 4096 1048576 16777216
net.ipv4.tcp_wmem = 4096 1048576 16777216

# Enable TCP fast open for faster Mainsail connections
net.ipv4.tcp_fastopen = 3

# Allow Klipper full real-time scheduling (no throttling)
kernel.sched_rt_runtime_us = -1
EOF

sudo sysctl --system
```

### Real-Time Scheduling Limits
Create `/etc/security/limits.d/99-klipper.conf`:
```bash
sudo tee /etc/security/limits.d/99-klipper.conf > /dev/null << 'EOF'
your_user soft rtprio 99
your_user hard rtprio 99
your_user soft memlock unlimited
your_user hard memlock unlimited
your_user soft nice -20
your_user hard nice -20
EOF
```
Replace `your_user` with your actual username.

### USB Low-Latency udev Rule
Create `/etc/udev/rules.d/99-klipper-usb.rules`:
```bash
sudo tee /etc/udev/rules.d/99-klipper-usb.rules > /dev/null << 'EOF'
# Give Klipper USB devices low latency
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{idProduct}=="0002", SYMLINK+="prusa-mk3s", MODE="0666"
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTR{latency_timer}="1"
EOF
```

### USB Autosuspend Fix
Required if using a USBasp programmer, also helps general USB stability:
```bash
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend
```
To make permanent, add `usbcore.autosuspend=-1` to `/boot/firmware/cmdline.txt`.

### Moonraker: Enable Object Processing
The Pi 5 has enough power to enable object processing (allows canceling individual objects mid-print). In `moonraker.conf`:
```ini
[file_manager]
enable_object_processing: True
```
This is NOT recommended on Pi Zero 2W due to limited RAM.

### Reboot
```bash
sudo reboot
```

## Verification

After reboot, verify all optimizations:
```bash
# Services disabled
systemctl is-active ModemManager packagekit triggerhappy
# Should all show "inactive"

# Sysctl applied
sysctl vm.swappiness kernel.sched_rt_runtime_us net.ipv4.tcp_fastopen
# Should show: 10, -1, 3

# Klipper running and connected
systemctl is-active klipper moonraker
# Should both show "active"

# USB stats clean
grep bytes_invalid ~/printer_data/logs/klippy.log | tail -3
# bytes_invalid should be 0 or very low
```

## Results (from actual testing)

| Metric | Pi Zero 2W | Pi 5 |
|--------|-----------|------|
| System load during print | 0.3-0.5 | 0.03-0.07 |
| RAM available during print | ~200MB | ~3.8GB |
| Pi temperature during print | 55-65°C | 45-49°C (with active cooler) |
| USB crash kills WiFi | Yes | No |
| enable_object_processing | Risky | Works fine |
| bytes_invalid (before Hoodserial) | 7,000+/min → crash | 140/sec → no crash |
| bytes_invalid (after Hoodserial) | N/A | 0 |
