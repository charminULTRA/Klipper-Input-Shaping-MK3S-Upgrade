# 32u2 Hoodserial Flash Procedure — Einsy RAMBo (Prusa MK3S+)

## TESTED AND CONFIRMED 2026-04-01

## Why
The ATmega 32u2 USB-to-serial bridge has defective Prusa firmware that corrupts serial data under Klipper's throughput. Hoodserial (by Arksine/Moonraker author) replaces it with optimized hand-written AVR assembly ISRs.

**Result: bytes_invalid went from 468,000+ per print (crashing) to ZERO.**

## Repository
https://github.com/PrusaOwners/mk3-32u2-firmware

## Hardware Used
- HiLetgo USBasp USB ISP Programmer (Amazon B00AX4WQ00) — $7.99
- Comimark 10-pin to 6-pin adapter (Amazon B07WZF6Z27) — $5.99
- Raspberry Pi 5 running MainsailOS

## CRITICAL LESSONS LEARNED (from actual experience)

### USBasp Jumpers (HiLetgo B00AX4WQ00)
- Board has **JP1** (2 pins) and **JP21** (3 pins, labeled 3.3V/5V on back of PCB)
- **REMOVE ALL JUMPERS** — the USBasp works without any jumpers installed
- VERIFIED: Jumpering JP21 kills the programmer (red light goes off, USB device disappears from lsusb)
- VERIFIED: Jumpering JP1 also causes issues
- Without jumpers, the USBasp does NOT supply VCC to the target — printer PSU must power the board instead

### 10-to-6 Pin Adapter Orientation (Comimark B07WZF6Z27)
- The 6-pin connector can plug into the Einsy header in 2 orientations — **only one works**
- VERIFIED: Wrong orientation reads `device signature = 0x000000` — chip appears dead
- VERIFIED: Correct orientation reads `device signature = 0x1e958a (probably m32u2)` instantly
- If you get `0x000000`, **flip the 6-pin connector 180°** on the Einsy header and retry
- Both orientations were tested — one gives 0x000000, the other works immediately

### USB Timeout Issue on Pi 5
- The USBasp drops USB connection after ~6 seconds of sustained transfer on Pi 5
- VERIFIED FIX: `echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend` — disabling USB autosuspend allowed the 7.4KB flash write to complete in 1.26 seconds
- Without this fix: flash write died at 13%, 28%, 43% on successive attempts
- With this fix: flash write completed 100% on first attempt
- Timeout errors AFTER 100% write are from the verify step and are harmless — the write succeeded
- Fuse writes (1 byte each) work reliably without this fix
- EEPROM backup (1KB) also completed successfully without this fix
- Unplug/replug USBasp between failed attempts to reset USB connection

