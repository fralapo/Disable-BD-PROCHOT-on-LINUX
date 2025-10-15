# Disable-BD-PROCHOT-on-LINUX

This repository offers a method for disabling BD PROCHOT (Bi-Directional Processor Hot) on Linux systems at boot time. BD PROCHOT is a thermal throttling feature that allows external devices to signal the CPU to throttle down to avoid overheating. Disabling BD PROCHOT can help maintain performance if thermal throttling is happening prematurely, but it should be done with caution.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Automatic Script](#automatic-script)
- [Cautions](#cautions)
- [License](#license)

## Prerequisites

Before proceeding, ensure you have `msr-tools` installed on your system. Here's how to install `msr-tools` on various Linux distributions:

- Arch Linux:
  ```bash
  sudo pacman -S msr-tools
  ```
- Ubuntu and derivatives:
  ```bash
  sudo apt-get install msr-tools
  ```
- Fedora and derivatives:
  ```bash
  sudo rpm-ostree install msr-tools
  ```
- CentOS:
  ```bash
  sudo yum install msr-tools
  ```

## Installation

### Step 1: Create the Script

1. Open a terminal and run the following command to create a new script file:
   ```sh
   sudo nano /usr/local/bin/disable_bd_prochot.sh
   ```
2. Insert the following commands into the script:
   ```bash
   #!/bin/bash
   modprobe msr
   rdmsr 0x1FC
   wrmsr 0x1FC value
   ```
3. Save and exit the editor (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Step 2: Make the Script Executable

Assign execution permissions to the script:
```sh
sudo chmod +x /usr/local/bin/disable_bd_prochot.sh
```

### Step 3: Create a Service File

1. Run the following command to create a new service file:
   ```sh
   sudo nano /etc/systemd/system/disable_bd_prochot.service
   ```
2. Insert the following content into the service file:
   ```ini
   [Unit]
   Description=Disable BD PROCHOT

   [Service]
   Type=oneshot
   ExecStart=/usr/local/bin/disable_bd_prochot.sh

   [Install]
   WantedBy=multi-user.target
   ```
3. Save and exit the editor.

### Step 4: Enable the Service

Enable the service to run at startup:
```sh
sudo systemctl enable disable_bd_prochot.service
```

### Step 5: Reboot

Reboot your system to verify the script executes at startup:
```sh
sudo reboot
```

## Usage

To disable BD PROCHOT automatically, you can use the provided script by running:
```bash
curl -LO https://raw.githubusercontent.com/subjec2change/Disable-BD-PROCHOT-on-Bazzite/refs/heads/main/Disable_BD_PROCHOT ; sudo bash Disable_BD_PROCHOT
```

## Automatic Script

For convenience, an automatic script is available. This script will perform all the necessary steps to disable BD PROCHOT on your system. Use the command shown in the [Usage](#usage) section to run it.

## Cautions

- The steps provided are indicative and might vary based on the Linux distribution you are using.
- Disabling BD PROCHOT can lead to potential overheating issues. Proceed with caution and ensure your data is backed up before making any changes.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
