# Ubuntu Hardware Fix Project

This project documents fixes and configuration for "Luxray", an Ubuntu 22.04 workstation with sleep/wake, power management, and remote access issues.

## System Details

- **Hostname:** luxray
- **OS:** Ubuntu 22.04, kernel 6.8.0-90-generic
- **CPU:** AMD Ryzen 9 5950X 16-Core
- **GPU:** NVIDIA RTX 4090 (driver 570.195.03)
- **Motherboard:** ASUS ROG CROSSHAIR VIII HERO (WI-FI), BIOS 5302
- **NICs:** Intel I211 (enp6s0), Realtek RTL8125

## Documentation

All fixes and session notes are in `/docs/`:

| File | Description |
|------|-------------|
| `nvidia-power-management-hang-fix.md` | **Critical** - GPU hangs on screen blank |
| `nvidia-suspend-fix.md` | NVIDIA suspend/resume video memory preservation |
| `usb-keyboard-resume-fix.md` | Keyboard unresponsive after idle/suspend |
| `wake-on-lan-fix.md` | WoL configuration and S3 limitations |
| `crash-monitoring-setup.md` | Background monitoring for diagnosing crashes |
| `potential_issues_10Jan2026.md` | Known issues (ACPI, AMD P-State, thermals) |
| `session-notes-12Jan2026.md` | Detailed session log with all changes |

## Key Fixes Applied

### GRUB Kernel Parameters
```
nvidia.NVreg_PreserveVideoMemoryAllocations=1
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```

### udev Rules
- `/etc/udev/rules.d/85-usb-power.rules` - USB controller power management
- `/etc/udev/rules.d/90-keyboard-nosuspend.rules` - Keyboard autosuspend

### Sleep Hooks
- `/usr/lib/systemd/system-sleep/wol` - WoL before suspend

### NoMachine
- `DisconnectedSessionExpiry 300` in `/usr/NX/etc/server.cfg`

## Current Power Settings

| Setting | Value |
|---------|-------|
| Auto-suspend (AC) | 30 min (testing) |
| Screen blank | 15 min |

## Outstanding Items

- **Sudo NOPASSWD** - Enabled for debugging, should be reverted when done
- **WoL from S3** - May not work (hardware dependent)
- **Monitoring logs** - `~/gpu_monitor.log`, `~/dmesg_monitor.log`, `~/journal_monitor.log`

## When Resuming Work

1. Read `docs/session-notes-12Jan2026.md` for latest status
2. Check if any new crashes occurred (review monitoring logs)
3. Remember sudo NOPASSWD is still enabled
