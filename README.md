# TUXEDO InfinityBook Gen10 Fan Control

Minimal, silent fan control for TUXEDO InfinityBook Pro Gen10.

> **Hardware Notice:** This project has only been tested on a **TUXEDO InfinityBook Pro AMD Gen10** with a **Ryzen AI 9 HX 370** processor. It may work on other InfinityBook Gen10 variants, but this is untested. Use at your own risk.

## Why?

The stock kernel has no fan control for Uniwill-based laptops. TUXEDO provides their Control Center and custom kernel modules, but Control Center is a heavy Electron app and the tuxedo-drivers caused issues on my system - including the CPU randomly getting stuck at 600MHz.

This project provides just fan control with no other baggage, keeping the rest native:

- **Minimal footprint** - ~17KB daemon + ~400KB kernel module
- **No dependencies** - works standalone without TUXEDO Control Center or tuxedo-drivers
- **Compatible with power-profiles-daemon** - handles only fan control, nothing else
- **Silent by default** - fans stay quiet when idle, ramp smoothly under load

## Recommended Setup

- Stock Arch Linux kernel (no tuxedo-drivers)
- `power-profiles-daemon` for power management
- This project for fan control

## Linux 6.19+ and Upstream Driver Status

**Current State (as of December 2025):**

- Linux 6.19 (currently in RC phase) includes the in-tree `uniwill-laptop` driver
- Upstream driver provides **read-only** monitoring: temperatures, fan RPM, PWM values via hwmon
- Upstream driver does **NOT** provide manual fan control or custom fan curves
- Future plans for upstream driver are unknown - manual control *may* be added later

**This Module's Role:**

- **Linux 6.19+**: Use alongside upstream `uniwill-laptop` driver
  - Read temps from upstream `uniwill` hwmon or other sources (k10temp, amdgpu)
  - Write manual PWM control via this module's separate hwmon device (`uniwill_ibg10_fanctl`)
  - Provides custom fan curves that upstream lacks

- **Linux <6.19**: Works standalone with full hwmon interface (read temps + write PWM)

**Note for Future:** Check upstream `uniwill-laptop` driver capabilities if manual fan control gets added. This module may still provide value through custom curve implementation.

## How It Works

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  User Space                                                                  │
│                                                                              │
│  uniwill_ibg10_fanctl (daemon)                                               │
│      │                                                                       │
│      ├──reads temps──▶ /sys/class/hwmon/ (uniwill→k10temp→EC, amdgpu→EC)     │
│      └──writes PWM──▶ /sys/class/hwmon/.../pwm1 (uniwill_ibg10_fanctl device)│
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                               │ sysfs hwmon
┌──────────────────────────────│───────────────────────────────────────────────┐
│  Kernel                      ▼                                               │
│                    uniwill_ibg10_fanctl.ko                                   │
│                         │                                                    │
│                         │ WMI calls                                          │
│                         ▼                                                    │
│                 ACPI WMI Interface ──▶ Embedded Controller                   │
│                                              │                               │
└──────────────────────────────────────────────│───────────────────────────────┘
                                               ▼
                                     CPU Fan & GPU Fan
```

**The daemon loop:**

1. Read CPU temp (priority: `uniwill` → `k10temp`), GPU temp from `amdgpu`. If both fail, use `uniwill` EC temp
2. Calculate target speed for each fan independently using interpolated fan curve
3. Apply hysteresis (6°C gap prevents oscillation)
4. Write target speed to both fans (unified control - both follow max temp due to shared heatpipes)
5. Sleep 1s

**Fan curve:**

```
Fan %
100 │                                                       *
 75 │                                               *
 50 │                                     *
 25 │                           *
 12 │***************** (minimum, prevents EC fighting)
    └─────────────────┬─────────┬─────────┬─────────┬───────┬───▶ Temp °C
                     62        70        78        86      92
