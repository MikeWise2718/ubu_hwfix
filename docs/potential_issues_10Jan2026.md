# Potential Issues Identified - 10 January 2026

## System Configuration

- **CPU:** AMD Ryzen 9 5950X 16-Core Processor
- **GPU:** NVIDIA RTX 4090 (driver 570.195.03)
- **Motherboard:** ASUS (ALASKA BIOS)
- **OS:** Ubuntu 22.04, kernel 6.8.0-90-generic

## Issues Found in dmesg

### 1. AMD P-State Driver Disabled

```
amd_pstate: driver load is disabled, boot with specific mode to enable this
```

**Status:** Using older `acpi-cpufreq` driver instead of `amd_pstate`

**Impact:** Suboptimal power management, not a crash cause

**Fix (optional):** Add `amd_pstate=active` to GRUB kernel parameters

### 2. ACPI _OSC Evaluation Failed

```
ACPI: _OSC evaluation for CPUs failed, trying _PDC
```

**Impact:** CPU power management falls back to legacy _PDC method. Common on ASUS/AMD boards, usually harmless.

**Fix:** None required, system falls back gracefully

### 3. Thermal Zone Read Failure

```
thermal thermal_zone0: failed to read out thermal zone (-61)
```

**Impact:** Kernel thermal zone failed to initialize, but hardware monitoring still works via `k10temp` and `asusec` modules.

**Fix:** None required, `sensors` command works correctly

## Current Thermal Readings (at time of check)

| Component | Temperature |
|-----------|-------------|
| CPU (Tctl) | 39°C |
| CPU CCD1 | 42°C |
| CPU CCD2 | 37°C |
| Chipset | 63°C |
| GPU | 36°C |
| NVMe #1 | 37°C |
| NVMe #2 | 47°C |
| VRM | 26°C |

All temperatures normal.

## Sudo Configuration Change (REVERT LATER)

To enable Claude Code to run diagnostic commands with sudo, passwordless sudo was enabled:

```bash
sudo visudo
```

Added line:
```
mike ALL=(ALL) NOPASSWD: ALL
```

### To Revert (recommended after debugging complete)

```bash
sudo visudo
```

Remove or comment out the line:
```
# mike ALL=(ALL) NOPASSWD: ALL
```

Or for more security, limit to specific commands only:
```
mike ALL=(ALL) NOPASSWD: /usr/bin/dmesg, /usr/sbin/update-grub
```

## Related Documentation

- `nvidia-suspend-fix.md` - NVIDIA PreserveVideoMemoryAllocations fix (applied)
- `crash-monitoring-setup.md` - Background monitoring for crash diagnosis
- `usb-keyboard-resume-fix.md` - USB keyboard wake fix
- `wake-on-lan-fix.md` - WoL NetworkManager fix

## Next Steps

1. Monitor system stability with NVIDIA fix applied
2. Check `~/gpu_monitor.log`, `~/dmesg_monitor.log`, `~/journal_monitor.log` if crashes recur
3. Consider enabling `amd_pstate=active` after stability confirmed
4. Revert sudo NOPASSWD when debugging complete
