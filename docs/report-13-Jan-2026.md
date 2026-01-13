# Status Report - 13 January 2026

## System Overview

| Component | Details |
|-----------|---------|
| Hostname | luxray |
| OS | Ubuntu 22.04, kernel 6.8.0-90-generic |
| CPU | AMD Ryzen 9 5950X 16-Core |
| GPU | NVIDIA RTX 4090 (driver 570.195.03) |
| Motherboard | ASUS ROG CROSSHAIR VIII HERO (WI-FI), BIOS 5302 |
| NICs | Intel I211 (enp6s0), Realtek RTL8125 (enp5s0) |

## Current Power Settings

| Setting | Value |
|---------|-------|
| Auto-suspend (AC) | 2 hours (7200s) |
| Screen blank | 15 min (900s) |

## Issue Status Summary

| Issue | Status | Notes |
|-------|--------|-------|
| GPU hang on screen blank | **FIXED** | Kernel params applied |
| GPU hang after S3 resume | **NOT FIXED** | Crash this morning |
| USB keyboard unresponsive | **FIXED** | udev rule applied |
| Wake-on-LAN from S3 | **NOT POSSIBLE** | Hardware limitation |
| Wake-on-LAN from S5 | **WORKING** | Power-off wake works |

## Crash Analysis - 13 January 2026 (~10:00)

### Timeline

| Time | Event |
|------|-------|
| Jan 12 21:57 | System entered S3 suspend |
| Jan 13 09:24 | System woke from S3 |
| Jan 13 09:52 | GPU WAIT warnings begin |
| Jan 13 09:53:41 | Xid 119: "Timeout after 45s waiting for RPC response from GPU" |
| Jan 13 09:54:26 | Xid 154: "GPU Reset Required" |
| Jan 13 10:02:41 | WAIT escalates to ERROR |
| Jan 13 10:02:53+ | `nvidia-modeset: ERROR: GPU:0: Error while waiting for GPU progress` (repeating every 5s) |
| Jan 13 ~10:07 | User rebooted (hard reset required) |

### Root Cause

The GPU crashed approximately **28 minutes after waking from S3 suspend**. This is NOT the same issue as the screen-blank hang.

**Key finding:** All NVIDIA kernel parameters were correctly applied:
```
nvidia.NVreg_PreserveVideoMemoryAllocations=1
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```

The `nvidia-suspend.service` and `nvidia-resume.service` both completed successfully. However, the GPU entered a corrupted state that manifested as a delayed failure.

### Conclusion

There are **two separate GPU issues**:

1. **Screen blank hang** - Caused by S0ix/Dynamic Power Management putting GPU to sleep during screen blank. **FIXED** with kernel parameters.

2. **S3 resume hang** - GPU fails ~30 minutes after waking from S3 suspend. **NOT FIXED**. This appears to be an NVIDIA driver bug with suspend/resume on RTX 4090.

## Fixes Applied

### GRUB Kernel Parameters
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableS0ixPowerManagement=0 nvidia.NVreg_DynamicPowerManagement=0"
```

### udev Rules
- `/etc/udev/rules.d/85-usb-power.rules` - USB controller power management (prevents keyboard issues)
- `/etc/udev/rules.d/90-keyboard-nosuspend.rules` - Keyboard autosuspend disabled
- `/etc/udev/rules.d/81-pcie-wakeup.rules` - PCIe bridge wakeup (WoL attempt)

### Sleep Hooks
- `/usr/lib/systemd/system-sleep/wol` - Enable WoL before suspend

### NoMachine
- `DisconnectedSessionExpiry 300` in `/usr/NX/etc/server.cfg`

## Recommendations

### Option 1: Disable Auto-Suspend (Recommended)

Given that:
- S3 suspend causes GPU crashes
- WoL from S3 doesn't work anyway (hardware limitation)

The safest approach is to disable auto-suspend entirely:
```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 0
```

Screen blanking will still work (saving monitor power), and the GPU screen-blank fix should prevent hangs during idle.

### Option 2: Keep Current Settings

Keep 2-hour auto-suspend timeout and accept occasional crashes after long sleep periods. Manual wake (keyboard/mouse) should work; just be aware the system may crash ~30 minutes later.

### Option 3: Investigate Further

- Try different NVIDIA driver versions
- Test with `nvidia.NVreg_TemporaryFilePath=/tmp` for different memory preservation behavior
- File bug report with NVIDIA

## Outstanding Items

1. **Decide on suspend policy** - Disable auto-suspend or accept occasional crashes
2. **Sudo NOPASSWD** - Still enabled for debugging, should be reverted when done
3. **Monitoring logs** - Background monitors still running:
   - `~/gpu_monitor.log`
   - `~/dmesg_monitor.log`
   - `~/journal_monitor.log`

## Configuration Verification

All configurations verified intact after BIOS update (checked 13 Jan 2026):

| Configuration | Status | Details |
|--------------|--------|---------|
| GRUB kernel params | ✅ OK | All 3 NVIDIA params in config, grub.cfg, and running kernel |
| udev: USB power (85-usb-power.rules) | ✅ OK | All USB hubs set to `control=on` |
| udev: Keyboard (90-keyboard-nosuspend.rules) | ✅ OK | autosuspend=-1 active |
| udev: PCIe wakeup (81-pcie-wakeup.rules) | ✅ OK | Rules present for both NICs |
| WoL sleep hook | ✅ OK | `/usr/lib/systemd/system-sleep/wol` present and executable |
| NoMachine | ✅ OK | DisconnectedSessionExpiry=300 |
| NVIDIA modprobe | ✅ OK | modeset=1, PreserveVideoMemory, TemporaryFilePath |
| NVIDIA services | ✅ OK | suspend/resume/hibernate/persistenced all enabled |

### NVIDIA Modprobe Settings (`/etc/modprobe.d/nvidia-graphics-drivers-kms.conf`)
```
options nvidia-drm modeset=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_TemporaryFilePath=/var/tmp
```

### Running Kernel Parameters (verified via `/proc/cmdline`)
```
nvidia.NVreg_PreserveVideoMemoryAllocations=1
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```

## GPU Firmware Status

| Component | Version |
|-----------|---------|
| VBIOS | 95.02.3C.40.FB |
| Driver | 570.195.03 |
| Kernel | 6.8.0-90-generic |

NVIDIA's UEFI firmware update tool (v1.2) addresses boot-time blank screens, not suspend/resume issues. The S3 resume crash appears to be a driver-level issue, not firmware.

## Files Modified This Session

- `~/.claude/statusline.sh` - Custom Claude Code status line
- `~/.claude/settings.json` - Status line configuration
- `~/.local/share/fonts/JetBrainsMono/` - Nerd Font installed
- `/etc/modprobe.d/nvidia-graphics-drivers-kms.conf` - Added TemporaryFilePath option
- GNOME power settings - Auto-suspend timeout set to 2 hours
