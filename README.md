# Livox MID-360 LiDAR Setup Guide for ROS2

Complete installation and configuration guide for Livox MID-360 LiDAR with ROS2 on Ubuntu.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Network Discovery and Configuration](#network-discovery-and-configuration)
- [Livox SDK2 Installation](#livox-sdk2-installation)
- [ROS2 Driver Installation](#ros2-driver-installation)
- [Configuration, Launch, and Testing](#configuration-launch-and-testing)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## Prerequisites

### System Requirements
- Ubuntu 20.04 (ROS2 Foxy) or Ubuntu 22.04 (ROS2 Humble)
- ROS2 Desktop installation
- Ethernet port for LiDAR connection
- Root/sudo access

### Install Basic Dependencies
```bash
sudo apt update
sudo apt install -y cmake build-essential nmap ethtool net-tools
```

## Network Discovery and Configuration

### 1. Find Your Ethernet Interface
```bash
# List all network interfaces
ip link show

# Or
ifconfig

# Common ethernet interface names: enp89s0, eth0, enp3s0, etc.
# Note your ethernet interface name for later use
```

### 2. Initial Network Setup
```bash
# Replace 'enp89s0' with your actual ethernet interface name
export ETH_INTERFACE="enp89s0"  # Change this to match your system

# Bring interface up
sudo ip link set $ETH_INTERFACE up

# Assign temporary IP address
sudo ip addr add 192.168.1.50/24 dev $ETH_INTERFACE

# Verify interface is configured
ip addr show $ETH_INTERFACE
```

### 3. Discover LiDAR Device
```bash
# Scan the network for devices
echo "Scanning for devices on network..."
nmap -sn 192.168.1.0/24

# The output will show discovered devices, for example:
# Nmap scan report for 192.168.1.50    <- Your computer
# Nmap scan report for 192.168.1.169   <- Potential LiDAR device
```

### 4. Identify Livox LiDAR
```bash
# Test connectivity to potential LiDAR devices
# Default Livox IP is 192.168.1.12, but it may be different

# Try default IP first
ping -c 3 192.168.1.12

# If no response, try other discovered IPs
ping -c 3 192.168.1.169  # Replace with discovered IP
```

**Identifying Livox Device Characteristics:**
- Responds to ping consistently
- Usually has IP 192.168.1.12 (default) or custom configured IP
- TTL typically 255 in ping responses
- Manufacturer info may show "Livox" or "DJI" in network scans

### 5. Set Up Automatic Network Configuration (Recommended)

Create persistent network configuration using NetworkManager:

```bash
# Create persistent connection profile
sudo nmcli connection add \
    type ethernet \
    ifname $ETH_INTERFACE \
    con-name livox-lidar \
    ip4 192.168.1.50/24 \
    ipv4.method manual

# Enable automatic connection on boot and cable plug-in
sudo nmcli connection modify livox-lidar connection.autoconnect yes
sudo nmcli connection modify livox-lidar connection.autoconnect-priority 10

# Activate the connection
sudo nmcli connection up livox-lidar

# Verify configuration
nmcli connection show livox-lidar | grep -E "(autoconnect|ipv4.addresses)"
```

**Verify Automatic Configuration:**
```bash
# Test by disconnecting and reconnecting ethernet cable
# Or reboot system and check if interface comes up automatically
sudo reboot

# After reboot, verify:
ip addr show $ETH_INTERFACE | grep 192.168.1.50
ping -c 3 192.168.1.169  # Replace with your LiDAR IP
```

## Livox SDK2 Installation

### 1. Download and Build SDK
```bash
# Clone Livox SDK2
cd ~
git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd Livox-SDK2

# Create build directory and compile
mkdir build && cd build
cmake .. && make -j$(nproc)

# Install SDK system-wide
sudo make install

# Verify installation
ls -la /usr/local/lib/liblivox_lidar_sdk*
ls -la /usr/local/include/livox_lidar*
```

### 2. Configure Library Path
```bash
# Add library path permanently
echo 'export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/lib' >> ~/.bashrc
source ~/.bashrc

# Update library cache
sudo ldconfig

# Verify library is found
ldconfig -p | grep livox
```

### 3. Test SDK Installation
```bash
# Test with quick start sample
cd ~/Livox-SDK2/build/samples/livox_lidar_quick_start

# Create test config file
cat > test_config.json << 'EOF'
{
  "MID360": {
    "lidar_net_info" : {
      "cmd_data_port"  : 56100,
      "push_msg_port"  : 56200,
      "point_data_port": 56300,
      "imu_data_port"  : 56400,
      "log_data_port"  : 56500
    },
    "host_net_info" : [
      {
        "host_ip"        : "192.168.1.50",
        "multicast_ip"   : "224.1.1.5",
        "cmd_data_port"  : 56101,
        "push_msg_port"  : 56201,
        "point_data_port": 56301,
        "imu_data_port"  : 56401,
        "log_data_port"  : 56501
      }
    ]
  }
}
EOF

# Test connection (replace 192.168.1.169 with your LiDAR IP)
# This should show connection messages if working
sudo ./livox_lidar_quick_start test_config.json
```

## ROS2 Driver Installation

### 1. Create ROS2 Workspace
```bash
# Create workspace
mkdir -p ~/ws_livox/src
cd ~/ws_livox/src

# Clone ROS2 driver
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
```

### 2. Install ROS2 Dependencies
```bash
cd ~/ws_livox

# Source ROS2 environment
source /opt/ros/humble/setup.bash  # or /opt/ros/foxy/setup.bash

# Install dependencies
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

### 3. Build ROS2 Driver
```bash
# Build driver using provided script
./src/livox_ros_driver2/build.sh humble  # or ROS2 for Foxy

# Alternative manual build
colcon build --symlink-install

# Source workspace
source install/setup.bash

# Add to bashrc for automatic sourcing
echo "source ~/ws_livox/install/setup.bash" >> ~/.bashrc
```

## Configuration, Launch, and Testing

### 1. Update LiDAR Configuration File
```bash
# Edit the MID-360 configuration file
gedit ~/ws_livox/src/livox_ros_driver2/config/MID360_config.json
```

**Replace with your network settings:**
```json
{
  "lidar_summary_info" : {
    "lidar_type": 8
  },
  "MID360": {
    "lidar_net_info" : {
      "cmd_data_port": 56100,
      "push_msg_port": 56200,
      "point_data_port": 56300,
      "imu_data_port": 56400,
      "log_data_port": 56500
    },
    "host_net_info" : {
      "cmd_data_ip" : "192.168.1.50",      # Your host computer IP
      "cmd_data_port": 56101,
      "push_msg_ip": "192.168.1.50",       # Your host computer IP
      "push_msg_port": 56201,
      "point_data_ip": "192.168.1.50",     # Your host computer IP
      "point_data_port": 56301,
      "imu_data_ip" : "192.168.1.50",      # Your host computer IP
      "imu_data_port": 56401,
      "log_data_ip" : "",
      "log_data_port": 56501
    }
  },
  "lidar_configs" : [
    {
      "ip" : "192.168.1.169",              # Your LiDAR IP (discovered earlier)
      "pcl_data_type" : 1,
      "pattern_mode" : 0,
      "extrinsic_parameter" : {
        "roll": 0.0,
        "pitch": 0.0,
        "yaw": 0.0,
        "x": 0,
        "y": 0,
        "z": 0
      }
    }
  ]
}
```

### 2. Pre-flight Check
```bash
# Verify network connectivity
ping -c 3 192.168.1.169  # Replace with your LiDAR IP

# Check if ports are available
sudo netstat -ulnp | grep -E "(56100|56101|56200|56201|56300|56301)"

# Verify ROS2 environment
ros2 --version
echo $ROS_DOMAIN_ID  # Should be set or empty
```

### 3. Launch LiDAR Node
```bash
cd ~/ws_livox
source install/setup.bash

# Launch with RViz visualization
ros2 launch livox_ros_driver2 rviz_MID360_launch.py

# Alternative: Launch without RViz
ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

### 4. Verify Data Stream
```bash
# In a new terminal, check ROS2 topics
ros2 topic list

# Expected topics:
# /livox/lidar          # Point cloud data
# /livox/imu            # IMU data

# Monitor point cloud data
ros2 topic hz /livox/lidar

# View sample point cloud data
ros2 topic echo /livox/lidar --field width
```

### 5. RViz Configuration
If using RViz visualization:
- Set **Fixed Frame** to `livox_frame` in Global Options
- Add **PointCloud2** display
- Set **Topic** to `/livox/lidar`
- Adjust **Size** and **Color Transformer** as needed

## Troubleshooting

### Common Issues and Solutions

#### Network Issues
```bash
# Cannot discover LiDAR
# 1. Check ethernet cable connection
# 2. Verify interface is up
ip link show $ETH_INTERFACE

# 3. Try different IP ranges
nmap -sn 192.168.1.0/24
nmap -sn 192.168.0.0/24

# 4. Check for DHCP assignment
ip route show
```

#### Library Issues
```bash
# "liblivox_lidar_sdk_shared.so not found" error
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/lib
sudo ldconfig

# Verify library path
ldd ~/ws_livox/install/livox_ros_driver2/lib/livox_ros_driver2/livox_ros_driver2_node | grep livox
```

### Network Management Commands
```bash
# View all NetworkManager connections
nmcli connection show

# Modify connection if needed
sudo nmcli connection modify livox-lidar ipv4.addresses "192.168.1.51/24"

# Restart connection
sudo nmcli connection down livox-lidar
sudo nmcli connection up livox-lidar

# Remove connection (if needed)
sudo nmcli connection delete livox-lidar
```

### Launch Commands
```bash
# Quick launch
cd ~/ws_livox && source install/setup.bash && ros2 launch livox_ros_driver2 rviz_MID360_launch.py

# Check topics
ros2 topic list
ros2 topic hz /livox/lidar
```

## References
- [Livox SDK2 GitHub](https://github.com/Livox-SDK/Livox-SDK2)
- [Livox ROS Driver 2 GitHub](https://github.com/Livox-SDK/livox_ros_driver2)
- [MID-360 Documentation](https://livox-wiki-en.readthedocs.io/en/latest/tutorials/new_product/mid360/mid360.html)
- [NetworkManager Documentation](https://networkmanager.dev/)

---

**Important Notes:**
1. Replace `enp89s0` with your actual ethernet interface name throughout this guide
2. Replace `192.168.1.169` with your discovered LiDAR IP address
3. Ensure your LiDAR is properly powered and connected via Ethernet cable
4. This guide assumes Ubuntu 20.04/22.04 with ROS2 Foxy/Humble installed

**Support:**
- For SDK issues: [Livox SDK2 Issues](https://github.com/Livox-SDK/Livox-SDK2/issues)
- For ROS driver issues: [Livox ROS Driver 2 Issues](https://github.com/Livox-SDK/livox_ros_driver2/issues)
