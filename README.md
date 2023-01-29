# Disable-BD-PROCHOT-on-LINUX
This repository provides steps to disable BD PROCHOT at boot time in a Linux system.
## Prerequisites
- [msr-tools](https://github.com/Intel/msr-tools) installato
	- Arch Linux:
	```sudo pacman -S msr-tools```
	- Ubuntu and derivatives:
	```sudo apt-get install msr-tools```
	- Fedora and derivatives:
	```sudo dnf install msr-tools```
	- CentOS:
	```sudo yum install msr-tools```

## Steps
<b>1. Creates a script containing the desired commands:</b>
```sh
sudo nano /usr/local/bin/disable_bd_prochot.sh
```
<b>2.  Insert the following commands into the script:</b>
```sh
#!/bin/bash
modprobe msr
rdmsr 0x1FC
wrmsr 0x1FC value
```
<b>3.  Assign execution permissions to the script:</b>
```sh
sudo chmod +x /usr/local/bin/disable_bd_prochot.sh
```
<b>4.  Creates a service file to execute the script at start-up:</b>
```sh
sudo nano /etc/systemd/system/disable_bd_prochot.service
```
<b>5.  Insert the following content in the service file:</b>
```sh
[Unit]
Description=Disable BD PROCHOT

[Service]
Type=oneshot
ExecStart=/usr/local/bin/disable_bd_prochot.sh

[Install]
WantedBy=multi-user.target
```
<b>6.  Activate the service:</b>
```sh
sudo systemctl enable disable_bd_prochot.service
```
<b>7.  Reboot the system to verify that commands are executed at start-up.</b>
```sh
sudo reboot
```
## Cautions

-   These steps are indicative and may vary depending on the Linux distribution in use.
-   It is possible that the commands will not work or cause problems for the system, so be very cautious and back up your data before proceeding.
