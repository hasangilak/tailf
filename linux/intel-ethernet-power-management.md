# Fixing Intel Ethernet Controller I225-V Power Management Issues on Ubuntu 24 LTS

## The Problem

After putting Ubuntu 24 LTS to sleep and waking it up, the ethernet connection stops working. The only way to restore connectivity has been a full system restart - until now.

## Hardware Details

- **Ethernet Controller**: Intel Corporation Ethernet Controller I225-V (rev 03)
- **Driver**: igc (Intel Gigabit Ethernet Controller driver)
- **Interface**: enp4s0
- **OS**: Ubuntu 24 LTS
- **Kernel**: Linux 6.14.0-27-generic

## Root Cause Analysis

The issue stems from aggressive power management settings on the Intel I225-V ethernet controller. When the system suspends, the ethernet adapter enters a low-power state and sometimes fails to properly resume, leaving the interface in an inconsistent state.

Investigation using `strace` and system logs revealed:
- The ethernet interface shows as "UP" but loses network connectivity
- Power management is enabled (`/sys/class/net/enp4s0/device/power/control` shows "on")
- NetworkManager attempts to reconnect but fails due to hardware state issues

## Solutions

### Quick Fix (Immediate Relief)

Instead of restarting the entire system, reconnect the interface:

```bash
sudo nmcli device disconnect enp4s0 && sudo nmcli device connect enp4s0
```

### Permanent Solution 1: Disable Power Management

Prevent the ethernet controller from entering power-saving modes:

```bash
# Temporary fix (until next reboot)
echo 'on' | sudo tee /sys/class/net/enp4s0/device/power/control

# Permanent fix - create udev rule
sudo tee /etc/udev/rules.d/99-intel-ethernet-power.rules << 'EOF'
# Disable power management for Intel I225-V ethernet controller
ACTION=="add", SUBSYSTEM=="net", KERNEL=="enp4s0", RUN+="/bin/sh -c 'echo on > /sys/class/net/enp4s0/device/power/control'"
EOF
```

### Permanent Solution 2: Automated Suspend/Resume Script

Create a systemd service that handles the interface during suspend/resume cycles:

```bash
sudo tee /etc/systemd/system/fix-ethernet-suspend.service << 'EOF'
[Unit]
Description=Fix ethernet after suspend
Before=sleep.target suspend.target hibernate.target hybrid-sleep.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/true
ExecStop=/bin/bash -c '/usr/bin/nmcli device disconnect enp4s0; sleep 2; /usr/bin/nmcli device connect enp4s0'
TimeoutStopSec=30

[Install]
WantedBy=sleep.target suspend.target hibernate.target hybrid-sleep.target
EOF

sudo systemctl enable fix-ethernet-suspend.service
sudo systemctl daemon-reload
```

### Alternative Solution: Driver Parameter Tuning

Modify the Intel IGC driver behavior:

```bash
# Create modprobe configuration for IGC driver
echo 'options igc disable_fw_lldp=Y' | sudo tee /etc/modprobe.d/igc.conf

# Rebuild initramfs
sudo update-initramfs -u
```

## Testing the Fix

1. Apply one of the permanent solutions above
2. Reboot to ensure changes take effect
3. Put the system to sleep: `systemctl suspend`
4. Wake up the system
5. Test network connectivity: `ping 8.8.8.8`

## Command History Investigation

The troubleshooting process involved:

```bash
# Network interface inspection
ip link show enp4s0
ip addr show enp4s0

# NetworkManager debugging
nmcli device status
systemctl status NetworkManager

# Hardware investigation
lspci -v | grep -A 15 "Ethernet"
ethtool enp4s0

# Power management inspection
cat /sys/class/net/enp4s0/device/power/control
cat /sys/class/net/enp4s0/device/power/wakeup

# System logs analysis
journalctl --since "2 hours ago" | grep -i "suspend\|resume\|ethernet\|enp4s0\|igc"
```

## Why This Happens

Intel I225-V controllers are known to have power management quirks on Linux systems. The controller can enter a state where:

1. The hardware appears functional (`Link detected: yes`)
2. The interface shows as UP in the kernel
3. But actual packet transmission/reception fails
4. NetworkManager can't establish a proper connection

This is a common issue with newer Intel ethernet controllers that prioritize power efficiency over reliability during suspend/resume cycles.

## Prevention Tips

- Always use the latest kernel available for your Ubuntu version
- Keep the `igc` driver updated through regular system updates
- Consider disabling ethernet wake-on-LAN if not needed
- Monitor system logs after suspend/resume cycles to catch issues early

## Conclusion

Power management issues with Intel ethernet controllers on Linux are frustrating but solvable. The permanent solutions above should eliminate the need for system restarts while maintaining stable network connectivity through suspend/resume cycles.

The key insight is that sometimes the "nuclear option" (full restart) isn't necessary - targeted interface management can resolve hardware state issues more efficiently.