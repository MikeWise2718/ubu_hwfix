# Wake-on-LAN Not Working After Suspend

## Problem

Wake-on-LAN (WoL) fails to wake the system from suspend, even though `ethtool` shows WoL is enabled.

## Cause

NetworkManager had the WoL setting set to `default`, which does not explicitly preserve the WoL configuration across suspend/resume cycles. The ethtool WoL setting can be reset during power state transitions.

## Affected Interface

- **Interface:** enp6s0
- **NetworkManager Connection:** Wired connection 2
- **Supports Wake-on:** pumbg (PHY, unicast, multicast, broadcast, magic packet)

## Solution

Configure NetworkManager to explicitly enable WoL (magic packet) for the connection.

### Set WoL to magic packet

```bash
sudo nmcli connection modify "Wired connection 2" 802-3-ethernet.wake-on-lan magic
```

### Verify the setting

```bash
nmcli connection show "Wired connection 2" | grep -i wake
```

Expected output:

```
802-3-ethernet.wake-on-lan:             magic
802-3-ethernet.wake-on-lan-password:    --
```

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

- WoL only works when the system is suspended or shut down (not fully powered off on all systems)
- The sending machine must be on the same Layer 2 network (same subnet/VLAN)
- Some routers/switches may block magic packets
- Setting persists across reboots via NetworkManager

## Date

2026-01-10
