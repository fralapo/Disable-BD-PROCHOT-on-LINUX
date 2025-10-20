# Disable BD PROCHOT on LINUX

This repository offers a comprehensive method for disabling BD PROCHOT (Bi-Directional Processor Hot) on Linux systems at boot time and after resume from suspend. BD PROCHOT is a thermal throttling feature that allows external devices to signal the CPU to throttle down to avoid overheating. Disabling BD PROCHOT can help maintain performance if thermal throttling is happening prematurely, but it should be done with caution.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [Troubleshooting](#troubleshooting)
- [Cautions](#cautions)
- [Uninstallation](#uninstallation)
- [License](#license)

## Prerequisites

Before proceeding, ensure you have `msr-tools` installed on your system. Here's how to install `msr-tools` on various Linux distributions:

- **Arch Linux:**
  ```
  sudo pacman -S msr-tools
  ```
- **Ubuntu and derivatives:**
  ```
  sudo apt-get install msr-tools
  ```
- **Bazzite and Fedora derivatives:**
  ```
  sudo rpm-ostree install msr-tools
  ```
- **CentOS:**
  ```
  sudo yum install msr-tools
  ```

## Installation

### Automatic Installation (Recommended)

The automated script handles all installation steps, including boot service and suspend/resume hooks.

Run the following command:
```
curl -LO https://raw.githubusercontent.com/fralapo/Disable-BD-PROCHOT-on-LINUX/main/Disable_BD_PROCHOT
sudo bash Disable_BD_PROCHOT
```

The script will:
1. Install `msr-tools` for your distribution
2. Create the BD PROCHOT disable script
3. Set up a systemd service for boot execution
4. Create a suspend/resume hook for post-suspend execution
5. Enable and start the service

### Manual Installation

If you prefer manual installation, follow these steps:

#### Step 1: Create the Main Script

1. Open a terminal and create the script file:
   ```
   sudo nano /usr/local/bin/disable_bd_prochot.sh
   ```
2. Insert the following commands:
   ```
   #!/bin/bash
   modprobe msr
   rdmsr 0x1FC
   wrmsr 0x1FC 2c005d
   modprobe msr
   rdmsr 0x1FC
   wrmsr 0x1FC 2c005d
   ```
3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`)

#### Step 2: Make the Script Executable

```
sudo chmod +x /usr/local/bin/disable_bd_prochot.sh
```

#### Step 3: Create the Systemd Service

1. Create the service file:
   ```
   sudo nano /etc/systemd/system/disable_bd_prochot.service
   ```
2. Insert the following content:
   ```
   [Unit]
   Description=Disable BD PROCHOT
   After=multi-user.target

   [Service]
   Type=oneshot
   ExecStart=/usr/local/bin/disable_bd_prochot.sh
   User=root
   Group=root
   RemainAfterExit=yes

   [Install]
   WantedBy=multi-user.target
   ```
3. Save and exit

#### Step 4: Create the Suspend/Resume Hook

1. Create the sleep hook script:
   ```
   sudo nano /usr/lib/systemd/system-sleep/disable_bd_prochot
   ```
   (Note: On some distributions like Ubuntu, use `/lib/systemd/system-sleep/` instead)

2. Insert the following content:
   ```
   #!/bin/sh
   # Script to re-disable BD PROCHOT after resume from suspend

   case $1 in
       post)
           # Re-disable BD PROCHOT after resume
           modprobe msr
           wrmsr 0x1FC 2c005d
           ;;
   esac
   ```
3. Save and exit

4. Make the hook executable:
   ```
   sudo chmod +x /usr/lib/systemd/system-sleep/disable_bd_prochot
   ```

#### Step 5: Enable and Start the Service

```
sudo systemctl daemon-reload
sudo systemctl enable disable_bd_prochot.service
sudo systemctl start disable_bd_prochot.service
```

#### Step 6: Reboot

Reboot your system to verify everything works:
```
sudo reboot
```

## Usage

After installation, BD PROCHOT will be automatically disabled:
- **At boot time** via the systemd service
- **After resume from suspend** via the systemd sleep hook

### Verify BD PROCHOT Status

Check the MSR register:

```
sudo rdmsr -a 0x1FC
```

Look at the **last digit** of the result:
- **Even digit** (0, 2, 4, 6, 8, A, C, E) = BD PROCHOT **disabled** ✓
- **Odd digit** (1, 3, 5, 7, 9, B, D, F) = BD PROCHOT **active** ✗

**Examples:**
```
2c005c  → Last digit 'c' (even) = disabled ✓
2c005d  → Last digit 'd' (odd) = active ✗
```

### Verify CPU Frequency

Monitor frequency in real-time:

```
watch -n 1 "lscpu | grep MHz"
```

**Expected output (BD PROCHOT disabled):**
```
CPU(s) scaling MHz:     80%      ← Varies between 20%-100%
CPU max MHz:            4000.0000
CPU min MHz:            400.0000
```

**Warning signs (BD PROCHOT still active):**
```
CPU(s) scaling MHz:     20%      ← Stuck at 20% (~800 MHz)
CPU max MHz:            4000.0000
CPU min MHz:            400.0000
```

If scaling is stuck at **20%**, it means your CPU is running at approximately **800 MHz** instead of reaching maximum frequency.

### Alternative Command

To see all core frequencies:

```
grep MHz /proc/cpuinfo
```

## How It Works

The solution implements two complementary mechanisms:

### Boot Service
A systemd service (`disable_bd_prochot.service`) runs at system startup to disable BD PROCHOT by writing to the CPU's Model-Specific Register (MSR) 0x1FC.

### Suspend/Resume Hook
A systemd sleep hook script in `/usr/lib/systemd/system-sleep/` automatically re-disables BD PROCHOT after the system resumes from suspend. This is necessary because entering ACPI S3 state (suspend) causes the BD PROCHOT MSR bit to be re-enabled by the system.

When systemd suspends or resumes the system, it executes all scripts in the `system-sleep` directory with arguments indicating the sleep state:
- `pre` - Before entering suspend
- `post` - After resuming from suspend

The hook script only acts on the `post` event to re-disable BD PROCHOT after wake-up.

## Troubleshooting

### CPU still throttled after suspend
If your CPU returns to 800 MHz after suspend, verify the sleep hook is installed correctly:
```
ls -l /usr/lib/systemd/system-sleep/disable_bd_prochot
# or on some systems:
ls -l /lib/systemd/system-sleep/disable_bd_prochot
```

Ensure the script is executable:
```
sudo chmod +x /usr/lib/systemd/system-sleep/disable_bd_prochot
```

### Service not starting
Check the service status:
```
sudo systemctl status disable_bd_prochot.service
```

View logs for errors:
```
journalctl -u disable_bd_prochot.service
```

### MSR module not loading
Ensure the `msr` kernel module is available:
```
sudo modprobe msr
lsmod | grep msr
```

## Cautions

⚠️ **Important Safety Information:**

- **Overheating Risk**: Disabling BD PROCHOT removes an important thermal protection mechanism. Monitor your CPU temperatures closely to prevent overheating.
- **Hardware Damage**: In extreme cases, prolonged high temperatures can damage your CPU or other components. Use at your own risk.
- **Warranty**: Modifying system registers may void your warranty.
- **Broken Sensors**: This solution is primarily intended for systems where BD PROCHOT is triggered by faulty thermal sensors, not for bypassing legitimate thermal limits.
- **Distribution Variations**: The steps provided may vary slightly based on your Linux distribution.
- **Backup Your Data**: Always backup important data before making system-level changes.

**Recommended**: Use thermal monitoring tools like `sensors` or `htop` to keep an eye on temperatures after disabling BD PROCHOT.

## Uninstallation

If you want to remove BD PROCHOT disabler and restore the default thermal management behavior, you can use the uninstall script.

### Automatic Uninstallation (Recommended)

Run the following command:
```
curl -LO https://raw.githubusercontent.com/fralapo/Disable-BD-PROCHOT-on-LINUX/main/Uninstall_BD_PROCHOT
sudo bash Uninstall_BD_PROCHOT
```

The script will:
1. Stop the disable_bd_prochot service
2. Disable the service from running at boot
3. Remove the systemd service file
4. Remove the main disable script
5. Remove the suspend/resume hook
6. Reload systemd daemon

After uninstallation, BD PROCHOT will be re-enabled and your CPU will return to normal thermal throttling behavior.

### Manual Uninstallation

If you prefer to uninstall manually, follow these steps:

#### Step 1: Stop and Disable the Service

```
sudo systemctl stop disable_bd_prochot.service
sudo systemctl disable disable_bd_prochot.service
```

#### Step 2: Remove the Service File

```
sudo rm /etc/systemd/system/disable_bd_prochot.service
```

#### Step 3: Remove the Main Script

```
sudo rm /usr/local/bin/disable_bd_prochot.sh
```

#### Step 4: Remove the Sleep Hook

```
# Try both locations (depends on your distribution)
sudo rm /usr/lib/systemd/system-sleep/disable_bd_prochot
sudo rm /lib/systemd/system-sleep/disable_bd_prochot
```

#### Step 5: Reload Systemd

```
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

#### Step 6: Reboot

Reboot your system for changes to take full effect:
```
sudo reboot
```

After rebooting, verify that BD PROCHOT is active again:

```
sudo rdmsr -a 0x1FC
```

Look at the **last digit** of the result:
- **Even digit** (0, 2, 4, 6, 8, A, C, E) = BD PROCHOT **disabled** ✓
- **Odd digit** (1, 3, 5, 7, 9, B, D, F) = BD PROCHOT **active** ✗

**Examples:**
```
2c005c  → Last digit 'c' (even) = disabled ✓
2c005d  → Last digit 'd' (odd) = active ✗
```

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Contributing

Contributions are welcome! If you encounter issues or have improvements, please:
1. Check existing issues on GitHub
2. Open a new issue with detailed information
3. Submit pull requests with clear descriptions

## Acknowledgments

- Thanks to the community members who identified the suspend/resume issue
- Inspired by similar solutions like [ThrottleStop](https://www.techpowerup.com/download/techpowerup-throttlestop/) for Windows
- Based on MSR manipulation techniques from the Linux kernel documentation
