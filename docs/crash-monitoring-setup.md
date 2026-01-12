# Crash Monitoring Setup

## Problem

System becomes unreachable after approximately 20 minutes. Investigation revealed this was not suspend but actual crashes/reboots with no clear cause in logs (typical of hard lockups, possibly GPU-related).

## Applied Fix

Added NVIDIA kernel parameter to preserve video memory during suspend/resume:

```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia.NVreg_PreserveVideoMemoryAllocations=1"/' /etc/default/grub
sudo update-grub
```

Rebooted to apply.

## Monitoring Setup

To capture data before any future crashes, three background monitoring processes were started:

### 1. GPU Monitoring (PID 7365)

Logs GPU temperature, power draw, memory usage, and utilization every 5 seconds.

```bash
nvidia-smi --query-gpu=timestamp,temperature.gpu,power.draw,memory.used,utilization.gpu --format=csv -l 5 >> ~/gpu_monitor.log 2>&1 &
```

**Log file:** `~/gpu_monitor.log`

### 2. Kernel Messages (PID 7423/7424)

Captures dmesg in real-time to catch driver errors before a crash.

```bash
sudo dmesg -w >> ~/dmesg_monitor.log 2>&1 &
```

**Log file:** `~/dmesg_monitor.log`

### 3. System Journal (PID 7483)

Follows system logs for suspend/power/service events.

```bash
journalctl -f >> ~/journal_monitor.log 2>&1 &
```

**Log file:** `~/journal_monitor.log`

## Checking Monitoring Status

```bash
ps aux | grep -E "(nvidia-smi|dmesg|journalctl)" | grep -v grep
```

## Stopping Monitoring

```bash
# Stop all monitoring processes
kill 7365 7423 7424 7483

# Or find and kill by name
pkill -f "nvidia-smi.*gpu_monitor"
sudo pkill -f "dmesg -w"
pkill -f "journalctl -f"
```

## After a Crash

Check the log files for clues:

```bash
# Check GPU temps/power before crash (look for spikes)
tail -100 ~/gpu_monitor.log

# Check for kernel errors
tail -100 ~/dmesg_monitor.log

# Check system events
tail -100 ~/journal_monitor.log
```

### What to Look For

- **GPU overheating:** Temperature above 85C
- **Power spikes:** Unusual power draw values
- **NVIDIA errors:** Any nvidia/drm errors in dmesg
- **Suspend events:** PM: suspend/resume messages (if it was actually sleeping)

## Notes

- Logs will grow over time (~few MB per day)
- Monitoring processes will stop on reboot
- PIDs will change if restarted

## Date

2026-01-10
