# Ubuntu 24 Wired Network Troubleshooting After Suspend/Wake

This guide provides offline-friendly solutions for fixing ethernet connectivity issues after suspending and waking your Ubuntu 24 system.

## Quick Diagnostics

### 1. Identify Your Network Interface and Driver

```bash
# List all network interfaces
ip link show

# Show network status
nmcli device status

# Find your ethernet interface name (usually eth0, enp3s0, etc.)
ip link show | grep -E "^[0-9]+: (eth|enp|eno)"

# Get driver information (replace enp3s0 with your interface)
sudo ethtool -i enp3s0

# Find ethernet hardware and PCI address
lspci | grep -i ethernet

# Get detailed ethernet controller info
lspci -v | grep -A 10 "Ethernet controller"
```

### 2. Check Current Network State

```bash
# Check if interface is up
ip link show enp3s0

# Check NetworkManager status
systemctl status NetworkManager

# View system logs for network errors
journalctl -xe | grep -i "network\|ethernet\|eth0\|enp"
```

## Immediate Manual Fixes (When Network is Down)

### Method 1: Restart NetworkManager

```bash
sudo systemctl restart NetworkManager
```

### Method 2: Manually Reload Ethernet Driver

First identify your driver from the diagnostics above, then:

```bash
# For Intel e1000e driver
sudo modprobe -r e1000e
sudo modprobe e1000e

# For Realtek r8169 driver
sudo modprobe -r r8169
sudo modprobe r8169

# For Intel igc driver (I225-V)
sudo modprobe -r igc
sudo modprobe igc

# For Marvell sky2 driver
sudo modprobe -r sky2
sudo modprobe sky2
```

### Method 3: Bring Interface Up Manually

```bash
# Replace enp3s0 with your interface name
sudo ip link set enp3s0 down
sudo ip link set enp3s0 up

# Or using nmcli
sudo nmcli device disconnect enp3s0
sudo nmcli device connect enp3s0
```

### Method 4: Full Network Stack Restart

```bash
# Stop networking
sudo systemctl stop NetworkManager
sudo systemctl stop systemd-resolved

# Reload driver (use your driver name)
sudo modprobe -r e1000e
sudo modprobe e1000e

# Start networking
sudo systemctl start systemd-resolved
sudo systemctl start NetworkManager
```

## Permanent Solutions

### Solution 1: Systemd Sleep Hook (Most Effective)

Create a script that automatically reloads your ethernet driver after resume:

```bash
# Create the sleep hook script
sudo nano /lib/systemd/system-sleep/ethernet-fix
```

Copy and paste this content (replace YOUR_DRIVER with your actual driver name):

```bash
#!/bin/bash
# Ethernet fix for suspend/resume issues
# Replace YOUR_DRIVER with your actual driver (e.g., e1000e, r8169, igc)

case $1/$2 in
  pre/suspend)
    echo "$(date): Unloading ethernet driver before suspend" >> /var/log/ethernet-suspend.log
    modprobe -r YOUR_DRIVER
    ;;
  post/suspend)
    echo "$(date): Reloading ethernet driver after resume" >> /var/log/ethernet-suspend.log
    modprobe -i YOUR_DRIVER
    # Optional: restart NetworkManager
    systemctl restart NetworkManager
    ;;
esac
```

Make it executable:

```bash
sudo chmod +x /lib/systemd/system-sleep/ethernet-fix
```

### Solution 2: NetworkManager Sleep Hook

Create a hook specifically for NetworkManager:

```bash
sudo nano /lib/systemd/system-sleep/network-manager-fix
```

Content:

```bash
#!/bin/bash
case $1/$2 in
  pre/suspend)
    systemctl stop NetworkManager
    ;;
  post/suspend)
    systemctl start NetworkManager
    ;;
esac
```

Make executable:

```bash
sudo chmod +x /lib/systemd/system-sleep/network-manager-fix
```

### Solution 3: Intel IGC Driver Specific Fix (For I225-V Controllers)

If you have Intel I225-V (igc driver), create this script:

```bash
sudo nano /lib/systemd/system-sleep/rebind-ethernet
```

First find your PCI address:

```bash
lspci | grep -i ethernet
# Look for something like "07:00.0 Ethernet controller"
```

Then use this content (replace 0000:07:00.0 with your PCI address):

```bash
#!/bin/bash
if [ "$1" = "post" ]; then
  echo "Rebinding Intel Ethernet" | logger
  echo 0000:07:00.0 > /sys/bus/pci/drivers/igc/unbind
  sleep 1
  echo 0000:07:00.0 > /sys/bus/pci/drivers/igc/bind
  systemctl restart NetworkManager
fi
```

Make executable:

```bash
sudo chmod +x /lib/systemd/system-sleep/rebind-ethernet
```

### Solution 4: Disable Power Management

Edit GRUB configuration:

