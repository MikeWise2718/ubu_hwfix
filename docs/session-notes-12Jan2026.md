# Session Notes - 12 January 2026

## Summary

Continued troubleshooting system hangs and keyboard unresponsiveness. Identified GPU power management as the root cause of hangs, and USB controller power management as cause of keyboard issues after idle.

## Issues Addressed Today

### 1. GPU Hang on Screen Blank (DIAGNOSED)

**Symptom:** System became completely unresponsive after ~20 minutes idle. Monitoring logs showed:
```
nvidia-modeset: ERROR: GPU:0: Error while waiting for GPU progress: 0x0000c77d:0 2:0:3072:3064
```

**Root Cause:** NVIDIA S0ix and Dynamic Power Management putting GPU to sleep when screen blanks; GPU fails to wake.

**Fix Applied:** Added kernel parameters to `/etc/default/grub`:
```
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```

**Status:** In GRUB config, requires reboot to take effect.

**Trade-off:** ~15-25W higher idle power consumption.

See: `nvidia-power-management-hang-fix.md`

### 2. USB Keyboard Unresponsive After Idle (FIXED)

**Symptom:** Keyboard stopped responding after system idle overnight, even though system didn't crash.

**Root Cause:** USB root hub/controller entering power-saving mode (all set to `auto`).

**Fix Applied:** Created udev rule `/etc/udev/rules.d/85-usb-power.rules`:
```
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1d6b", ATTR{power/control}="on"
```

**Status:** Active immediately (applied live).

See: `usb-keyboard-resume-fix.md` (updated)

### 3. NoMachine Session Persistence

**Observation:** NoMachine sessions persist after client disconnects, keeping system "active" and preventing idle/sleep.

**Fix Applied:** Set disconnected sessions to expire after 5 minutes:
```
DisconnectedSessionExpiry 300
```
Added to `/usr/NX/etc/server.cfg`

**Note:** `nxserver --restart` disrupts active sessions and may restart display manager.

### 4. Sleep Timeout Adjusted for Testing

**Change:** Reduced auto-suspend timeout from 2 hours to 30 minutes for testing.

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 1800
```

## Current System Settings

### Power Management

| Setting | Value | Location |
|---------|-------|----------|
| Auto-suspend (AC) | 30 min (1800s) | GNOME settings |
| Screen blank | 15 min (900s) | GNOME settings |
| Auto-suspend (battery) | 20 min (1200s) | GNOME settings |

### NVIDIA Parameters (in GRUB, pending reboot)

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableS0ixPowerManagement=0 nvidia.NVreg_DynamicPowerManagement=0"
```

### USB Power Management

| Device | Setting | Status |
|--------|---------|--------|
| Keyboard (1a2c:2124) | autosuspend=-1 | Active |
| USB root hubs (1d6b) | control=on | Active |

### NoMachine

| Setting | Value |
|---------|-------|
| DisconnectedSessionExpiry | 300 seconds (5 min) |

## Files Modified Today

- `/etc/udev/rules.d/85-usb-power.rules` (created)
- `/usr/NX/etc/server.cfg` (added DisconnectedSessionExpiry)
- GNOME settings (sleep timeout)

## Documentation Updated

- `usb-keyboard-resume-fix.md` - Added USB controller power management fix
- `nvidia-power-management-hang-fix.md` (created) - GPU idle hang fix

## Outstanding Items

1. **Reboot required** - NVIDIA power management parameters are in GRUB but not active in current boot
2. **Sudo NOPASSWD** - Still enabled, remember to revert when done debugging
3. **Test sleep/wake cycle** - After reboot, verify system sleeps and wakes correctly
4. **Monitoring logs** - Can be stopped once system is confirmed stable:
   ```bash
   pkill -f "nvidia-smi.*csv"
   sudo pkill -f "dmesg -w"
   pkill -f "journalctl -f"
   ```

## Current Boot Command Line

```
BOOT_IMAGE=/boot/vmlinuz-6.8.0-90-generic root=UUID=76365766-626b-4284-ba7b-c6b5a5c26622 ro quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1 vt.handoff=7
```

Note: `EnableS0ixPowerManagement=0` and `DynamicPowerManagement=0` are NOT active until reboot.
