# USB Keyboard Not Working After Suspend/Resume or Idle

## Problem

The USB keyboard becomes unresponsive in two scenarios:

1. **After waking from suspend (S3 sleep)** - mouse works, keyboard doesn't
2. **After extended idle period** - system never suspended, but keyboard stops responding

System logs may show:

```
usbhid: probe of 1-1:1.1 failed with error -71
```

Error -71 (EPROTO) indicates a USB protocol communication failure during device re-initialization after resume.

## Cause

Two separate issues can cause this:

1. **Keyboard autosuspend** - USB autosuspend was enabled for the keyboard with a 2-second timeout. During suspend/resume, the USB power management state becomes inconsistent.

2. **USB controller power management** - Even with keyboard autosuspend disabled, the USB root hub/controller itself can enter power-saving mode after idle, causing connected devices to become unresponsive.

## Affected Device

- **Vendor ID:** 1a2c
- **Product ID:** 2124
- **Description:** China Resource Semico Co., Ltd Keyboard
- **USB Path:** Bus 001, Port 1 (`/sys/bus/usb/devices/1-1`)

## Solution

Two fixes are required for complete resolution.

### Fix 1: Disable keyboard autosuspend

Create a udev rule for the specific keyboard:

```bash
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1a2c", ATTR{idProduct}=="2124", ATTR{power/autosuspend}="-1"' | sudo tee /etc/udev/rules.d/90-keyboard-nosuspend.rules
```

### Fix 2: Disable USB controller power management

Create a udev rule to keep USB root hubs always on (vendor 1d6b = Linux Foundation USB hubs):

```bash
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1d6b", ATTR{power/control}="on"' | sudo tee /etc/udev/rules.d/85-usb-power.rules
```

### Reload udev rules

```bash
sudo udevadm control --reload-rules
```

### Apply immediately (without reboot)

```bash
# Keyboard autosuspend
echo -1 | sudo tee /sys/bus/usb/devices/1-1/power/autosuspend

# USB controller power management
for usb in /sys/bus/usb/devices/usb*; do sudo sh -c "echo on > $usb/power/control"; done
```

## Verification

Check keyboard autosuspend is disabled:

```bash
cat /sys/bus/usb/devices/1-1/power/autosuspend
# Should output: -1
```

Check USB controllers have power management disabled:

```bash
for usb in /sys/bus/usb/devices/usb*; do echo "$usb: $(cat $usb/power/control)"; done
# All should output: on
```

## Notes

- Setting `autosuspend = -1` keeps the keyboard powered at all times
- Setting USB hub `power/control = on` prevents controller from sleeping
- Negligible impact on power consumption
- Both udev rules persist across reboots
- Rule files:
  - `/etc/udev/rules.d/90-keyboard-nosuspend.rules` (keyboard)
  - `/etc/udev/rules.d/85-usb-power.rules` (USB controllers)

## Date

2026-01-10 (initial fix)
2026-01-12 (added USB controller power management fix)
