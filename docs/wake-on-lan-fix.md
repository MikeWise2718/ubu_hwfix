# Wake-on-LAN Configuration

## Problem

Wake-on-LAN (WoL) behavior differs between power states:

- **S5 (power off):** WoL works reliably
- **S3 (suspend/sleep):** WoL may not work depending on hardware/BIOS

## Cause

Three issues can prevent WoL from working:

1. **NetworkManager default settings** - WoL setting set to `default` doesn't explicitly preserve WoL configuration across power state transitions

2. **PCIe bridge wakeup disabled** - On AMD systems with PCIe switches, the intermediate bridges between the NIC and CPU may have wakeup disabled, blocking the wake signal from propagating

3. **S3 suspend limitations** - Some NICs or BIOS configurations don't support WoL from S3 sleep state, only from S5 power-off

## Affected Interfaces

This motherboard has two onboard NICs. Both were tested for WoL from S3.

| Interface | NIC | Speed | PCIe Address | MAC Address | WoL from S3 |
|-----------|-----|-------|--------------|-------------|-------------|
| enp6s0 | Intel I211 | 1 Gbit | 06:00.0 | 50:EB:F6:B6:80:36 | **Not working** |
| enp5s0 | Realtek RTL8125 | 2.5 Gbit | 05:00.0 | 50:EB:F6:B6:80:35 | **Not working** |

Both NICs support WoL (pumbg - PHY, unicast, multicast, broadcast, magic packet) but neither works from S3 suspend due to ACPI/BIOS limitations.

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

### Fix 3: Enable PCIe Bridge Wakeup (Critical for S3)

On AMD Matisse/X570 systems, the NIC connects through PCIe switches. The intermediate bridges may have wakeup disabled by default, blocking wake signals from reaching the CPU.

**PCIe path for Intel I211:**
```
0000:00:01.2 (Root Port - GPP1)
  └─ 0000:02:00.0 (Matisse Switch Upstream)
       └─ 0000:03:05.0 (Matisse PCIe GPP Bridge)
            └─ 0000:06:00.0 (Intel I211 NIC)
```

**Check current status:**
```bash
for dev in 0000:00:01.2 0000:02:00.0 0000:03:05.0 0000:06:00.0; do
  echo "$dev: $(cat /sys/bus/pci/devices/$dev/power/wakeup 2>/dev/null || echo 'N/A')"
done
```

**Enable wakeup on bridges manually:**
```bash
sudo sh -c 'echo enabled > /sys/bus/pci/devices/0000:02:00.0/power/wakeup'
sudo sh -c 'echo enabled > /sys/bus/pci/devices/0000:03:05.0/power/wakeup'
```

**Make persistent with udev rule:**
```bash
cat << 'EOF' | sudo tee /etc/udev/rules.d/81-pcie-wakeup.rules
# Enable wakeup on PCIe bridges for WoL to work from S3 suspend
# Path: Root port -> Switch -> Downstream port -> Intel I211 NIC
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:02:00.0", ATTR{power/wakeup}="enabled"
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:03:05.0", ATTR{power/wakeup}="enabled"
EOF
sudo udevadm control --reload-rules
```

## Additional Requirements

### BIOS/UEFI Settings (Critical)

The ACPI wakeup table shows PCIe devices only support wake from **S4**, not S3:

```
GPP1    S4    *enabled   pci:0000:00:01.2
```

This means the BIOS doesn't expose S3 wake capability.

#### ASUS ROG CROSSHAIR VIII HERO (WI-FI) Specific Settings

**Motherboard:** ASUS ROG CROSSHAIR VIII HERO (WI-FI)
**BIOS:** AMI Version 5302 (Oct 2025) - updated from 4702

Make sure you're in **Advanced Mode** (press F7 if in EZ Mode).

**Advanced → APM Configuration:**
- **ErP Ready** → **Disabled** (critical - ErP disables WoL entirely)
- **Power On By PCI-E** → **Enabled**

**Advanced → Onboard Devices Configuration:**
- **USB power delivery in Soft Off state (S5)** → **Enabled** (may help)

#### Known Issues with This Board

Users report WoL from S3 (suspend) can be unreliable on this board:
- May only work within a few minutes of sleep
- After hours of sleep, WoL may stop working
- WoL from S5 (power off) works reliably

**ASUS recommended troubleshooting:**
1. Reset BIOS to defaults
2. Only enable "Power On By PCI-E"
3. Test WoL

#### References

- [Wake-on-LAN fix for ROG Crosshair VIII Dark Hero](https://dannyda.com/2021/12/28/how-to-enable-fix-wake-on-lan-wol-for-asus-rog-crosshair-viii-dark-hero-may-work-for-other-models-of-motherboard-or-even-other-brands-of-motherboard/)
- [ASUS Official: How to enable WOL in BIOS](https://www.asus.com/support/faq/1045950/)
- [ROG Forum: x570 Crosshair VIII Hero WOL issues](https://rog-forum.asus.com/t5/other-motherboards/x570-rog-crosshair-viii-hero-wol/td-p/825242)

Without proper BIOS configuration, Linux cannot override the ACPI limitations.

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
| S3 (suspend) | **Not working** - BIOS/ACPI limitation |

### What Was Tried

1. **BIOS update:** Updated from version 4702 (Oct 2023) to 5302 (Oct 2025) - no change to ACPI wakeup table
2. **Intel I211 NIC:** Configured WoL, enabled PCIe bridge wakeup - did not work
3. **Realtek RTL8125 NIC:** Switched to alternate NIC, configured WoL, enabled PCIe bridge wakeup - did not work
4. **PCIe bridge wakeup:** Enabled on all intermediate bridges for both NICs - did not help
5. **Sleep hook:** Created systemd sleep hook to set WoL before suspend - did not help

### Why S3 WoL Doesn't Work

Even with all Linux-side fixes applied (NetworkManager, sleep hook, PCIe bridge wakeup), WoL from S3 fails because:

1. **ACPI table limitation:** `/proc/acpi/wakeup` shows `GPP1 S4 *enabled` - the BIOS only exposes wake from S4 (hibernate), not S3 (suspend)

2. **PME not enabled by driver:** `lspci -vv` shows `PME-Enable-` even after suspend - the driver doesn't enable PME because ACPI doesn't advertise S3 wake support

3. **BIOS update did not help:** Updating to latest BIOS (5302) did not change the ACPI S4-only limitation

4. **Both NICs affected:** Neither the Intel I211 nor Realtek RTL8125 can wake from S3 - this is a motherboard-level limitation, not NIC-specific

## Files

- `/usr/lib/systemd/system-sleep/wol` - Sleep hook for ethtool
- `/etc/udev/rules.d/81-pcie-wakeup.rules` - PCIe bridge wakeup

## Date

2026-01-10 (initial fix)
2026-01-12 (added sleep hook and S3 notes)
2026-01-12 (added PCIe bridge wakeup fix)
2026-01-12 (BIOS update 4702→5302, tested both NICs - S3 WoL confirmed not working)