```

## Features

- **Silent fan curve**: Smooth, quiet operation with hysteresis
- **Direct EC control**: Communicates with EC via WMI interface
- **Unified dual fan control**: Both fans follow max temperature (shared heatpipes)
- **Real hwmon integration**: Reads temps from k10temp and amdgpu sensors
- **EC fallback**: Uses EC temperature sensor if hwmon unavailable
- **Systemd service**: Runs automatically on boot
- **No runtime dependencies**: Single binary, only links to libc

## Compatibility

**Tested on:**

- TUXEDO InfinityBook Pro AMD Gen10 (Ryzen AI 9 HX 370)
- Arch Linux, kernel 6.x
- WMI GUID `ABBC0F6F-8EA1-11D1-00A0-C90629100000`

See the "Linux 6.19+ and Upstream Driver Status" section above for kernel compatibility details.

## Installation

### Prerequisites

```bash
sudo pacman -S base-devel linux-headers dkms
```

### Step 1: Build and Test the Module

```bash
git clone https://github.com/timohubois/tuxedo-infinitybook-gen10-fan.git
cd tuxedo-infinitybook-gen10-fan

# Build module + daemon
make

# Load module for testing
sudo make load

# Verify hwmon appears (uniwill_ibg10_fanctl)
ls /sys/class/hwmon/*/name

# Read temp (millidegrees)
cat /sys/class/hwmon/*/temp1_input

# Test manual control (0-255 hwmon scale)
echo 128 | sudo tee /sys/class/hwmon/*/pwm1  # ~50%
echo 2   | sudo tee /sys/class/hwmon/*/pwm1_enable  # Restore auto
```

If this works, proceed to step 2.

### Step 2: Test the Daemon

```bash
# Run daemon manually (Ctrl+C to stop)
sudo ./daemon/uniwill_ibg10_fanctl
```

You should see temperature and fan speed updating. Run `./daemon/uniwill_ibg10_fanctl -h` for help. If this works, proceed to step 3.

### Step 3: Install Permanently

#### Option A: DKMS (Recommended)

DKMS automatically rebuilds the module when you update your kernel:

```bash
sudo make uniwill-ibg10-fanctl-install-dkms
```

This installs and starts everything automatically.

#### Option B: Manual Installation

Without DKMS, you'll need to rebuild manually after kernel updates:

```bash
sudo make install-all
```

This installs and starts everything automatically.

### Manual Installation (Step-by-Step)

If you prefer step-by-step:

```bash
# Install module via DKMS (or use 'make install' for non-DKMS)
sudo make uniwill-ibg10-fanctl-install-dkms

# Auto-load on boot
sudo make install-autoload

# Install, enable, and start service (daemon built in ./daemon)
sudo make install-service
```

## Usage

### Manual Control

**Note:** Stop the daemon first if it's running: `sudo systemctl stop uniwill-ibg10-fanctl.service`

```bash
# Load module
sudo modprobe uniwill_ibg10_fanctl

