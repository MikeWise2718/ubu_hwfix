# NVIDIA GPU Power Management Hang Fix

## Problem

System becomes completely unresponsive after idle period (~15-20 minutes). Keyboard unresponsive, cannot wake display, cannot connect via NoMachine. Requires hard reboot.

## Symptoms

- System works fine while actively in use
- Hangs occur after screen blanks due to idle timeout
- dmesg shows repeating errors:
  ```
  nvidia-modeset: ERROR: GPU:0: Error while waiting for GPU progress: 0x0000c77d:0 2:0:3072:3064
  ```
- Errors repeat every 5 seconds until hard reboot

## Cause

NVIDIA's power management features (S0ix and Dynamic Power Management) cause the GPU to enter a low-power state when the display blanks. The GPU fails to wake from this state, causing a complete system hang.

Relevant NVIDIA driver parameters before fix:
```
EnableS0ixPowerManagement: 1        # Modern standby - PROBLEMATIC
DynamicPowerManagement: 3           # Dynamic PM level 3 - PROBLEMATIC
```

## Affected Hardware

- **GPU:** NVIDIA RTX 4090
- **Driver:** 570.195.03
- **CPU:** AMD Ryzen 9 5950X
- **OS:** Ubuntu 22.04, kernel 6.8.0-90-generic

## Solution

Disable NVIDIA S0ix and Dynamic Power Management via kernel parameters.

### Apply the fix

```bash
sudo sed -i 's/nvidia.NVreg_PreserveVideoMemoryAllocations=1/nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableS0ixPowerManagement=0 nvidia.NVreg_DynamicPowerManagement=0/' /etc/default/grub
sudo update-grub
```

### Final GRUB line

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableS0ixPowerManagement=0 nvidia.NVreg_DynamicPowerManagement=0"
```

### Reboot required

```bash
sudo reboot
```

## Verification

Check parameters are active:

```bash
cat /proc/driver/nvidia/params | grep -E "(EnableS0ix|DynamicPower)"
```

Expected output:
```
EnableS0ixPowerManagement: 0
DynamicPowerManagement: 0
```

## Trade-offs

Disabling power management has some downsides:

| Aspect | Impact |
|--------|--------|
| Idle power | ~20-30W instead of ~7W |
| Heat | Slightly warmer at idle |
| Electricity cost | ~$20-40/year extra |
| GPU lifespan | Negligible impact |

These are acceptable trade-offs for a stable system.

## Alternative (Less Aggressive)

If power consumption is a concern, try disabling only S0ix:

```
nvidia.NVreg_EnableS0ixPowerManagement=0
```

Without disabling DynamicPowerManagement. This may still allow some power saving while avoiding the hang.

## Diagnosis Method

The issue was identified using background monitoring:

```bash
# GPU monitoring
nvidia-smi --query-gpu=timestamp,temperature.gpu,power.draw,memory.used,utilization.gpu --format=csv -l 5 >> ~/gpu_monitor.log &

# Kernel messages
sudo dmesg -w >> ~/dmesg_monitor.log &
```

After a hang, the dmesg log showed the nvidia-modeset errors starting exactly when the screen would have blanked.

## Related Issues

- `nvidia-suspend-fix.md` - PreserveVideoMemoryAllocations for suspend/resume
- `crash-monitoring-setup.md` - Monitoring setup used to diagnose this issue

## Date

2026-01-11
