# Drone SLAM & Obstacle Avoidance in Gazebo

ArduPilot SITL + Gazebo Harmonic + ROS 2 Humble (WSL2, Ubuntu 22.04).
A lidar-equipped Iris quadcopter flies in a 3D maze, builds a 2D map with Cartographer, and flies to a target while avoiding an obstacle.

**Demo video:** TODO (paste YouTube Unlisted / Google Drive link)

See `setup_notes.md` for versions, problems hit and how they were solved.

## Repository contents

| File | Purpose |
|---|---|
| `soham_maze.launch.py` | Starts Gazebo server + GUI with the maze world and spawns `iris_with_lidar` at (0, 0, 0.5) |
| `scripts/avoid_to_target.py` | Level 3: lidar-based reactive obstacle avoidance |
| `map.png`, `map.yaml` | Level 2: saved 2D SLAM map |
| `setup_notes.md` | Versions, problems, fixes |
| `report.pdf` | Research report incl. AWS study (TODO) |

## Setup (summary)

1. Ubuntu 22.04 (WSL2), ROS 2 Humble, Gazebo Harmonic, ArduPilot SITL built in `~/ardupilot`.
2. Workspace `~/ros2_ws` containing `ardupilot_gz`, `ardupilot_gazebo`, `ardupilot_ros`, and `ros_gz` (humble branch) **built from source with `GZ_VERSION=harmonic`** (the apt `ros_gz` targets Fortress and will not load the drone).
3. Create `~/env.sh` and `source ~/env.sh` in every terminal:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash 2>/dev/null
export GZ_VERSION=harmonic
export GZ_RENDERING_ENGINE=ogre
```

4. Extra packages: `ros-humble-cartographer-ros`, `ros-humble-nav2-map-server`, `imagemagick`, `pip3 install --user pymavlink`.

## Level 1: Fly in simulation

TODO: add the commands you used for Level 1 (SITL + Gazebo, arm, take off, fly to points, land).

## Level 2: SLAM mapping

Start Gazebo first, SITL second. Use one terminal per step.

**Terminal 1: Gazebo, maze and drone**
```bash
source ~/env.sh
ros2 launch ardupilot_gz_bringup soham_maze.launch.py
```

**Terminal 2: ArduPilot SITL** (wait until the prompt shows `STABILIZE>`)
```bash
source ~/env.sh
cd ~/ardupilot
sim_vehicle.py -v ArduCopter -f gazebo-iris --model JSON --map --console
```
At the MAVProxy prompt: `output add 127.0.0.1:14560` (needed for Level 3).

**Terminal 3: bridge lidar and odometry into ROS 2**
```bash
source ~/env.sh
ros2 run ros_gz_bridge parameter_bridge "/lidar@sensor_msgs/msg/LaserScan[gz.msgs.LaserScan" "/model/iris/odometry@nav_msgs/msg/Odometry[gz.msgs.Odometry" "/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock" --ros-args -r /lidar:=/scan -r /model/iris/odometry:=/odometry
```

**Terminal 4: static frames, then Cartographer**
```bash
source ~/env.sh
ros2 run tf2_ros static_transform_publisher --x 0 --y 0 --z 0 --roll 0 --pitch 0 --yaw 0 --frame-id base_link --child-frame-id imu_link &
ros2 run tf2_ros static_transform_publisher --x 0 --y 0 --z 0 --roll 0 --pitch 0 --yaw 0 --frame-id base_link --child-frame-id base_scan &
sleep 2
ros2 launch ardupilot_cartographer cartographer.launch.py
```
In RViz set Fixed Frame to `map`.

**Fly** (MAVProxy prompt): `mode guided`, `arm throttle`, `takeoff 2`, then move slowly through the maze with `guided` commands at about 2 m height (a 2D lidar only scans one slice, and the scan rate is only about 5 Hz).

**Save the map**
```bash
source ~/env.sh
ros2 run nav2_map_server map_saver_cli -f ~/map
convert ~/map.pgm ~/map.png
```

What each piece does:
- `soham_maze.launch.py`: runs Gazebo (server and GUI) with `maze.sdf` and spawns the drone.
- `sim_vehicle.py`: runs the real ArduCopter autopilot code on the PC and connects to Gazebo's ArduPilot plugin.
- `parameter_bridge`: copies Gazebo topics into ROS 2 (`/scan`, `/odometry`, `/clock`).
- `static_transform_publisher`: gives Cartographer the fixed links between `base_link`, `imu_link` and `base_scan`.
- `cartographer.launch.py`: runs Cartographer (graph-based 2D lidar SLAM) and an occupancy-grid node that publishes `/map`.

## Level 3: Obstacle avoidance

**Approach:** own lidar-based reactive logic (`scripts/avoid_to_target.py`).
The target is a point 6 m ahead of the drone's starting heading. On every scan the script tests directions from -90 to +90 degrees in 15-degree steps. A direction counts as free if nothing is closer than `SAFE` within a window of `HALF` degrees either side. It picks the free direction closest to the target direction and sends a body-frame velocity command over MAVLink. Velocity commands keep the height fixed, which keeps the 2D scan consistent.

**Why this choice:** ArduPilot's built-in avoidance needs the lidar fed to the autopilot as proximity data (not set up here), and a ROS 2 planner such as Nav2 needs more configuration than time allowed. The own logic only needs `/scan` and is easy to explain. It is a local avoider, not a global planner.

**Run** (after Level 2 steps 1 to 3, with the drone hovering at 2 m):

Place the obstacle (once):
```bash
source ~/env.sh
gz service -s /world/maze/create --reqtype gz.msgs.EntityFactory --reptype gz.msgs.Boolean --timeout 5000 --req 'sdf: "<sdf version=\"1.9\"><model name=\"obstacle\"><static>true</static><pose>4 0 1.5 0 0 0</pose><link name=\"l\"><collision name=\"c\"><geometry><box><size>0.5 1.5 3</size></box></geometry></collision><visual name=\"v\"><geometry><box><size>0.5 1.5 3</size></box></geometry><material><ambient>1 0 0 1</ambient><diffuse>1 0 0 1</diffuse></material></visual></link></model></sdf>"'
```
Run the avoidance:
```bash
source ~/env.sh
python3 scripts/avoid_to_target.py
```
Land afterwards with `mode land` at the MAVProxy prompt.

Parameters (top of the script): `TARGET_FWD` 6 m, `SPEED`, `SAFE`, `TOL` 0.5 m, `HALF`.

## Results and limits

**What works**
- Level 1: TODO (confirm).
- Level 2: the drone flies in the maze and Cartographer builds a 2D map (290 x 320 cells at 0.05 m/pixel), saved as `map.png`.

**What does not work or is uncertain**
- Level 3: TODO. First attempt: the drone hit the obstacle and fell (cause not confirmed). Parameters were then changed (`SAFE` 2.5, `SPEED` 0.4, `HALF` 30 degrees, obstacle 4 m ahead). Write the result of the retry here, whatever it was.
- The `time moved backwards` warnings from SITL persist under WSL.
- Static frame offsets are set to zero, so the lidar is assumed to sit at the drone's centre.
- The scan rate is only about 5 Hz, so the drone must fly slowly.
- The lidar is configured with a vertical field of view but bridged as a single flat scan, and a 2D lidar cannot see obstacles above or below its slice.
- The avoider is reactive only: it can get stuck in dead ends and has no map-based planning.

**What I would do next**
- Use the real lidar offset in the TF frames.
- Raise the lidar rate and test a larger safety window systematically (bonus experiment table).
- Add a global planner (Nav2 with the saved map) on top of the local avoidance.
- Try ArduPilot's built-in BendyRuler avoidance using the lidar as a proximity sensor.