### Power Configuration
- VERIFIED: Printer PSU must be ON (the USBasp doesn't supply VCC without jumpers)
- VERIFIED: Printer USB cable must be DISCONNECTED from Pi during programming (only USBasp USB connected)
- The README says to disconnect PSU, but that assumes USBasp supplies VCC via jumper — ours doesn't

## Wiring
USBasp → 10-pin ribbon cable → 10-to-6 adapter → Einsy 32u2 ICSP header

The 32u2 ICSP is the 6-pin header near the USB port on the Einsy board (between Reset button and J19). There is only one 6-pin ICSP header on the board.

## Pre-Flash Checklist
- [ ] Printer USB cable DISCONNECTED from Pi
- [ ] Printer PSU ON (powers the 32u2)
- [ ] All USBasp jumpers REMOVED
- [ ] USBasp → ribbon cable → 10-to-6 adapter → Einsy 32u2 ICSP
- [ ] 6-pin adapter oriented correctly (test with avrdude, flip if 0x000000)
- [ ] USBasp USB cable plugged into Pi
- [ ] Red light on USBasp
- [ ] avrdude installed: `sudo apt install avrdude`
- [ ] Disable USB autosuspend: `echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend`

## Step-by-Step (TESTED)

### Step 1: Verify communication
```bash
sudo avrdude -p m32u2 -F -c usbasp -P usb
```
Must show: `device signature = 0x1e958a (probably m32u2)`
If `0x000000` — flip the 6-pin connector 180° and retry.

### Step 2: Backup fuses and EEPROM
```bash
mkdir -p ~/mk3_32u2_backup && cd ~/mk3_32u2_backup
sudo avrdude -p m32u2 -F -c usbasp -P usb \
  -U lfuse:r:lowfuse:h -U hfuse:r:highfuse:h \
  -U efuse:r:exfuse:h -U lock:r:lockfuse:h
sudo avrdude -p m32u2 -F -c usbasp -P usb \
  -U eeprom:r:eeprom.hex:i
```
Note: Flash backup may timeout on HiLetgo USBasp. Fuses and EEPROM are what matter (EEPROM has Prusa serial number). If flash read times out, skip it — it's the defective firmware we're replacing.

### Step 3: Set high fuse to preserve EEPROM
```bash
sudo avrdude -p m32u2 -F -c usbasp -P usb -U hfuse:w:0xD1:m
```

### Step 4: Flash Hoodserial
```bash
wget https://raw.githubusercontent.com/PrusaOwners/mk3-32u2-firmware/master/hex_files/DFU-hoodserial-combined-PrusaMK3-32u2.hex -O /tmp/DFU-hoodserial-combined-PrusaMK3-32u2.hex

# First attempt — includes chip erase:
sudo avrdude -p m32u2 -F -c usbasp -P usb \
  -U flash:w:/tmp/DFU-hoodserial-combined-PrusaMK3-32u2.hex

# If it times out before 100%, unplug/replug USBasp and retry WITH -D flag
# (chip was already erased on first attempt, -D skips re-erase):
sudo avrdude -p m32u2 -F -c usbasp -P usb \
  -D -U flash:w:/tmp/DFU-hoodserial-combined-PrusaMK3-32u2.hex
```
Keep retrying until you see `Writing | ##### 100%`. Timeout errors AFTER 100% are harmless.

### Step 5: Set final fuses
```bash
sudo avrdude -p m32u2 -F -c usbasp -P usb \
  -U lfuse:w:0xFF:m -U hfuse:w:0xD9:m -U efuse:w:0xF4:m
```

### Step 6: Verify
- Remove USBasp and adapter from Einsy
- Connect printer USB to Pi
- Power cycle printer (unplug PSU, wait 5 sec, plug back in)
```bash
lsusb  # Should show: 2c99:0002 Prusa Original Prusa i3 MK3
ls /dev/serial/by-id/  # Should show serial number preserved
```

### Step 7: Flash Klipper back to ATmega2560
```bash
cd ~/klipper
# Use YOUR serial path from Step 6:
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Prusa_Research__prusa3d.com__Original_Prusa_i3_MK3_YOURSERIALNUMBER-if00
sudo systemctl enable klipper
sudo systemctl restart klipper
```
Note: The Klipper build config (.config) must already exist for your board (ATmega2560, 16MHz, UART0). If `make flash` complains about missing config, run `make menuconfig` first.

### Step 8: Verify clean USB
```bash
grep bytes_invalid ~/printer_data/logs/klippy.log | tail -5
```
bytes_invalid should be 0. bytes_retransmit should be 0 or near-0.

## Fuse Values

| Fuse  | Before (stock) | After (Hoodserial) | Purpose |
|-------|---------------|-------------------|---------|
| lfuse | 0xEF          | 0xFF              | External crystal, no clock division |
| hfuse | 0xD9          | 0xD1 → 0xD9      | 0xD1 set BEFORE flash to preserve EEPROM, 0xD9 restored AFTER |
| efuse | 0xF4          | 0xF4              | HWBE enabled, BOD 2.6V (unchanged) |
| lock  | 0xCF          | not changed       | Lock bits preserved |

## Results (VERIFIED from actual Klipper logs)

| Metric | Before (Prusa 32u2 FW) | After (Hoodserial) | Source |
|--------|------------------------|--------------------|--------|
| bytes_invalid after 75min print | 468,627 (Pi 5) / 55-60k crash (Pi Zero 2W) | 0 | klippy.log stats lines |
| bytes_retransmit after 75min | 30,026 | 9 (startup only, not climbing) | klippy.log stats lines |
| bytes_write after 80min | 8.4MB+ | 8.4MB+ (same throughput, zero errors) | klippy.log stats lines |
| USB crashes | Every print on Pi Zero 2W, eventually on Pi 5 | None | Direct observation |
| Klipper MCU flash via USB | Not tested with old FW | Worked perfectly (41KB written + verified, no errors) | avrdude output |
| Serial number preserved | N/A | Yes: CZPX5121X004XK31157 | lsusb + /dev/serial/by-id/ |
| USB VID:PID | 2c99:0002 | 2c99:0002 (unchanged) | lsusb |

## Hardware Details
- **Printer:** Prusa MK3S+ (Einsy RAMBo 1.1a, serial CZPX5121X004XK31157)
- **Pi:** Raspberry Pi 5 4GB running MainsailOS 2.2.2
- **Klipper:** v0.13.0-593-g2f05309d5
- **USBasp:** HiLetgo B00AX4WQ00 (ATmega8A-AU based clone)
- **Adapter:** Comimark 10-pin to 6-pin ISP adapter (B07WZF6Z27)
- **avrdude:** Version 7.1

## Sources
- Hoodserial firmware and flash commands: https://github.com/PrusaOwners/mk3-32u2-firmware/blob/master/README.md
- MK3S compatibility: repo Issue #2 — Arksine confirmed: "Yes, it will work on the MK3S, as it uses the same Einsy Rambo that the MK3 did."
- Power cycle after flash: repo Issue #1 — user noradtux confirmed unpowering the 32u2 resolved post-flash issues
- USBasp jumper info: fischl.de/usbasp/Readme.txt, Protostack USBasp V2.0 User Guide
- Einsy board diagram: repo images/einsy_uno.png, reprap.org/wiki/EinsyRambo
- USB autosuspend fix: tested empirically — flash write went from failing at 13-43% to completing 100% in 1.26s
