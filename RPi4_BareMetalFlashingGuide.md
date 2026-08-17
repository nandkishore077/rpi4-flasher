# Bare-Metal Flashing Guide — Raspberry Pi 4


---

## What is Bare-Metal?

Bare-metal means your code runs **directly on the hardware** with no
operating system. There is no Linux, no RTOS, no drivers — just your
code and the hardware registers. The Raspberry Pi 4 boots from an SD
card and runs whatever binary is placed there as `kernel8.img`.

---

## What You Need

### Hardware
- Raspberry Pi 4 Model B (any RAM variant)
- MicroSD card (8GB or more, Class 10 recommended)
- USB-UART adapter (CP2102 or CH340 based, 3.3V logic)
- Micro-USB or USB-C power supply (5V 3A)
- Jumper wires

### Software (on your Linux development machine)
- `aarch64-none-elf-gcc` — ARM64 bare-metal GCC toolchain
- `make` — build system
- `screen` or `minicom` — serial terminal
- SD card with RPi4 firmware files (boot partition)

---

## Step 1 — Install the Toolchain

```bash
# Download ARM GNU Toolchain (AArch64 bare-metal)
# From: https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
# Choose: aarch64-none-elf (not linux-gnu)

# Extract and add to PATH
tar -xf arm-gnu-toolchain-*.tar.xz
export PATH=$PATH:/path/to/arm-gnu-toolchain/bin

# Verify
aarch64-none-elf-gcc --version
```

---

## Step 2 — Prepare the SD Card

The RPi4 needs firmware files on the SD card boot partition to start up.
These files are provided by the Raspberry Pi Foundation.

### Option A — Use an existing RPi4 SD card
If you already have a Raspberry Pi OS SD card, the boot partition
already has all required firmware files. You just replace `kernel8.img`.

### Option B — Fresh SD card setup
```bash
# Format SD card with FAT32 boot partition (256MB minimum)
# Then copy these files to the boot partition:
#   bootcode.bin   (not needed on RPi4, but harmless)
#   start4.elf     (GPU firmware — REQUIRED)
#   fixup4.dat     (GPU firmware fix-up — REQUIRED)
#   config.txt     (boot configuration)

# Minimum config.txt for bare-metal AArch64:
cat > config.txt << 'EOF'
arm_64bit=1
kernel=kernel8.img
core_freq=500
EOF
```

Download firmware files from:
`https://github.com/raspberrypi/firmware/tree/master/boot`

Files needed: `start4.elf`, `fixup4.dat`

---

## Step 3 — Project File Structure

Every bare-metal project for RPi4 needs these files:

```
your-project/
├── start.S        — CPU startup code (REQUIRED — provided by your project)
├── link.ld        — Linker script (REQUIRED — provided by your project)
├── Makefile       — Build rules (REQUIRED — provided by your project)
├── main.c         — Your application code
├── your-driver.h  — Your driver header
├── your-driver.c  — Your driver implementation
├── gpio.h         — GPIO register definitions
└── uart.h         — UART debug output
```

---

## Step 4 — Build Your Code

```bash
# Go to your project folder
cd your-project/

# Clean previous build and compile
make clean && make

# If successful you will see:
# === Build Complete ===
#    text    data     bss     dec     hex filename
#   XXXXX       0      XX   XXXXX    XXXX kernel8.elf
# Output: kernel8.img (XXXXX bytes)
```

If there are errors, fix them before proceeding. Do not flash a broken build.


## Step 5 — Flash the SD Card

```bash
# Insert SD card into your laptop

# Find the SD card device
lsblk | grep -v loop

# Look for a disk with a ~256MB partition (the boot partition)
# Example output:
# sdb       29.1G  ← this is the SD card
# ├─sdb1   512M   /media/user/bootfs1   ← boot partition (FAT32)
# └─sdb2   5.5G   /media/user/rootfs    ← root partition

# Copy your kernel to the boot partition
sudo cp kernel8.img /media/$USER/bootfs1/kernel8.img && sync

# Eject safely
sudo eject /dev/sdb
```

---