# Find the hwmon device (the number varies, e.g., hwmon11)
HWMON_DEV=$(grep -l uniwill_ibg10_fanctl /sys/class/hwmon/*/name | sed 's|/name||')

# Check current values
cat $HWMON_DEV/temp1_input  # EC temp (millidegrees C, divide by 1000)
cat $HWMON_DEV/pwm1         # Fan1 PWM (0-255 scale)
cat $HWMON_DEV/pwm2         # Fan2 PWM (0-255 scale)
cat $HWMON_DEV/pwm1_enable  # 1=manual, 2=auto

# Set manual mode and control fan speed (0-255 hwmon scale)
echo 1 | sudo tee $HWMON_DEV/pwm1_enable   # Switch to manual mode
echo 128 | sudo tee $HWMON_DEV/pwm1        # Set fan1 to ~50%
echo 128 | sudo tee $HWMON_DEV/pwm2        # Set fan2 to ~50%

# Restore automatic mode (EC control)
echo 2 | sudo tee $HWMON_DEV/pwm1_enable
echo 2 | sudo tee $HWMON_DEV/pwm2_enable

# Restart daemon when done
sudo systemctl start uniwill-ibg10-fanctl.service
```

### Fan Curve Daemon

```bash
# Run manually (interactive mode with status display)
sudo ./daemon/uniwill_ibg10_fanctl

# Show help and configuration
./daemon/uniwill_ibg10_fanctl -h

# Or use the systemd service
sudo systemctl start uniwill-ibg10-fanctl.service
sudo systemctl status uniwill-ibg10-fanctl.service
```

### Configuration

The fan curve thresholds are compiled into the binary. To customize, edit `daemon/uniwill_ibg10_fanctl.c` and rebuild:

```c
/* Temperature thresholds (C) */
#define TEMP_SILENT     62      /* Below this: minimum speed */
#define TEMP_LOW        70      /* Start of low speed */
#define TEMP_MED        78      /* Medium speed */
#define TEMP_HIGH       86      /* High speed */
#define TEMP_MAX        92      /* Maximum speed */

/* Fan speeds (0-200) */
#define SPEED_MIN       25      /* 12.5% - minimum to prevent EC fighting */
#define SPEED_LOW       50      /* 25% */
#define SPEED_MED       100     /* 50% */
#define SPEED_HIGH      150     /* 75% */
#define SPEED_MAX       200     /* 100% */
```

> **Note:** `SPEED_MIN` is 25 because values below 25 cause the EC's safety logic to periodically override the fan speed, resulting in annoying start/stop cycling.

Then rebuild and reinstall:

```bash
make daemon
sudo make install-service
```

## Uninstallation

### DKMS

```bash
sudo make uniwill-ibg10-fanctl-uninstall-dkms
```

### Non-DKMS

```bash
sudo make uninstall-all
```

Or manually:

```bash
sudo systemctl disable --now uniwill-ibg10-fanctl.service
sudo make uninstall-service
sudo make uninstall-autoload
sudo make uninstall
```

## Troubleshooting

### Module won't load

Check if WMI interface exists:

```bash
ls /sys/bus/wmi/devices/ | grep ABBC0F6
```

Check kernel logs:

```bash
dmesg | grep uniwill_ibg10_fanctl
```

### Fans not responding

Verify the hwmon interface exists:

```bash
# Find the hwmon device for this module
ls /sys/class/hwmon/*/name | xargs grep -l uniwill_ibg10_fanctl

# Read current PWM value
HWMON_DEV=$(grep -l uniwill_ibg10_fanctl /sys/class/hwmon/*/name | sed 's|/name||')
cat $HWMON_DEV/pwm1
```

### Service not starting

Check if module is loaded:

```bash
lsmod | grep uniwill_ibg10_fanctl
```

Check service status:

```bash
sudo systemctl status uniwill-ibg10-fanctl.service
sudo journalctl -u uniwill-ibg10-fanctl.service
```

### Fan never fully stops

This is intentional. The daemon keeps the fan at a minimum of 12.5% to prevent the EC from fighting for control, which would cause annoying start/stop cycling. The minimum speed is barely audible.

## Technical Details

### Hwmon Interface (0-255 PWM)

| Path (example) | Access | Description |
|------|--------|-------------|
| `/sys/class/hwmon/hwmonX/name` | RO | Should read `uniwill_ibg10_fanctl` |
| `/sys/class/hwmon/hwmonX/temp1_input` | RO | EC CPU temperature (millidegrees) |
| `/sys/class/hwmon/hwmonX/pwm1` | RW | Fan1 PWM (0-255) |
| `/sys/class/hwmon/hwmonX/pwm2` | RW | Fan2 PWM (0-255) |
| `/sys/class/hwmon/hwmonX/pwm1_enable` | RW | 1=manual, 2=auto |
| `/sys/class/hwmon/hwmonX/pwm2_enable` | RW | 1=manual, 2=auto |

### EC Registers

The module uses the Uniwill WMI interface to communicate with the EC:

- Custom fan table: `0x0f00-0x0f5f`
- Direct fan control: `0x1804`, `0x1809`
- Fan mode: `0x0751`
- Custom profile mode: `0x0727` (bit 6) - required for IBP Gen10
- Manual mode: `0x0741`
- Custom fan table enable: `0x07c5` (bit 7)

### Fan Speed Values

The hwmon-visible fan speed range is 0-255 (mapped to EC 0-200 internally):

| Value | Behavior |
|-------|----------|
| 0 | Fan off (internally clamped to safe minimum) |
| 1-31 | Clamped to ~12.5% (minimum running speed) |
| 32-255 | Direct PWM control (mapped to EC scale) |

## License

GPL-2.0+

## Credits

- Based on reverse engineering of [tuxedo-drivers](https://github.com/tuxedocomputers/tuxedo-drivers)
- WMI interface discovery from TUXEDO Control Center
