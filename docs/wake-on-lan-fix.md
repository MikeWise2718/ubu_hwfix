# Wake-on-LAN Configuration

## Problem

Wake-on-LAN (WoL) behavior differs between power states:

- **S5 (power off):** WoL works reliably
- **S3 (suspend/sleep):** WoL may not work depending on hardware/BIOS

## Cause

Two issues can prevent WoL from working:

1. **NetworkManager default settings** - WoL setting set to `default` doesn't explicitly preserve WoL configuration across power state transitions

2. **S3 suspend limitations** - Some NICs or BIOS configurations don't support WoL from S3 sleep state, only from S5 power-off

## Affected Interface

- **Interface:** enp6s0
- **NIC:** Intel Corporation I211 Gigabit Network Connection
- **PCIe Address:** 06:00.0
- **NetworkManager Connection:** Wired connection 2
- **Supports Wake-on:** pumbg (PHY, unicast, multicast, broadcast, magic packet)

## Solution

### Fix 1: Configure NetworkManager WoL

Set WoL to magic packet:

```bash
sudo nmcli connection modify "Wired connection 2" 802-3-ethernet.wake-on-lan magic
```

Verify the setting:

```bash
nmcli connection show "Wired connection 2" | grep -i wake
```

Expected output:

```
802-3-ethernet.wake-on-lan:             magic
802-3-ethernet.wake-on-lan-password:    --
```

### Fix 2: Sleep Hook for S3 Suspend

Create a systemd sleep hook to ensure WoL is set before suspend:

```bash
cat << 'EOF' | sudo tee /usr/lib/systemd/system-sleep/wol
#!/bin/sh
# Ensure Wake-on-LAN is set before suspend and after resume

case "$1" in
    pre)
        # Set WoL before going to sleep
        /usr/sbin/ethtool -s enp6s0 wol g
        ;;
    post)
        # Re-set WoL after waking (in case it was reset)
        /usr/sbin/ethtool -s enp6s0 wol g
        ;;
esac
EOF
sudo chmod +x /usr/lib/systemd/system-sleep/wol
```

This ensures ethtool explicitly sets WoL before the system enters S3 sleep.

## Additional Requirements

### BIOS/UEFI Settings

Ensure these options are enabled in firmware settings:

- "Wake on LAN" or "Power On By PCI-E/PCIe"
- "Deep Sleep" may need to be disabled on some systems

### Testing WoL

From another machine on the same network:

```bash
# Get the MAC address of enp6s0
ip link show enp6s0 | grep ether

# Send magic packet (install wakeonlan or etherwake if needed)
wakeonlan <MAC_ADDRESS>
# or
etherwake <MAC_ADDRESS>
```

## Notes

- WoL works reliably from S5 (power off) state
- WoL from S3 (suspend) is hardware-dependent and may not work on all systems
- The sending machine must be on the same Layer 2 network (same subnet/VLAN)
- Some routers/switches may block magic packets
- NetworkManager setting persists across reboots
- Sleep hook file: `/usr/lib/systemd/system-sleep/wol`

## Alternative Remote Wake Methods

If WoL from S3 doesn't work, consider these alternatives:

### 1. Disable Sleep for Remote Access

```bash
# Disable auto-suspend entirely
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 0
```

### 2. RTC Wake (Scheduled Wake)

```bash
# Sleep for 1 hour then automatically wake
sudo rtcwake -m mem -s 3600
```

### 3. Smart Plug

Use a smart plug to power cycle the machine remotely. Requires BIOS set to "Power on after AC restore".

### 4. Wake-on-USB

Some motherboards support waking via USB device activity (keyboard/mouse).

## Current Status

| Power State | WoL Status |
|-------------|------------|
| S5 (power off) | Working |
| S3 (suspend) | May not work - hardware dependent |

## Date

2026-01-10 (initial fix)
2026-01-12 (added sleep hook and S3 notes)