## Step 6 — Boot and Monitor

```bash
# Connect USB-UART adapter to laptop
# Find the serial port device
ls /dev/ttyUSB*
# or
ls /dev/ttyACM*

# Open serial terminal at 115200 baud
screen /dev/ttyUSB0 115200
# or
minicom -D /dev/ttyUSB0 -b 115200
```

Then:
1. Insert the flashed SD card into RPi4
2. Power on the RPi4
3. You should see your program output in the serial terminal

To exit `screen`: press `Ctrl+A` then `K` then `Y`
To exit `minicom`: press `Ctrl+A` then `X`

---

## Step 7 — Iterate

To update your code and reflash:

```bash
# 1. Edit your source files
# 2. Rebuild
make clean && make

# 3. Insert SD card into laptop
# 4. Flash
sudo cp kernel8.img /media/$USER/bootfs1/kernel8.img && sync
sudo eject /dev/sdb

# 5. Insert SD card into RPi4 and power on
```

---

## Common Issues and Fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| Nothing prints on serial | UART not initialised or wrong baud | Check baud is 115200, check TX/RX wiring |
| Garbled serial output | Wrong baud rate | Set serial terminal to 115200 |
| RPi4 does not boot | Missing `start4.elf` or `fixup4.dat` | Copy firmware files to SD card |
| Build fails: undefined reference | Wrong or missing header file | Check all `.h` files are in project folder |
| Build fails: no such file | Wrong filename in Makefile | Check Makefile matches your actual `.c` filenames |
| Code hangs immediately | Stack not set up or BSS not zeroed | Check `start.S` sets SP and zeros BSS before `main()` |
| Code hangs on register write | Wrong base address (VPU vs ARM) | Use `0xFE...` not `0x7E...` for BCM2711 |
| Division causes hang | `-nostdlib` has no `__udivsi3` | Replace `/` and `%` with bit-shifts or lookup tables |
| `printf` causes hang | `va_list` needs 16-byte stack alignment | Use non-variadic print functions instead |

---

## BCM2711 Key Addresses (RPi4)

| Peripheral | ARM Physical Base | Notes |
|-----------|-------------------|-------|
| GPIO       | 0xFE200000        | Function select, set, clear, pull |
| UART0 (PL011) | 0xFE201000    | Debug serial output |
| SPI0       | 0xFE204000        | Primary SPI bus |
| BSC0 (I2C) | 0xFE205000        | GPIO 0/1 — reserved for HAT EEPROM |
| BSC1 (I2C) | 0xFE804000        | GPIO 2/3 — primary user I2C bus |
| GIC-400    | 0xFF840000        | Interrupt controller |
| ARM Timer  | 0xFE00B400        | System timer |

**Rule:** Always use `0xFE...` (ARM physical). Never use `0x7E...` (VPU bus).

---

## Core Clock Reference

| Clock | Frequency | How to verify |
|-------|-----------|---------------|
| CPU (ARM) | 1500 MHz | `vcgencmd measure_clock arm` |
| Core (VPU) | 500 MHz | `vcgencmd measure_clock core` |
| UART | 48 MHz | `cat /sys/kernel/debug/clk/uart0/clk_rate` |
| EMMC | varies | `vcgencmd measure_clock emmc` |

Run these commands on a Linux RPi4 to verify clocks before writing driver code.

---

## Minimum config.txt for Bare-Metal AArch64

```ini
# Bare-metal AArch64 configuration
arm_64bit=1
kernel=kernel8.img
core_freq=500
enable_uart=1
```

Place this file in the SD card boot partition alongside `kernel8.img`.

---

## Quick Reference — Full Flash Sequence

```bash
# One-liner: build and flash
make clean && make && \
sudo cp kernel8.img /media/$USER/bootfs1/kernel8.img && \
sync && \
sudo eject /dev/sdb && \
echo "Ready — insert SD card into RPi4 and power on"
```

---

*Document prepared for internal trainee use.*
*Hardware: Raspberry Pi 4 Model B, BCM2711 SoC.*
*Toolchain: arm-gnu-toolchain aarch64-none-elf.*
