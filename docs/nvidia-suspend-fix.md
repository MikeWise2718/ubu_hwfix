# NVIDIA GPU Suspend/Resume Fix

## Problem

After waking from suspend, the system experiences instability or fails to wake properly. System logs show NVIDIA driver errors:

```
nvidia-modeset: ERROR: GPU:0: Failed to query display engine channel state: 0x0000c67e:6:0:0x00000062
```

## Cause

The NVIDIA proprietary driver does not always properly preserve video memory allocations across suspend/resume cycles, leading to display and GPU state corruption.

## Solution

Enable NVIDIA's video memory preservation feature and use the official NVIDIA suspend/resume/hibernate systemd services.

### Step 1: Add kernel parameter

Edit `/etc/default/grub` and add the parameter to `GRUB_CMDLINE_LINUX_DEFAULT`:

```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1"/' /etc/default/grub
```

Or manually edit the file to change:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1"
```

### Step 2: Update GRUB

```bash
sudo update-grub
```

### Step 3: Enable NVIDIA suspend services

```bash
sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
```

These services handle saving and restoring GPU state during power transitions.

### Step 4: Reboot

A reboot is required for the GRUB changes to take effect.

## Verification

Check that the kernel parameter is active:

```bash
cat /proc/cmdline | grep -o 'nvidia.NVreg_PreserveVideoMemoryAllocations=[0-9]'
# Should output: nvidia.NVreg_PreserveVideoMemoryAllocations=1
```

Check that services are enabled:

```bash
systemctl is-enabled nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
# Should output: enabled (three times)
```

## Notes

- `NVreg_PreserveVideoMemoryAllocations=1` tells the driver to save VRAM contents to system RAM during suspend
- This may slightly increase suspend/resume time depending on VRAM usage
- The nvidia-suspend/resume/hibernate services are provided by the nvidia-driver package
- If services don't exist, you may need to install `nvidia-driver` or a newer version

## Date

2026-01-10
