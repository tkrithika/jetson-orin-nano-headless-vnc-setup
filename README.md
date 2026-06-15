<div align="center">

![NVIDIA Jetson](https://img.shields.io/badge/NVIDIA_Jetson-76B900?style=for-the-badge&logo=nvidia&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![SSH](https://img.shields.io/badge/SSH-4D4D4D?style=for-the-badge&logo=openssh&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)

</div>

# Jetson Orin Nano – Headless GUI Access via VNC

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Network Access and SSH Validation](#2-network-access-and-ssh-validation)
3. [VNC Server Installation and Authentication](#3-vnc-server-installation-and-authentication)
4. [Ensuring Graphical Target Mode](#4-ensuring-graphical-target-mode)
5. [Forcing a Virtual Display Using EDID Injection](#5-forcing-a-virtual-display-using-edid-injection)
6. [Xorg Configuration for Headless GPU Acceleration](#6-xorg-configuration-for-headless-gpu-acceleration)
7. [Desktop Auto-Login Configuration](#7-desktop-auto-login-configuration)
8. [Validating Display Initialization](#8-validating-display-initialization)
9. [Manual VNC Server Test](#9-manual-vnc-server-test)
10. [Persistent Service Creation (systemd)](#10-persistent-service-creation-systemd)
11. [Verification and Troubleshooting](#11-verification-and-troubleshooting)


## 1. System Architecture Overview

A headless Jetson Orin Nano operates without a directly attached monitor, keyboard, or mouse. 
However, many development workflows require access to the full Ubuntu desktop environment.

To enable remote graphical access, this setup uses:

- **Xorg** as the display server
- **x11vnc** to mirror the active X session
- **TigerVNC** as the client viewer
- A forced **EDID configuration** to simulate a connected display

Unlike dummy display drivers, this approach keeps the native NVIDIA GPU driver active, preserving hardware acceleration and proper resolution handling.

The VNC server attaches to display `:0`, which represents the primary graphical session managed by Xorg.

---

## 2. Network Access and SSH Validation

Before configuring remote desktop access, confirm that the Jetson Orin Nano is reachable over the local network.

### 2.1 Determine the Jetson IP Address

If the Jetson is temporarily connected to a display, retrieve its IP address using:

```bash
hostname -I
```
For detailed interface information:
```
ip a
```

### 2.2 Establish SSH Connection

From your client machine (Linux/macOS terminal or Windows PowerShell):
```
ssh <username>@<jetson-ip-address>
```
Example:
```
ssh rytle@172.20.10.3
```
Successful SSH login confirms:

Network connectivity
Proper user authentication
Remote administrative access

---

## 3. VNC Server Installation and Authentication

The system will use **x11vnc**, which attaches to the active X session and mirrors the existing desktop environment rather than creating a separate virtual session.

### 3.1 Install x11vnc

Update package lists and install the VNC server:

```bash
sudo apt update
sudo apt install x11vnc -y
```

### 3.2 Configure VNC Authentication

Set a password to secure remote access:
```
x11vnc -storepasswd
```
You will be prompted to enter and confirm a password.
This password will be required when connecting from the VNC client.

The password file is typically stored in:
```
~/.vnc/passwd
```

---

## 4. Ensuring Graphical Target Mode

For VNC to mirror an active desktop session, the system must boot into a graphical environment rather than a multi-user (CLI-only) mode.

### 4.1 Verify Current Boot Target

Check the system's default boot target:

```bash
systemctl get-default
```

If the output returns:
graphical.target

the system is already configured to launch the graphical desktop environment at boot.

### 4.2 Set Graphical Target (If Required)

If the system reports multi-user.target, switch it to graphical mode:
```
sudo systemctl set-default graphical.target
sudo reboot
```
After reboot, the display manager (GDM) should initialize the Xorg session automatically.

This step ensures that display :0 is created during system startup, which is required for x11vnc to attach successfully.

---

## 5. Forcing a Virtual Display Using EDID Injection

When operating headless, the Jetson does not detect a physical monitor.  
Without a detected display, Xorg may:

- Fail to initialize properly  
- Default to a very low resolution (e.g., 640×480)  
- Prevent GPU-accelerated desktop rendering  

To maintain full hardware acceleration, a simulated display must be presented to the NVIDIA driver.

---

### 5.1 Why EDID Injection Is Preferred

A dummy software display driver can be used to emulate a monitor.  
However, this approach disables hardware acceleration and may reduce performance.

A better solution is to inject a valid **EDID (Extended Display Identification Data)** file.

An EDID file contains display capability information such as:

- Supported resolutions  
- Refresh rates  
- Timing parameters  

By forcing the NVIDIA driver to load an EDID file, the system behaves as if a real monitor is connected, while preserving GPU acceleration.

### 5.2 Obtaining an EDID File

A valid EDID binary file is required to simulate a connected monitor.

Pre-collected EDID files for various display models are available from public repositories such as:

- https://github.com/linuxhw/EDID

Download a suitable `.bin` file that matches your desired resolution 
(e.g., 1920×1080 @ 60Hz).

After downloading the EDID `.bin` file to your local machine, transfer it to the Jetson using SCP.

From your client machine:

```bash
scp <EDID-file>.bin <username>@<jetson-ip>:/tmp
```

### 5.3 Install the EDID File on Jetson

Create the firmware directory (if not already present):
```
sudo mkdir -p /lib/firmware/edid
```
Copy the EDID file:
```
sudo cp /tmp/EDID.bin /lib/firmware/edid/EDID.bin
```
Verify installation:
```
ls -lh /lib/firmware/edid/
```
Expected output:
```
-rw-r--r-- 1 root root 271K EDID.bin
```
The EDID file is now correctly placed and ready to be referenced by the Xorg configuration.

---

## 6. Xorg Configuration for Headless GPU Acceleration

After installing the EDID file, Xorg must be explicitly instructed to use it.  
This ensures that the NVIDIA driver initializes a virtual display with the desired resolution while preserving GPU acceleration.

---

### 6.1 Create or Edit the Xorg Configuration File

Open (or create) the Xorg configuration file:

```bash
sudo nano /etc/X11/xorg.conf
```

### 6.2 Configure the NVIDIA Device Section

Insert the following configuration block:

```
Section "Device"  
 Identifier  "Tegra0"  
 Driver      "nvidia"  
 Option      "PrimaryGPU" "Yes"  
 Option      "AllowEmptyInitialConfiguration" "true"  
 Option      "ConnectedMonitor" "DFP-0"  
 Option      "CustomEDID" "DFP-0:/lib/firmware/edid/EDID.bin"  
 Option      "UseDisplayDevice" "DFP-0"  
 Option      "IgnoreEDIDChecksum" "DFP-0"  
 Option      "AllowNonEdidModes" "true"  
EndSection  
```

---

### 6.3 Define Monitor and Screen Sections

Append the following sections below the device configuration:
```
Section "Monitor"  
 Identifier "Monitor0"  
 VendorName "VirtualDisplay"  
 ModelName  "Headless1920x1080"  
EndSection  

Section "Screen"  
 Identifier "Screen0"  
 Device     "Tegra0"  
 Monitor    "Monitor0"  
 DefaultDepth 24  
 SubSection "Display"  
  Depth 24  
  Modes "1920x1080"  
 EndSubSection  
EndSection  
```

Save the file

•	Press Ctrl + O, then Enter

•	Press Ctrl + X

---

### 6.4 Configuration Explanation

This configuration performs the following:

- Forces the NVIDIA driver to initialize even without a physical monitor  
- Associates the injected EDID file with display output `DFP-0`  
- Defines a virtual monitor profile  
- Sets the default resolution (example: 1920×1080)  
- Preserves hardware acceleration  

---

## 7. Desktop Auto-Login Configuration

For x11vnc to mirror the primary desktop session (`:0`), a full user session must be started automatically during boot.

If auto-login is not enabled, the system may stop at the graphical login screen (GDM greeter). In that state, the user session is not fully initialized, and x11vnc may fail to attach correctly.

---

### 7.1 Modify GDM Configuration

Edit the GDM custom configuration file:

```bash
sudo nano /etc/gdm3/custom.conf
```

Locate the [daemon] section and configure it as follows:
```
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=<your-username>
```
Replace <your-username> with the actual Linux user account on the Jetson.

Save the file and exit.

### 7.2 Reboot the System

Apply the changes:
```
sudo reboot
```
After reboot, the system should:

Automatically log into the desktop environment

Initialize the graphical session on display :0

Allow x11vnc to attach without manual intervention

### 7.3 Important Security Consideration

Enabling auto-login removes the requirement for local password authentication at boot.

This configuration is acceptable for:

Development environments

Lab setups

Controlled internal networks

However, for production or publicly accessible systems, auto-login should be disabled and alternative secure remote access mechanisms should be considered.

---

## 8. Validating Display Initialization

After rebooting the system, confirm that the graphical session is properly initialized and that the injected EDID configuration is active.

---

### 8.1 Verify Active Display Session

From the SSH session, run:

```bash
DISPLAY=:0 xrandr
```

This command queries the X server attached to display :0.

If the configuration is correct, the output should show:

An active display (e.g., DP-0 or HDMI-0)

The configured resolution (e.g., 1920x1080)

Available refresh rates

Example output snippet (In my case):
```
Screen 0: minimum 8 x 8, current 1920 x 1080, maximum 32767 x 32767
DP-0 connected primary 1920x1080+0+0
   1920x1080     60.00*
```
The presence of a valid resolution confirms:

Xorg successfully initialized

The EDID file was applied

The NVIDIA driver is functioning correctly

A virtual monitor is active

### 8.2 Troubleshooting Display Issues

If xrandr returns:

Can't open display :0 → The graphical session is not active

Very low resolution (e.g., 640x480) → EDID may not be applied correctly

No connected display → Xorg configuration needs review

At this stage, the system should be running a full desktop environment internally, even without a physical monitor connected.

---

## 9. Manual VNC Server Test

Before configuring automatic startup, verify that the VNC server can attach to the active X session manually.

---

### 9.1 Start x11vnc Manually

From the SSH session on the Jetson, run:

```bash
x11vnc -usepw -forever -display :0
```

If successful, the terminal should show output indicating:

Display :0 detected

Framebuffer initialized

Listening on TCP port 5900

Example snippet:
```
The VNC desktop is:  <hostname>:0
PORT=5900
```

### 9.2 Connect from the Client Machine

On your laptop or workstation, install a VNC viewer if not already installed.

For Ubuntu-based systems:
```
sudo apt install tigervnc-viewer -y
```
Then connect using:
```
vncviewer <jetson-ip>:5900
```
Example:
```
vncviewer 172.20.10.3:5900
```
When prompted, enter the password configured earlier.

If successful, the Jetson desktop should appear on your client machine.

### 9.3 Confirm Proper Operation

Verify:

Desktop loads correctly

Resolution matches configured EDID

Mouse and keyboard input function normally

GPU acceleration remains active

If the connection fails, ensure:

Port 5900 is not blocked by firewall

x11vnc is still running

The graphical session on :0 is active

---

## 10. Persistent Service Creation (systemd)

After confirming that x11vnc works manually, configure it to start automatically during system boot using a systemd service.

---

### 10.1 Create the Service File

Create a new service definition:

```bash
sudo nano /etc/systemd/system/x11vnc.service
```

Insert the following configuration and replace <your-username> accordingly:
```
[Unit]
Description=x11vnc Headless Service
Requires=display-manager.service
After=display-manager.service

[Service]
Type=simple
User=<your-username>
ExecStart=/usr/bin/x11vnc -auth guess -display WAIT:0 -loop -usepw -forever -rfbport 5900
Restart=on-failure

[Install]
WantedBy=graphical.target
```

### 10.2 Enable the Service

Enable automatic startup:
```
sudo systemctl enable x11vnc.service
```
Reload systemd configuration:
```
sudo systemctl daemon-reload
```
Restart the service manually for testing:
```
sudo systemctl restart x11vnc.service
```

### 10.3 Verify Service Status

Check that the service is running:
```
systemctl status x11vnc.service
```
If the output shows:
```
Active: active (running)
```
the VNC server will now start automatically at every system boot.

### 10.4 Final Reboot Test

Reboot the system to confirm persistence:
```
sudo reboot
```
After reboot, attempt to connect directly using:
```
vncviewer <jetson-ip>:5900
```
If the desktop appears without manually starting x11vnc, the configuration is complete.

---

## 11. Verification and Troubleshooting

This section provides systematic checks to validate the configuration and diagnose common issues.

---

### 11.1 Service Status Check

Verify that the x11vnc service is active:

```bash
systemctl status x11vnc.service
```

Expected output:
```
Active: active (running)
```
If inactive, restart:
```
sudo systemctl restart x11vnc.service
```
Next Reboot:
```
sudo reboot
```

### 11.2 Confirm Display Availability

Ensure that the graphical session is active:
```
DISPLAY=:0 xrandr
```
If you receive:
```
Can't open display :0
```
Possible causes:

Graphical target not enabled

Auto-login not configured

Xorg failed to start

Check default boot target:
```
systemctl get-default
```

### 11.3 Verify Port 5900 Is Listening

Confirm that the VNC server is bound to port 5900:
```
sudo ss -tulnp | grep 5900
```
You should see an entry indicating that x11vnc is listening.

If nothing appears:

The service may not be running

A firewall rule may be blocking the port


### 11.4 Establishing the VNC Client Connection

Now go to TigerVNC and type:
```
<your ip address>:5900
```
The jetson screen will appear on your laptop, even with the monitor disconnected

---

### 11.5 Quick Reconnect Procedure (Headless Mode)

If the Jetson is operating without a monitor and you need to reconnect from your laptop:

1. (If required) Transfer the EDID file again from your local machine:

```bash
scp <EDID-file>.bin <username>@<jetson-ip>:/tmp
```

2. Open TigerVNC or Launch the VNC client from your laptop:
```
vncviewer <jetson-ip>:5900
```

3. Enter the configured VNC password when prompted.

If the systemd service is active, the Jetson desktop should load immediately, even without a physical monitor connected.

---


## Disclaimer

This project is an independent technical guide and is not affiliated with or endorsed by NVIDIA Corporation.



























