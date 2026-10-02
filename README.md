# 🚗 SAEV — Smart Autonomous Electric Vehicle

A ROS 2-based **Smart Autonomous Electric Vehicle (SAEV)** prototype built for autonomous navigation in unfamiliar indoor environments. The system brings together **LiDAR-based SLAM, AMCL localization, QR-based waypoint identification, Nav2 path planning, obstacle avoidance, and embedded motor control** on a single vehicle platform.

> Presented at **IEEE SeFeT 2026, VNIT Nagpur**.

---

## 📑 Table of Contents

- [Key Features](#-key-features)
- [System Architecture](#️-system-architecture)
- [Hardware](#-hardware)
- [Software Stack](#-software-stack)
- [How It Works](#️-how-it-works)
- [Repository Structure](#-repository-structure)
- [Getting Started](#️-getting-started)
- [Future Work](#-future-work)
- [License](#-license)
- [Author](#-author)

---

## 🚀 Key Features

- 🗺️ **2D mapping** of unknown environments with SLAM Toolbox
- 📍 **Localization** on a saved map using AMCL
- 📡 **LiDAR perception** for mapping and obstacle detection (RPLIDAR S2)
- 📷 **Two-stage QR pipeline** — YOLOv8 detects, PyZBar decodes
- 🧭 **QR-based waypoint navigation** and re-localization
- 🛣️ **Path planning and control** with ROS 2 Nav2
- 🚧 **Dynamic obstacle avoidance** via Nav2 costmaps
- ⚙️ **Wheel-encoder odometry** feedback
- 🔌 **Embedded motor control** on Arduino Uno / ESP32 with an L298N driver

---

## 🏗️ System Architecture

```text
                         12V Battery
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
        DC-DC Converter                 Motor Driver
          12V → 5V                         (L298N)
               │                             │
               ▼                             ▼
        UBEC (5V / 3A)                   DC Motors
               │                             │
               ▼                             ▼
   Onboard Computer                    Wheel Encoders
 (Intel NUC / Raspberry Pi)                  │
               │                             │
     ┌─────────┼──────────┐                  │
     ▼         ▼          ▼                  │
   LiDAR    Camera    Arduino / ESP32 ◄──────┘
                          │
                          └──► PWM / direction → L298N

  Software layer (ROS 2 Humble)
  ┌──────────────┬──────────────┬──────────────┬──────────────┐
  │ SLAM Toolbox │     AMCL     │     Nav2     │  QR Pipeline │
  │  (mapping)   │ (localizing) │  (planning)  │ YOLOv8+PyZBar│
  └──────────────┴──────────────┴──────┬───────┴──────────────┘
                                       ▼
                               /cmd_vel → Vehicle Control → Motors
```

---

## 🔧 Hardware

| Category   | Components                                   |
|------------|----------------------------------------------|
| Computing  | Intel NUC, Raspberry Pi, Arduino Uno / ESP32 |
| Sensors    | RPLIDAR S2, camera module, wheel encoders    |
| Actuation  | L298N motor driver, DC motors                |
| Power      | 12V battery, DC-DC converter, 5V / 3A UBEC   |

---

## 🧠 Software Stack

| Technology       | Purpose                                     |
|------------------|---------------------------------------------|
| **ROS 2 Humble** | Communication and system integration        |
| **SLAM Toolbox** | Building the 2D occupancy grid map          |
| **AMCL**         | Estimating robot pose on the saved map      |
| **Nav2**         | Global/local planning and obstacle avoidance|
| **YOLOv8**       | Detecting QR codes in the camera feed       |
| **PyZBar**       | Decoding the detected QR codes              |
| **Python / C++** | ROS 2 nodes and embedded firmware           |

---

## ⚙️ How It Works

### 1. Mapping

The vehicle is driven through an unfamiliar environment while SLAM Toolbox fuses LiDAR scans with wheel odometry to produce a **2D occupancy grid** of free, occupied and unknown cells. The map is saved for later navigation.

```text
LiDAR Scans + Wheel Odometry → SLAM Toolbox → 2D Occupancy Grid Map
```

### 2. Localization

With a saved map, **AMCL (Adaptive Monte Carlo Localization)** uses a particle filter over LiDAR and odometry data to estimate the vehicle's pose.

```text
Saved Map + LiDAR + Odometry → AMCL → Robot Pose (x, y, θ)
```

### 3. QR Detection & Waypoints

QR codes placed in the environment act as identifiable waypoints. A two-stage pipeline keeps decoding fast and reliable:

```text
Camera Feed → YOLOv8 (detect) → Crop QR Region → PyZBar (decode) → Waypoint Goal
```

The decoded data maps to a navigation point, which is sent to Nav2 as a goal.

### 4. Autonomous Navigation

```text
Robot Pose + Navigation Goal → Nav2 → Path Plan → /cmd_vel → Vehicle Controller → Motors
```

### 5. Obstacle Avoidance

LiDAR data continuously updates Nav2's costmaps. When a new obstacle appears, the local planner adjusts the path and the vehicle keeps moving toward its goal.

```text
LiDAR → Costmap Update → Path Adjustment → Updated Velocity Commands
```

### 6. QR-Based Re-Localization

If localization confidence drops, the vehicle can use a known QR marker as a reference point.

```text
Re-localization Needed → Navigate to Nearest QR → Detect & Decode → Update Pose Reference → Resume
```

### 7. Vehicle Control

```text
Nav2 (/cmd_vel) → Onboard Computer → Arduino / ESP32 → L298N → DC Motors
                                            ▲
                       Wheel Encoders ──────┘  (odometry feedback)
```

### End-to-End Workflow

```text
                     START → Power On
                            │
                    Environment Mapped?
                     /              \
                   No                Yes
                   │                  │
            Mapping Phase        Load Saved Map
         (SLAM Toolbox)         + AMCL Localization
                   │                  │
                   └────────┬─────────┘
                            ▼
              Navigation Goal (from QR / user)
                            │
                            ▼
                          Nav2
                            │
                      Obstacle? ──Yes──► Path Adjustment
                            │                  │
                           No ◄────────────────┘
                            │
                      Goal Reached?
                       /         \
                     No           Yes
                     │             │
              Continue Nav    Task Complete
```

---

## 📂 Repository Structure

This repository is a ROS 2 `ament_cmake` package named **`my_bot`**.

```text
my_bot/
├── config/          # Nav2, SLAM Toolbox, AMCL and controller parameters
├── description/     # Robot URDF / xacro description
├── launch/          # Launch files
├── worlds/          # Simulation worlds
├── CMakeLists.txt
├── package.xml
├── LICENSE
└── README.md
```

---

## 🛠️ Getting Started

### Prerequisites

- Ubuntu 22.04 with **ROS 2 Humble**
- Python 3

Install ROS 2 dependencies:

```bash
sudo apt update
sudo apt install ros-humble-navigation2 ros-humble-nav2-bringup \
                 ros-humble-slam-toolbox ros-humble-xacro libzbar0
```

Install Python dependencies:

```bash
pip install ultralytics pyzbar opencv-python
```

Set up the RPLIDAR driver (ROS 2 branch of `rplidar_ros`) in the same workspace.

### Build

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone <your-repo-url> my_bot
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

### Run

Launch each subsystem in its own terminal (source the workspace in each):

```bash
# 1. LiDAR
ros2 launch rplidar_ros rplidar_s2_launch.py

# 2. Mapping (first run in a new environment)
ros2 launch slam_toolbox online_async_launch.py \
    slam_params_file:=src/my_bot/config/mapper_params_online_async.yaml

# Save the map once mapping is complete
ros2 run nav2_map_server map_saver_cli -f ~/maps/my_map

# 3. Localization on the saved map
ros2 launch nav2_bringup localization_launch.py map:=~/maps/my_map.yaml

# 4. Navigation
ros2 launch nav2_bringup navigation_launch.py
```

Then start the QR detection node and the vehicle-control bridge to the Arduino/ESP32.

> Adjust file and node names above to match the launch files and configs in your `launch/` and `config/` directories.

---

## 🔮 Future Work

- Higher localization accuracy in feature-poor areas
- More robust QR-based navigation under poor lighting
- Better dynamic obstacle handling
- Additional sensor fusion (e.g. IMU)
- Advanced path-planning strategies
- Closed-loop PID tuning for low-level motor control
- Real-time telemetry dashboard
- Simulation and hardware-in-the-loop testing

---

## 📄 License

Distributed under the **Apache License 2.0**. See [`LICENSE`](LICENSE) for details.

---

## 👩‍💻 Author

**Anshu**
B.E. Electronics & Communication Engineering
Thapar Institute of Engineering & Technology, Patiala
