# Ubuntu Hardware Fixes

Fixes for sleep/wake, power management, and remote access issues on an Ubuntu 22.04 workstation with NVIDIA RTX 4090.

## System

- **OS:** Ubuntu 22.04 LTS (kernel 6.8.0-90-generic)
- **CPU:** AMD Ryzen 9 5950X
- **GPU:** NVIDIA RTX 4090 (driver 570.x)
- **Motherboard:** ASUS ROG CROSSHAIR VIII HERO (WI-FI)

## Problems Solved

### GPU Hang on Screen Blank

**Symptom:** System becomes completely unresponsive after idle (~15-20 min). Requires hard reboot.

**Cause:** NVIDIA S0ix and Dynamic Power Management cause GPU to enter low-power state when display blanks; GPU fails to wake.

**Fix:** Disable NVIDIA power management via kernel parameters:
```
nvidia.NVreg_PreserveVideoMemoryAllocations=1
nvidia.NVreg_EnableS0ixPowerManagement=0
nvidia.NVreg_DynamicPowerManagement=0
```

See: [nvidia-power-management-hang-fix.md](docs/nvidia-power-management-hang-fix.md)

### USB Keyboard Unresponsive After Idle/Suspend

**Symptom:** Keyboard stops working after suspend or extended idle. Mouse still works.

**Cause:** USB autosuspend on keyboard + USB controller power management.

**Fix:** udev rules to disable power management:
```bash
# Keyboard autosuspend
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1a2c", ATTR{idProduct}=="2124", ATTR{power/autosuspend}="-1"' | sudo tee /etc/udev/rules.d/90-keyboard-nosuspend.rules

# USB controller power management
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1d6b", ATTR{power/control}="on"' | sudo tee /etc/udev/rules.d/85-usb-power.rules
```

See: [usb-keyboard-resume-fix.md](docs/usb-keyboard-resume-fix.md)

### NVIDIA Suspend/Resume Errors

**Symptom:** Display corruption or system instability after waking from suspend.

**Cause:** NVIDIA driver not preserving video memory across suspend.

**Fix:** Enable video memory preservation + systemd services:
```bash
# Kernel parameter
nvidia.NVreg_PreserveVideoMemoryAllocations=1

# Enable services
sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
```

See: [nvidia-suspend-fix.md](docs/nvidia-suspend-fix.md)

### Wake-on-LAN

**Symptom:** WoL works from power-off but not from suspend.

**Cause:** NIC WoL settings not preserved through suspend; S3 WoL is hardware-dependent.

**Fix:** NetworkManager config + sleep hook.

See: [wake-on-lan-fix.md](docs/wake-on-lan-fix.md)

## Documentation

| File | Description |
|------|-------------|
| [nvidia-power-management-hang-fix.md](docs/nvidia-power-management-hang-fix.md) | GPU hang on screen blank |
| [nvidia-suspend-fix.md](docs/nvidia-suspend-fix.md) | NVIDIA suspend/resume |
| [usb-keyboard-resume-fix.md](docs/usb-keyboard-resume-fix.md) | USB keyboard issues |
| [wake-on-lan-fix.md](docs/wake-on-lan-fix.md) | Wake-on-LAN configuration |
| [crash-monitoring-setup.md](docs/crash-monitoring-setup.md) | Monitoring for crash diagnosis |
| [potential_issues_10Jan2026.md](docs/potential_issues_10Jan2026.md) | Other known issues |

## Quick Reference

### GRUB Configuration

Edit `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableS0ixPowerManagement=0 nvidia.NVreg_DynamicPowerManagement=0"
```

Then run:
```bash
sudo update-grub
sudo reboot
```

### Verify Fixes

```bash
# Check NVIDIA parameters
cat /proc/cmdline | grep nvidia

# Check USB power management
for usb in /sys/bus/usb/devices/usb*; do echo "$usb: $(cat $usb/power/control)"; done

# Check WoL
ethtool enp6s0 | grep Wake-on
```

## License

These are personal notes and fixes. Use at your own risk. Your hardware may differ.
