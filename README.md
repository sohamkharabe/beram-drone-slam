# Drone SLAM & Obstacle Avoidance in Gazebo

## Overview
This repository contains the setup and implementation for a 3D drone simulation using ArduPilot SITL, ROS 2 Humble, and Gazebo Harmonic. The project demonstrates manual flight, SLAM-based 2D mapping using Cartographer, and obstacle avoidance in a walled 3D environment.

## Exact Setup Steps
The simulation runs on Ubuntu 22.04 via WSL2. 

1. **Install ROS 2 Humble**: Follow the official ROS 2 documentation for Ubuntu 22.04.
2. **Install Gazebo Harmonic**: Installed via official binaries[cite: 10].
3. **Setup ArduPilot SITL**: Cloned the ArduPilot repository and installed dependencies via `Tools/environment_install/install-prereqs-ubuntu.sh -y`[cite: 10].
4. **Setup ROS 2 Workspace (`ros2_ws`)**:
   - Cloned `ardupilot_gazebo`, `ardupilot_ros`, and related packages[cite: 10].
   - Switched `ardupilot_gazebo` to the `harmonic` branch to fix protocol mismatch errors.
   - Built the workspace using `colcon build --symlink-install`.

---

## How to Run Each Level

### Level 1: Fly in Simulation[cite: 10]
**Goal:** Arm the drone, take off, fly to a few points and land using a ground station[cite: 10].

**Terminal 1: Launch Gazebo 3D World**
```bash
cd ~
source ~/ros2_ws/install/setup.bash
gz sim -v4 -r ~/ros2_ws/src/ardupilot_gazebo/worlds/iris_runway.sdf
```

**Terminal 2: Launch Console
```
cd ~/ardupilot/ArduCopter
sim_vehicle.py -v ArduCopter -f gazebo-iris --model JSON --console
```