```bash
sudo nano /etc/default/grub
```

Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and add these parameters:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash eee_enable=0 pcie_port_pm=off pcie_aspm=off"
```

Update GRUB:

```bash
sudo update-grub
```

### Solution 5: Wake-on-LAN Configuration

Edit your netplan configuration:

```bash
# Find your netplan file
ls /etc/netplan/

# Edit it (filename may vary)
sudo nano /etc/netplan/01-network-manager-all.yaml
```

Add wakeonlan setting:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp3s0:  # Replace with your interface name
      wakeonlan: true
```

Apply changes:

```bash
sudo netplan apply
```

### Solution 6: Disable Wake-on-LAN (If Causing Issues)

Create a systemd service:

```bash
sudo nano /etc/systemd/system/disable-wol.service
```

Content (replace enp3s0 with your interface):

```ini
[Unit]
Description=Disable Wake On Lan
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -s enp3s0 wol d

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable disable-wol.service
```

## Driver-Specific Solutions

### Intel e1000e

```bash
# Create sleep hook
sudo nano /lib/systemd/system-sleep/e1000e-fix

# Content:
#!/bin/bash
case $1/$2 in
  post/suspend)
    modprobe -r e1000e
    modprobe e1000e
    echo 1 > /sys/class/net/*/device/reset
    ;;
esac

# Make executable
sudo chmod +x /lib/systemd/system-sleep/e1000e-fix
```

### Realtek r8169

```bash
# Create sleep hook
sudo nano /lib/systemd/system-sleep/r8169-fix

# Content:
#!/bin/bash
case $1/$2 in
  post/suspend)
    modprobe -r r8169
    modprobe r8169
    ethtool -s enp3s0 wol d  # Replace enp3s0
    ;;
esac

# Make executable
sudo chmod +x /lib/systemd/system-sleep/r8169-fix
```

## Troubleshooting Steps

### 1. Check if Scripts are Running

```bash
# Check logs after suspend/resume
sudo journalctl -b | grep -i "ethernet-fix\|suspend\|resume"

# Check custom log if you added logging
cat /var/log/ethernet-suspend.log
```

### 2. Test Sleep Hook Manually

```bash
# Test the script
sudo /lib/systemd/system-sleep/ethernet-fix post suspend
```

### 3. Check for Module Loading Issues

```bash
# Check if module can be loaded
sudo modprobe -v YOUR_DRIVER

# Check for module errors
dmesg | grep -i "YOUR_DRIVER\|ethernet"
```

### 4. Verify Permissions

```bash
# Check script permissions
ls -la /lib/systemd/system-sleep/

# Ensure scripts are executable
sudo chmod +x /lib/systemd/system-sleep/*
```

### 5. Alternative Locations for Scripts

If `/lib/systemd/system-sleep/` doesn't work, try:

```bash
# User-specific location
/etc/systemd/system-sleep/

# Or create as systemd service
/etc/systemd/system/
```

## Emergency Recovery

If network won't come up at all:

```bash
# 1. Boot into recovery mode (hold Shift during boot)

# 2. Enable networking in recovery
mount -o remount,rw /
dhclient -v

# 3. Remove problematic scripts
rm /lib/systemd/system-sleep/ethernet-fix

# 4. Reset network configuration
mv /etc/netplan/*.yaml /etc/netplan/backup.yaml
netplan generate
netplan apply
```

## Testing Your Fix

After implementing a solution:

```bash
# 1. Test suspend
systemctl suspend

# 2. Wake the system

# 3. Check network immediately
ip link show
ping -c 3 8.8.8.8

# 4. Check logs
journalctl -xe --since "5 minutes ago"
```

## Common Issues and Solutions

### Issue: Script runs but network doesn't come back
- Add sleep delays in the script
- Try stopping/starting NetworkManager instead of restart
- Check for Secure Boot interference

### Issue: Multiple ethernet interfaces
- Modify scripts to handle all interfaces
- Use wildcards: `/sys/class/net/*/device/reset`

### Issue: Network comes back but no DHCP
- Add `dhclient YOUR_INTERFACE` to the script
- Or use `nmcli device connect YOUR_INTERFACE`

## Additional Commands Reference

```bash
# Force DHCP renewal
sudo dhclient -r enp3s0
sudo dhclient enp3s0

# Reset all network settings
sudo ip addr flush dev enp3s0
sudo systemctl restart NetworkManager

# Check hardware issues
sudo lshw -C network
sudo ethtool enp3s0

# Monitor network in real-time
watch -n 1 "ip link show; echo; nmcli device status"
```

---

**Remember**: 
- Always replace `YOUR_DRIVER` with your actual driver name (e.g., e1000e, r8169, igc)
- Replace `enp3s0` with your actual interface name
- Test one solution at a time
- Keep this guide accessible offline for when your network is down!