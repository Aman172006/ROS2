# Automated ROS 2 & MAVProxy Setup Guide for Nvidia Jetson

This document outlines the complete process for setting up an Nvidia Jetson to automatically launch a MAVProxy bridge, followed by a ROS 2 (Jazzy) mission management node as soon as the system receives power.

## Overview

Because `systemd` runs in the background as a blank slate (it does not load standard user environments like `.bashrc`), we must explicitly define the environment, wait for network connections, and manage the execution order.

We split this into two main components:

1. **MAVProxy Service**: Starts the communication bridge with the flight controller.
2. **ROS 2 Service & Script**: Waits for MAVProxy to start, loads the ROS 2 environment, and launches the mission code.

---

# Step 1: Create the MAVProxy systemd Service

Before ROS 2 can listen for telemetry, MAVProxy must be routing the flight controller data to the local network ports.

## 1. Create the file

```bash
sudo nano /etc/systemd/system/mavproxy.service
```

## 2. Paste the configuration

```ini
[Unit]
Description=MAVProxy MAVLink Routing Bridge
After=network.target

[Service]
Type=simple
User=jetson

# Note: Replace this with your EXACT mavproxy command if different
ExecStart=/usr/local/bin/mavproxy.py \
  --master=/dev/ttyACM0 \
  --out=udp:127.0.0.1:14550 \
  --out=udp:127.0.0.1:14551

Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

## Explanation

### `After=network.target`

Ensures the Jetson's local network interfaces are initialized so the UDP ports can be opened.

### `User=jetson`

Runs the service as your standard user, preventing root permission issues.

---

# Step 2: Create the ROS 2 Launch Script

We use a bash script to prepare the workspace environment before running the ROS 2 command.

> **Note:** This file is placed in your home directory, NOT in a root system directory.

## 1. Create the script

```bash
nano ~/start_mission.sh
```

## 2. Paste the configuration

```bash
#!/bin/bash

# Wait 15 seconds to ensure MAVProxy negotiates the serial connection with the flight controller
sleep 15

# Source the core ROS 2 Jazzy environment
source /opt/ros/jazzy/setup.bash

# Source your local AAE_edge workspace
source /home/jetson/AAE_edge/install/setup.bash

# Launch the mission management code
ros2 launch mission_management main.launch.py \
  fcu_url:="udp://:14550@127.0.0.1:14551" \
  odcl_agl_override:=25.0 \
  task_sequence:=sentry,airdrop \
  sentry_max_duration_s:=160.0 \
  sentry_battery_min_pct:=30.0
```

## 3. Make the script executable

```bash
chmod +x ~/start_mission.sh
```

## Explanation

### `sleep 15`

Gives the flight controller and MAVProxy time to physically connect and begin streaming data.

### `source /opt/ros/jazzy/setup.bash`

Crucial step. `systemd` does not load your `.bashrc`. Without this line, the system won't know what `ros2` is or where its libraries are located.

---

# Step 3: Create the ROS 2 systemd Service

This system service triggers your bash script but specifically waits for MAVProxy to be running first.

## 1. Create the file

```bash
sudo nano /etc/systemd/system/ros2_mission.service
```

## 2. Paste the configuration

```ini
[Unit]
Description=ROS 2 Automatic Mission Management
After=network.target network-online.target mavproxy.service
Requires=mavproxy.service

[Service]
Type=simple
User=jetson
WorkingDirectory=/home/jetson/AAE_edge

ExecStart=/bin/bash /home/jetson/start_mission.sh

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Explanation

### `After=network-online.target`

Waits for DDS communication protocols (which rely on IP addresses) to be fully available.

### `Requires=mavproxy.service` & `After=mavproxy.service`

Enforces strict dependency order. ROS 2 cannot start unless MAVProxy is actively running.

### `Restart=on-failure`

If the mission code crashes, the system will wait 5 seconds (`RestartSec=5`) and automatically restart it.

### `WorkingDirectory=...`

Sets the context folder to your workspace before running the script.

---

# Step 4: Enable and Start Services

Tell Ubuntu to recognize your new files and run them on boot.

## Reload systemd daemon

```bash
sudo systemctl daemon-reload
```

## Enable both services to start automatically on boot

```bash
sudo systemctl enable mavproxy.service
sudo systemctl enable ros2_mission.service
```

---

# Useful Commands for Debugging

Since the code runs in the background, you will not see standard terminal output. Use these commands to monitor or control your drone's software.

## View live logs of ROS 2 (Ctrl+C to exit)

```bash
journalctl -u ros2_mission.service -f
```

## View live logs of MAVProxy

```bash
journalctl -u mavproxy.service -f
```

## Check the status (running, failed, etc.)

```bash
systemctl status ros2_mission.service
```

## Manually stop the automation (e.g., to run tests manually)

```bash
sudo systemctl stop ros2_mission.service
sudo systemctl stop mavproxy.service
```
