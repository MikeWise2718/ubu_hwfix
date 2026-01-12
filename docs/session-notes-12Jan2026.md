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

- `/etc/udev/rules.d/85-usb-power.rules` (created - USB controller power)
- `/etc/udev/rules.d/81-pcie-wakeup.rules` (created - PCIe bridge wakeup)
- `/usr/NX/etc/server.cfg` (added DisconnectedSessionExpiry)
- `/usr/lib/systemd/system-sleep/wol` (created - WoL sleep hook)
- GNOME settings (sleep timeout)

## Documentation Updated

- `usb-keyboard-resume-fix.md` - Added USB controller power management fix
- `nvidia-power-management-hang-fix.md` (created) - GPU idle hang fix
- `wake-on-lan-fix.md` - Added S3 sleep hook and alternative wake methods

## Outstanding Items

1. **REBOOT REQUIRED** - NVIDIA power management fix NOT applied until reboot (see crash analysis below)
2. **Sudo NOPASSWD** - Still enabled, remember to revert when done debugging
3. ~~**Test sleep/wake cycle**~~ - Need to re-test after reboot with GPU fix active
4. **WoL from S3** - Not possible (BIOS/ACPI limitation confirmed)
5. **Monitoring logs** - Can be stopped once system is confirmed stable:
   ```bash
   pkill -f "nvidia-smi.*csv"
   sudo pkill -f "dmesg -w"
   pkill -f "journalctl -f"
   ```

## Additional Changes (12 Jan - Later)

### Wake-on-LAN from S3 Suspend

**Issue:** WoL works from power-off (S5) but not from suspend (S3).

**Fix attempted:** Created sleep hook `/usr/lib/systemd/system-sleep/wol` to ensure WoL is set before suspend.

**Status:** May still not work - WoL from S3 is hardware/BIOS dependent. The Intel I211 NIC may not support it.

**Alternatives documented:**
- Disable sleep entirely for remote access
- RTC wake (scheduled)
- Smart plug power cycling

### PCIe Bridge Wakeup Fix (Later Discovery)

**Issue:** WoL still not working from S3 despite sleep hook.

**Root cause found:** Intermediate PCIe bridges (02:00.0 and 03:05.0) had `power/wakeup` disabled, blocking the wake signal from propagating from the NIC to the CPU.

**Fix:** Enable wakeup on PCIe bridges:
```bash
sudo sh -c 'echo enabled > /sys/bus/pci/devices/0000:02:00.0/power/wakeup'
sudo sh -c 'echo enabled > /sys/bus/pci/devices/0000:03:05.0/power/wakeup'
```

**Persistent via:** `/etc/udev/rules.d/81-pcie-wakeup.rules`

**Status:** Did not fix the issue

### Final Diagnosis: BIOS/ACPI Limitation

**Issue:** WoL from S3 still not working after all Linux fixes.

**Root cause:** ACPI wakeup table only exposes S4 (hibernate) wake, not S3 (suspend):
```
GPP1    S4    *enabled   pci:0000:00:01.2
```

The BIOS doesn't advertise S3 wake capability for PCIe devices, so the kernel/driver doesn't enable PME for S3 suspend.

**Solution:** Check ASUS BIOS settings:
- Advanced → APM Configuration → "Power On By PCI-E" → Enable
- "Wake on LAN" → Enable for S3/S4/S5
- "ErP Ready" → Disable
- "Deep Sleep" → Disable

**Status:** Requires BIOS configuration change - cannot be fixed from Linux

### BIOS Update (Later)

**Action:** Updated BIOS from version 4702 (Oct 2023) to 5302 (Oct 2025)

**Result:** No change - ACPI wakeup table still shows S4 only:
```
GPP1    S4    *enabled   pci:0000:00:01.2
```

### Realtek NIC Test (Later)

**Idea:** Try the alternate onboard Realtek RTL8125 2.5GbE NIC instead of Intel I211

**Configuration:**
- Switched cable to Realtek port (enp5s0)
- IP: 192.168.25.198
- MAC: 50:EB:F6:B6:80:35
- Enabled WoL via NetworkManager and ethtool
- Enabled PCIe bridge wakeup (03:03.0 and 05:00.0)
- Updated sleep hook for both NICs

**Result:** Did not work - same ACPI S4-only limitation affects all PCIe devices on this motherboard

### Conclusion

WoL from S3 suspend is **not possible** on the ASUS ROG CROSSHAIR VIII HERO (WI-FI) due to:
- BIOS/ACPI only exposes S4 (hibernate) wake, not S3 (suspend)
- This affects ALL PCIe devices (both NICs tested)
- BIOS update to latest version (5302) did not fix the issue
- This is a motherboard firmware limitation that cannot be worked around from Linux

**Workarounds:**
1. Disable auto-sleep when remote access is needed
2. Use WoL from power-off (S5) which works
3. Increase sleep timeout to give more time before suspend

See: `wake-on-lan-fix.md` (updated)

## Current Boot Command Line

```
BOOT_IMAGE=/boot/vmlinuz-6.8.0-90-generic root=UUID=76365766-626b-4284-ba7b-c6b5a5c26622 ro quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1 vt.handoff=7
```

Note: `EnableS0ixPowerManagement=0` and `DynamicPowerManagement=0` are NOT active until reboot.

## GPU Crash During Dinner (12 Jan ~20:45)

### Incident

User went to dinner for ~1 hour. Expected system to sleep and wake with keyboard. Instead, system became completely unresponsive and required power cycle.

### Log Analysis

The previous boot's journal shows this was **NOT a sleep/wake failure** - it was the **same GPU hang issue**:

```
Jan 12 20:39:17 NVIDIA(0): WAIT (2-S, 17, 0x19fb35, 0x00006844, 0x00006ecc)
...
Jan 12 20:45:07 NVIDIA(GPU-0): WAIT (2, 8, 0x8000, 0x00006844, 0x00007590)  [ERROR]
Jan 12 20:45:19 nvidia-modeset: ERROR: GPU:0: Error while waiting for GPU progress: 0x0000c77d:0 2:0:3224:3216
```

GPU WAIT warnings started at 20:39, escalated to errors at 20:45, and the system became unresponsive. The errors repeated every 5 seconds until power cycle.

### Root Cause

**The NVIDIA power management fix was never applied!**

The GRUB config file (`/etc/default/grub`) had the parameters:
```
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```

But the **running kernel** only had:
```
nvidia.NVreg_PreserveVideoMemoryAllocations=1
```

The critical parameters were in the config but `update-grub` was not run after editing, so the grub.cfg didn't have them.

**Note:** The BIOS update (4702→5302) likely triggered a GRUB regeneration, which would explain why the parameters disappeared from grub.cfg even if update-grub had been run earlier. Always verify `/proc/cmdline` after BIOS updates.

### Fix Applied

```bash
sudo update-grub
```

Verified the parameters are now in `/boot/grub/grub.cfg`.

### Status

**REBOOT REQUIRED** - The GPU hang fix is now in grub.cfg but will only take effect after reboot.

After reboot, verify with:
```bash
cat /proc/cmdline | grep -o 'nvidia\.[A-Za-z_=0-9]*'
```

Expected output:
```
nvidia.NVreg_PreserveVideoMemoryAllocations=1
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```
