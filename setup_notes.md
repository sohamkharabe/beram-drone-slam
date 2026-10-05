# Setup Notes & Troubleshooting

## Versions Installed
- **OS:** Ubuntu 22.04 (via WSL2 on Windows)[cite: 10]
- **Simulator:** Gazebo Harmonic[cite: 10]
- **ROS 2:** Humble Hawksbill[cite: 10]
- **Autopilot:** ArduPilot SITL[cite: 10]

## Problems Hit and Solutions

**1. Issue: SITL "Waiting for heartbeat" (Stuck on boot)**
- **Problem:** When running `sim_vehicle.py`, MAVProxy would start but wait for a heartbeat indefinitely. SITL failed to boot its background console.
- **Cause:** WSL2 does not come with the `xterm` terminal emulator pre-installed, which ArduPilot SITL uses by default to launch its background process.
- **Solution:** Installed the missing package manually: `sudo apt install xterm -y`.

**2. Issue: ArduPilot and Gazebo Connection Failure (Incorrect protocol magic)**
- **Problem:** Gazebo flooded the terminal with `[Wrn] [ArduPilotPlugin.cc:1575] Incorrect protocol magic 0 should be 18458` warnings, and the drone wouldn't connect.
- **Cause:** The default ArduPilot SITL command was using the legacy protocol instead of the modern JSON protocol expected by Gazebo Harmonic. Also, the `ardupilot_gazebo` plugin was not on the correct stable branch.
- **Solution:** 
  1. Switched `ardupilot_gazebo` to the `harmonic` branch and rebuilt the package using `colcon build`.
  2. Modified the SITL launch command to enforce the JSON protocol: Added the `--model JSON` parameter.

**3. Issue: PreArm: Motors: Check frame class and type**
- **Problem:** When attempting to arm the drone, MAVProxy threw a red error failing the arming sequence due to unknown frame class.
- **Cause:** Corrupt or missing default parameters in the SITL EEPROM after multiple crash loops.
- **Solution:** 
  1. Booted SITL with the `-w` flag (`sim_vehicle.py ... -w`) to wipe existing parameters.
  2. Manually forced the Quadcopter frame in MAVProxy terminal:
     `param set FRAME_CLASS 1`
     `param set FRAME_TYPE 1`
  3. Ran `reboot` in MAVProxy. After reboot, the drone armed successfully.
