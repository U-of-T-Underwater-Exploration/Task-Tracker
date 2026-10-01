# uuv-code: New Feature Architecture

This document describes the nodes that implement the Task-Tracker items. It uses the same format as `uuv-code_dev_nodes.md`.

Each node is marked with one of three statuses:
- **dev:** already merged into `dev`.
- **branch:** in development on a `feature/*` branch.
- **new:** to be written.

Where a feature branch already covers part of a task, the design builds on that branch rather than adding a new node.

**Conventions (apply to every node)**
- Units are SI: m, m/s, rad, N, N·m, Pa. Duty cycles and normalised commands are fractions shown as {%} (0-1 or -1 to 1), as in the existing doc.
- Frames follow [REP-105](https://www.ros.org/reps/rep-0105.html): `map → odom → base_link`. Sensor frames must match the link names in `uuv.urdf.xacro`: `imu_link`, `compass_link`, `baro_ext_link` and `cam_link` exist already. `dvl_link` and `sonar_link` are new.
- **Topic names have no namespace prefix** (for example `/imu/data`, not `/utux/sensor/imu/data`). The `dev` nodes and the dashboard already use un-prefixed names. Remove the `namespace=` lines from the branch launch files; see [Integration fixes](#integration-fixes-for-feature-branches).
- Every message gets a `std_msgs/Header`, stamped with `get_clock().now()` and the correct `frame_id`.
- New custom messages go in one new package, `uuv_msgs` (listed at the end).

---

## System overview

```
                 ┌────────────────────── SENSORS ──────────────────────┐
Navigator IMU  ─► imu_driver_node            [branch] ─► /imu/data
Navigator mag  ─► compass_driver_node        [branch] ─► /compass/data
Bar30 (I2C)    ─► external_barometer_pub.    [branch] ─► /baro/external/data
Kogger DVL     ─► uuv_dvl_driver             [branch] ─► /dvl/twist, /dvl/altitude, /dvl/raw
Ping360 (UDP)  ─► ping360_node        [third-party]  ─► /sonar/echo, /sonar/scan
JK-BMS (UART)  ─► bms_node                   [branch] ─► /bms/data, /bms/temperature
                                                │
                 ┌──────────── STATE ESTIMATION ┴──────────────────────┐
/imu/data + /compass/data + /baro/external/data + /dvl/twist
        ─► state_estimator [branch, extended] ─► /state_estimate  (+ TF odom→base_link)
                                                │
                 ┌──────────── MAPPING / PLANNING ┴────────────────────┐
/sonar/echo + /state_estimate ─► sonar_mapper [new] ─► /map
/map + /goal_pose ─► global_planner (A*) [new] ─► /plan
/plan + /sonar/scan ─► local_planner (potential field) [new] ─► /control/reference
                                                │
                 ┌──────────── CONTROL ─────────┴──────────────────────┐
/control/reference + /state_estimate ─► adaptive_controller [new] ─► /control/wrench
                                                │
joystick_hal [dev] ─/input/command─┐            │
                                   ▼            ▼
                  motion_converter_node [dev] (mode 0 = manual, 1 = auto)
                                   │ /thruster/command_raw
                                   ▼
                  safety_monitor [new] ── zeros on fault ──► /thruster/command
                                   ▼
                  thruster_driver_node [dev] ─► /pwm/thrusters ─► pwm_driver [dev]

uuv_dashboard [branch] reads the battery, barometer and thruster topics for the operator.
```

**In simulation** (`use_sim:=true`), Stonefish and `sim_bridge` replace every hardware node: the sensor drivers, `ping360_node`, `bms_node`, `thruster_driver_node` and `pwm_driver`. Everything from `state_estimator` downward runs unchanged.

### Changes to existing nodes

Several existing nodes have to change to support this design: `joystick_hal`, `motion_converter_node`, `thruster_driver_node`, `state_estimator` and the branch sensor drivers. Each is specified in [Changes to existing nodes](#changes-to-existing-nodes-modified-not-new) at the end of this document.

---

# 0. In-development nodes (feature branches)

This section documents what is already written, so that new work plugs into it.

## IMU Driver (`imu_driver_node`, package `uuv_imu_driver`, branch `feature/sensor-imu`)

**Subscription(s):** none. It reads the Navigator IMU through `bluerobotics_navigator`.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /imu/data_raw | *sensor_msgs/msg/Imu* | {m/s², rad/s} | frame `imu_link` | Bias-removed acceleration and gyro readings, unfiltered |
| /imu/data | *sensor_msgs/msg/Imu* | {m/s², rad/s} | frame `imu_link` | The same readings after a first-order low-pass filter |

**Parameters:** `timer_period` (0.1 s), `cutoff_frequency` (25 Hz), `calibration_time` (s), `coordinate_system_linear` and `coordinate_system_angular` (±1 sign flip per axis)

### Functionality:

Averages readings for `calibration_time` while the vehicle sits still, and stores the result as a bias (gravity is kept on z). After that it removes the bias, low-pass filters, applies the axis sign flips, and publishes.

It does not fill `orientation`; the state estimator computes it.

## Compass Driver (`compass_driver_node`, package `uuv_compass_driver`, branch `feature/sensor-compass`)

**Subscription(s):** none. It reads the Navigator magnetometer.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /compass/data_raw | *sensor_msgs/msg/MagneticField* | {as returned by the library; check whether µT or T} | frame `compass_link` | Raw magnetic field |
| /compass/data | *sensor_msgs/msg/MagneticField* | same | frame `compass_link` | Low-pass filtered magnetic field |

**Parameters:** `timer_period` (0.1 s), `cutoff_frequency` (25 Hz), `coordinate_frame` (±1 per axis)

### Functionality:

Reads the magnetometer, low-pass filters it, applies the axis sign flips, and publishes. There is no hard-iron or soft-iron calibration yet. The state estimator needs that calibration for a usable heading near the vehicle's motors and battery.

## External Barometer (`external_barometer_publisher`, package `uuv_baro_ext`, branch `feature/sensor-ext-barometer`)

**Subscription(s):** none. It reads the Bar30 (MS5837-30BA) over I2C bus 6.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /baro/external/data_raw | *sensor_msgs/msg/FluidPressure* | {Pa} | – | Raw water pressure |
| /baro/external/data | *sensor_msgs/msg/FluidPressure* | {Pa} | – | Low-pass filtered water pressure. **The state estimator uses this for depth.** |
| /baro/external/temperature_raw | *sensor_msgs/msg/Temperature* | {°C} | – | Raw water temperature |
| /baro/external/temperature | *sensor_msgs/msg/Temperature* | {°C} | – | Filtered water temperature |

**Parameters:** `publish_rate` (Hz), `frame_id`, `cutoff_frequency`, `sampling_frequency`. The fluid density is hard-coded to fresh water.

### Functionality:

Reads pressure and temperature, applies an exponential low-pass filter, and publishes. Depth is not computed here; the state estimator converts pressure to depth.

## DVL Driver (`uuv_dvl_driver`, package `uuv_dvl_driver`, branch `feature/dvl-driver`)

**Status:** only the payload parser is written. `parse_dvl_vel()` decodes the 68-byte `ID_DVL_VEL` payload into a map of field names to values. The node file `uuv_dvl_driver.cpp` is empty. The design for the rest is in [Section 2](#2-dvl-reading).

## State Estimator (`state_estimator`, package `uuv_state_estimator`, branch `feature/control-state_estimator`)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /imu/data | *sensor_msgs/msg/Imu* | {m/s², rad/s} | – | Acceleration and turn rate |
| /compass/data | *sensor_msgs/msg/MagneticField* | {µT} | – | Heading reference |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s} | frame `odom`, child `base_link` | Vehicle pose and velocity |

**Parameters:** `frame_id` (odom), `publish_rate` (50 Hz), `lpf_cutoff` (5 Hz), `g_ref` (9.81), `magnetic_ref_hor` and `magnetic_ref_ver` (Toronto defaults, µT)

### Functionality (current):

Works in the NED frame and runs three steps:
1. **IMU corrector:** removes the lever-arm acceleration caused by the IMU being offset from `base_link`.
2. **Mahony filter:** estimates orientation from the accelerometer, gyro and magnetometer.
3. **6-state linear Kalman filter:** `[p, v]`, predicted from acceleration rotated into the world frame.

`KalmanFilter` already has `updateBarometer()` and `updateGPS()` methods, but the node calls neither. **It does not compile yet** (see Integration fixes). The extension that adds the DVL is in [Section 3](#3-using-the-dvl-state-estimation).

## Battery Management (`bms_node`, package `bms-comm`, branch `feature/bms-comm`)

**Subscription(s):** none. It polls the JK-BMS over UART (`/dev/ttyS0`, 115200 baud).

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /bms/data | *sensor_msgs/msg/BatteryState* | {V, A, %} | 8 cells (8S) | Pack voltage, current, state of charge, cell voltages and health |
| /bms/temperature | *std_msgs/msg/Float32MultiArray* <br>`{float[3]}` | {°C} | – | [0] MOSFET, [1] left probe, [2] right probe |

**Parameters:** `serial_port`, `baud_rate`, `publish_frequency` (2 Hz)

### Functionality:

Parses the BMS serial protocol and publishes battery state for the dashboard. In this design it also feeds `safety_monitor` (low voltage and over-temperature checks).

The same branch also changes the thruster pulse limits to 1100-1900 µs and makes `pwm_driver` start at a neutral duty of 0.075.

## Other branch nodes (used by the operator, not by autonomy)

| Node | Branch | Topics |
| --- | --- | --- |
| `adc_publisher` (`uuv_adc_driver`) | `feature/sensor-adc` | Publishes `/adc/data` (`Float32MultiArray`): every Navigator ADC channel |
| `uuv_camera_driver` (`camera_ros`) | `feature/sensor-camera` | Publishes `/camera/*` images at 640×480 RGB888 |
| `uuv_dashboard` (PyQt6 GUI) | `feature/dashboard` | Subscribes to `/baro/external/data` and `/baro/external/temperature`, `/baro/internal/data` and `/baro/internal/temperature`, `/bms/data` and `/bms/temperature`, and `/thruster/command` |

---

# 1. Simulation

**Goal:** test the control algorithm before pool time. Keep the setup minimal: import the CAD model and run the existing ROS 2 stack inside [Stonefish](https://github.com/patrykcieslak/stonefish_ros2).

## Stonefish Simulator (`stonefish_simulator`, package `stonefish_ros2`, third-party)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sim/thrusters | *std_msgs/msg/Float64MultiArray* <br>`{double[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | Thruster setpoints, in the order the thrusters appear in the scenario file |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sim/imu | *sensor_msgs/msg/Imu* | {rad/s, m/s²} | – | Simulated IMU |
| /sim/dvl | *stonefish_ros2/msg/DVL* | {m/s, m} | – | Simulated DVL velocity and beam ranges |
| /sim/dvl/altitude | *sensor_msgs/msg/Range* | {m} | – | Altitude above the seabed |
| /sim/pressure | *sensor_msgs/msg/FluidPressure* | {Pa} | – | Simulated Bar30 |
| /sim/odometry | *nav_msgs/msg/Odometry* | {m, rad, m/s} | – | **Ground truth**, for scoring the state estimator and the controller only |
| /sim/sonar | *sensor_msgs/msg/Image* | {intensity} | – | Mechanical scanning imaging sonar (Ping360-like) |

**Parameters:** scenario file `uuv_sim/scenarios/uuv.scn`, simulation rate, graphics on or off

### Functionality:

Runs the physics: buoyancy, hydrodynamic drag, thrusters and sensors. The robot is defined in an XML scenario file:
- **Meshes:** export two `.obj` files from CAD, a detailed visual mesh and a simplified physics mesh.
- **Mass properties:** mass, centre of gravity and inertia, taken from CAD.
- **Thrusters:** positions and directions copied from `uuv_motion_converter/config/params.yaml`, so that the simulated robot matches the allocation matrix.
- **Sensors:** each one gets a `<ros_publisher topic=...>` tag.

Graphics need OpenGL 4.3 or newer. The Stonefish library version must match the `stonefish_ros2` version.

## Sim Bridge (`sim_bridge`, package `uuv_sim`, new)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | The same output the real `thruster_driver_node` would receive |
| /sim/dvl | *stonefish_ros2/msg/DVL* | {m/s} | – | Simulated DVL |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sim/thrusters | *std_msgs/msg/Float64MultiArray* | {%} | -1.0 ≤ cmd ≤ 1.0 | Reordered from thruster `id` to scenario order |
| /dvl/twist | *geometry_msgs/msg/TwistWithCovarianceStamped* | {m/s} | – | Same topic as the real DVL driver |

### Functionality:

Adapts the simulator to the real topic names and types, so the stack cannot tell it is in simulation. It converts Float32 to Float64, reorders the thrusters, and converts the DVL message to a twist with covariance.

Topics whose type already matches need only a launch-file remap: `/sim/imu → /imu/data`, `/sim/pressure → /baro/external/data` and `/sim/dvl/altitude → /dvl/altitude`.

Check whether Stonefish can simulate a magnetometer for `/compass/data`. If it can't, the state estimator's yaw will drift in simulation.

To score the controller, plot `/control/reference` against `/sim/odometry`. To score the state estimator, plot `/state_estimate` against `/sim/odometry`.

---

# 2. DVL Reading

**Goal:** read data from the Kogger Micro DVL and produce a report on what it actually outputs. **Start from branch `feature/dvl-driver`**, which already has the `ID_DVL_VEL` payload parser and the protocol PDF.

**What the protocol PDF says** ([Kogger-Protocol](https://github.com/koggertech/Kogger-Protocol), `Kogger SB protocol.pdf`):
- **Frame:** `0xBB 0x55 | ROUTE | MODE | ID | LENGTH(0-128) | PAYLOAD | CHECK1 CHECK2`.
  - Checksum is Fletcher-16.
  - Values are little-endian; floats are IEEE754.
  - MODE bits 0:1 give the TYPE: 1 = CONTENT (device to host), 2 = SETTING, 3 = GETTING. Bits 3:5 give the message version.
- **`ID_DVL_VEL` (0x79), version 2, 68 bytes:**
  - `FLAGS` (U4): the bit meanings are not documented.
  - `TIMESTAMP` (U4, ms), `DELTA_TIME` (F4, s), `LATENCY` (F4, s).
  - `VELOCITY_X/Y/Z/Z1/Z2` (F4, m/s), each with a matching `UNCERTAINTY_*` (F4, m/s).
  - `DISTANCE_Z/Z1/Z2` (F4, m).
- **Other IDs the device may also emit:**
  - `ID_DIST` (0x02, mm)
  - `ID_ATTITUDE` (0x04): Euler angles in 0.01°, or a quaternion
  - `ID_TEMP` (0x05, 0.01 °C)
  - `ID_TIMESTAMP` (0x01, ms)
  - `ID_DATASET` (0x10): configures periodic output, but its bitmask does **not** list DVL_VEL.
- **Not documented:** beam angle, beamwidth, the meaning of `FLAGS`, the meaning of Z1 and Z2, and how to enable `DVL_VEL` output.

## DVL Driver (`uuv_dvl_driver`, package `uuv_dvl_driver`, branch `feature/dvl-driver`, to be completed)

**Subscription(s):** none. It reads the serial port directly.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /dvl/raw | *uuv_msgs/msg/KoggerDvlVel* <br>`{uint32 flags, timestamp; float delta_time, latency, vel[5], unc[5], dist[3]}` | {$\varnothing$, ms, s, m/s, m} | – | Every field of `ID_DVL_VEL` v2, unmodified |
| /dvl/twist | *geometry_msgs/msg/TwistWithCovarianceStamped* | {m/s} | frame `dvl_link` | VELOCITY_X/Y/Z, with covariance on the diagonal equal to UNCERTAINTY². The state estimator reads this. |
| /dvl/altitude | *sensor_msgs/msg/Range* | {m} | min/max range from the datasheet | DISTANCE_Z, the height above the seabed |
| /dvl/status | *diagnostic_msgs/msg/DiagnosticArray* | – | – | Frame rate, checksum errors, raw FLAGS value |

**Parameters:** `port` (/dev/ttyUSB0), `baudrate`, `frame_id` (dvl_link), `max_valid_uncertainty` (m/s)

### Functionality:

Reads serial bytes and finds `0xBB 0x55`. It checks LENGTH and the Fletcher-16 checksum, then dispatches frames by ID. For 0x79 it reuses the existing `parse_dvl_vel()`. Replacing that function's `std::map` output with a packed struct would be faster and avoid string lookups.

`ID_DVL_VEL` v2 is published on all three DVL topics. Unknown IDs are counted, so the profiler can report them. Samples are dropped from `/dvl/twist` when the velocity is NaN or the uncertainty exceeds `max_valid_uncertainty`; `/dvl/raw` still publishes them.

The timestamp is the receive time minus `LATENCY`.

## DVL Profiler (`dvl_profiler`, package `uuv_dvl_driver`, new, a bench and pool test tool)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /dvl/raw | *uuv_msgs/msg/KoggerDvlVel* | as above | – | Raw DVL data |
| /dvl/status | *diagnostic_msgs/msg/DiagnosticArray* | – | – | Driver statistics |

**Publish(s):** none. It writes `dvl_report.md` when it shuts down.

### Functionality:

Produces the **data profile** this task asks for:
- Which message IDs and versions appeared, and at what rate (Hz).
- Minimum, maximum, mean and standard deviation of each field, plus the NaN and zero counts.
- Each `FLAGS` bit pattern observed, alongside the conditions logged at the time (bottom lock or no lock, in air, out of range).
- Typical uncertainty against altitude.

**Test plan:** record a `ros2 bag` for each case below and run the profiler on it.
1. In air.
2. In a tank, sitting still.
3. Pushed at a known speed.
4. At several heights above the bottom, down to the minimum range.

From these runs, infer what Z1, Z2 and the FLAGS bits mean. Ask Kogger directly for the beam angle.

---

# 3. Using the DVL: State Estimation

**Goal:** fuse DVL odometry into the state estimator and use altitude to avoid hitting the seabed. Seabed avoidance is in [Rover Safety](#6-rover-safety).

**Approach:** extend the existing `uuv_state_estimator` on branch `feature/control-state_estimator`. Do not add a third-party EKF.

The changes are specified in [C4. State Estimator](#c4-state-estimator-changed) under *Changes to existing nodes* at the end of this document.

---

# 4. Sonar Mapping

**Goal:** build a map of the lake or pool from the Ping360. **Before choosing a SLAM method, ask Prof. Busheer what he uses for lake mapping.** Phase 1 below is odometry-based mapping, which works whatever SLAM method comes later. See the [underwater SLAM survey](https://www.mdpi.com/2072-4292/15/10/2496) for options.

## Ping360 Driver (`ping360_node`, package `ping360_sonar`, third-party, branch `ros2` of [CentraleNantesRobotics/ping360_sonar](https://github.com/CentraleNantesRobotics/ping360_sonar))

**Subscription(s):** none. It connects over UDP, by default 192.168.2.2:9092, or over serial at 115200 baud.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sonar/echo | *ping360_sonar_msgs/msg/SonarEcho* <br>`{angle, gain, number_of_samples, transmit_frequency, speed_of_sound, range, intensities[]}` | {deg, m, 0-255} | – | One beam: intensity against range at one angle |
| /sonar/scan | *sensor_msgs/msg/LaserScan* | {rad, m} | – | First strong return per angle, for obstacle avoidance |
| /sonar/image | *sensor_msgs/msg/Image* | – | – | Polar sweep image, for the operator |

**Parameters:** `angle_sector` (60-360°), `angle_step` (1-20°), `range_max` (1-50 m), `frequency` (650-850 kHz)

### Functionality:

This node already exists upstream. The driver was written for Foxy, so check that it builds on our ROS 2 distro. Remap its topics into `/sonar/*` and set `frame_id = sonar_link`.

## Sonar Mapper (`sonar_mapper`, package `uuv_sonar`, new)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sonar/echo | *ping360_sonar_msgs/msg/SonarEcho* | {deg, m} | – | Raw beams |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad} | – | Vehicle pose when each beam fired (taken from TF) |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /map | *nav_msgs/msg/OccupancyGrid* | {m/cell} | -1 = unknown, 0 = free, 100 = occupied | 2D map at the operating depth, frame `map` |

**Parameters:** `resolution` (0.2 m), `map_size` (m), `intensity_threshold`, `min_range` (to ignore near-field ringing)

### Functionality:

For each beam:
1. Threshold the intensity samples.
2. Mark the first strong return as occupied and the cells before it as free.
3. Transform the cells into `map` using the pose at the time the beam fired.
4. Update the cells with a log-odds update.

The Ping360 is a slowly rotating single beam, so the vehicle moves during a sweep. Always use the pose at the exact beam timestamp, never the pose at the end of the sweep.

**Map format decision:**
- Use `nav_msgs/OccupancyGrid`, the standard Nav2 type. Save it with `ros2 run nav2_map_server map_saver_cli`, which writes a `.pgm` and a `.yaml`.
- The map is 2D because the vehicle holds depth with the barometer. Move to a 3D OctoMap only if multi-depth mapping is needed.
- Phase 2 (SLAM) publishes `map → odom` to correct the DVL drift.

---

# 5. Path Planning

## Global Planner: A* (`global_planner`, package `uuv_planning`, new)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /map | *nav_msgs/msg/OccupancyGrid* | {m/cell} | – | Static or sonar-built map |
| /state_estimate | *nav_msgs/msg/Odometry* | {m} | – | Start position |
| /goal_pose | *geometry_msgs/msg/PoseStamped* | {m, rad} | frame `map`; z = target depth | Goal, for example clicked in RViz or sent by a mission script |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /plan | *nav_msgs/msg/Path* | {m} | frame `map` | Collision-free list of waypoints |

**Parameters:** `robot_radius` (m, for inflating obstacles), `occupied_threshold` (50), `allow_unknown` (false), `replan_period` (s)

### Functionality:

1. Inflate obstacles by `robot_radius`.
2. Run A* on the 8-connected grid. The cost is the step distance and the heuristic is the Euclidean distance to the goal.
3. Remove waypoints that have line of sight to each other, to shorten the path.
4. Publish the result.

It replans when a new goal arrives, when the map changes along the path, or every `replan_period`. If there is no path, it logs an error and publishes an empty `Path`.

## Local Planner: Potential Field (`local_planner`, package `uuv_planning`, new)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /plan | *nav_msgs/msg/Path* | {m} | – | Path from A* |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, m/s} | – | Current state |
| /sonar/scan | *sensor_msgs/msg/LaserScan* | {m} | – | Live obstacles that are not yet in the map |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /control/reference | *uuv_msgs/msg/TrajectoryReference* <br>`{Pose pose; Twist twist; Accel accel}` | {m, rad, m/s, m/s²} | speed ≤ `max_speed` | Desired pose, velocity and acceleration for the controller |

**Parameters:** `lookahead` (m), `k_att`, `k_rep`, `influence_radius` (m), `max_speed` (m/s), `max_accel` (m/s²), `rate_hz` (10)

### Functionality:

The force on the vehicle is the sum of two parts:
- **Attraction** toward a point `lookahead` metres along `/plan`.
- **Repulsion** from each obstacle point within `influence_radius`.

The node converts that force into a desired velocity, limits the speed and acceleration, and integrates the velocity into a smooth pose reference. Heading points along the velocity, and depth comes from the goal z.

Potential fields can get stuck in local minima. If the vehicle stops short of the goal for more than N seconds, ask the global planner to replan.

---

# 6. Rover Safety

**Goal:** detect sensor or state malfunctions and cut motor commands.

## Safety Monitor (`safety_monitor`, package `uuv_safety`, new)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command_raw | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | Output of `motion_converter_node`, to be gated |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | – | – | Operator heartbeat (manual mode) |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | – | Controller heartbeat (autonomous mode) |
| /imu/data, /compass/data, /baro/external/data, /dvl/twist, /state_estimate | as above | – | – | Sensor and state health |
| /dvl/altitude | *sensor_msgs/msg/Range* | {m} | – | Seabed clearance |
| /bms/data, /bms/temperature | *sensor_msgs/msg/BatteryState*, *Float32MultiArray* | {V, °C} | – | Battery health |
| /pwm/generated | *pwm_msg/msg/ThrustersPWM* | {%, $\varnothing$} | – | Hardware PWM validity flags |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | `command_raw` passed through when OK, **all zeros** on FAULT |
| /safety/status | *uuv_msgs/msg/SafetyStatus* <br>`{Header; uint8 state; string[] reasons}` | – | OK = 0, WARN = 1, FAULT = 2 | Current safety state and why; can be shown on the dashboard |

**Service(s):** `/safety/reset` (*std_srvs/srv/Trigger*) clears a latched FAULT, if every check passes.

**Parameters:** `rate_hz` (20), a per-topic `timeout_s`, `min_altitude` (m), `max_depth` (m), `max_tilt` (rad), `max_dvl_uncertainty` (m/s), `min_cell_voltage` (V), `max_battery_temp` (°C), `max_position_std` (m)

### Functionality:

Runs on a **timer**, not on message callbacks. Because of this, a silent publisher still trips a timeout.

| Check | Condition | Result |
| --- | --- | --- |
| Command heartbeat | No `/input/command` (manual) or `/control/wrench` (auto) for more than the timeout | FAULT |
| Sensor alive | Any sensor topic silent for more than its timeout | FAULT |
| Invalid values | NaN or Inf in any command or state | FAULT |
| Seabed | `/dvl/altitude` < `min_altitude` | FAULT |
| Depth | depth > `max_depth` | FAULT |
| Attitude | \|roll\| or \|pitch\| > `max_tilt` | FAULT |
| Estimator health | Position standard deviation in `/state_estimate` > `max_position_std` (in auto mode) | FAULT |
| DVL quality | Uncertainty > `max_dvl_uncertainty` | WARN |
| Battery | Any cell < `min_cell_voltage`, any temperature > `max_battery_temp`, or `power_supply_health` not GOOD | FAULT |
| PWM hardware | Any `pwm_valid` is false on channels 0-7 | FAULT |

A FAULT **latches**: the node outputs zero thrust (neutral pulse) until `/safety/reset` is called.

The node sits after `motion_converter_node`, so it covers both manual and autonomous modes. The `thruster_driver_node` timeout from the changes table is the backstop if `safety_monitor` itself dies.

---

# 7. Adaptive Controller

**Goal:** a Slotine-Li adaptive controller whose model parameters are learned online with gradient updates.

## Adaptive Controller (`adaptive_controller`, package `uuv_control`, new)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s, rad/s} | – | Actual state η, ν |
| /control/reference | *uuv_msgs/msg/TrajectoryReference* | {m, rad, m/s, m/s²} | – | Desired η_d, η̇_d, η̈_d |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | frame `base_link`; clamped to the max wrench | Body-frame force and torque, sent to `motion_converter_node` |
| /control/params | *std_msgs/msg/Float64MultiArray* | mixed | within `theta_min` to `theta_max` | Current estimate θ̂, for logging and tuning |

**Parameters:** `Lambda` (6), `K_D` (6), `Gamma` (one entry per parameter), `theta_init`, `theta_min`, `theta_max`, `deadzone` (on ‖s‖), `rate_hz` (50), `adapt_enabled`

### Functionality:

**Model.** The vehicle follows Fossen's model: M ν̇ + C(ν)ν + D(ν)ν + g(η) = τ.

The unknown parameters θ appear linearly, so the left side can be written as Y(η, ν, ν_r, ν̇_r) θ. The parameters are:
- Rigid-body and added mass
- Linear and quadratic damping
- Net buoyancy and the offset between the centre of gravity and the centre of buoyancy

**Each step:**
1. Compute the tracking error η̃ = η − η_d and the composite error s = η̃̇ + Λ η̃.
2. Compute the reference velocity η̇_r = η̇_d − Λ η̃. Convert it to the body frame with ν_r = J(η)⁻¹ η̇_r.
3. Control law: τ = Y θ̂ − K_D s, with s converted to the body frame.
4. Gradient update: θ̂̇ = −Γ Yᵀ s.
5. Integrate the update and clamp θ̂ to its bounds (projection).

Skip the update when ‖s‖ < `deadzone`, so that sensor noise does not make the parameters drift.

`state_estimator` works in NED; convert its output once at the controller input and keep a single convention internally.

**Testing.** Tune in simulation first. With adaptation off and θ̂ fixed, the controller is a plain computed-torque controller, which makes a good baseline. Then turn adaptation on and check that θ̂ converges and that tracking error falls. Repeat in the pool.

---

# New messages (`uuv_msgs`)

| Message | Fields |
| --- | --- |
| `KoggerDvlVel` | `Header header`; `uint32 flags`; `uint32 device_timestamp_ms`; `float32 delta_time, latency`; `float32 vel_x, vel_y, vel_z, vel_z1, vel_z2`; `float32 unc_x, unc_y, unc_z, unc_z1, unc_z2`; `float32 dist_z, dist_z1, dist_z2` |
| `TrajectoryReference` | `Header header`; `geometry_msgs/Pose pose`; `geometry_msgs/Twist twist`; `geometry_msgs/Accel accel` |
| `SafetyStatus` | `Header header`; `uint8 state` (OK = 0, WARN = 1, FAULT = 2); `string[] reasons` |

# Integration fixes for feature branches

These need fixing before the branches merge into `dev` and the new nodes can rely on them:

| Branch | Issue | Fix |
| --- | --- | --- |
| all sensor branches | They branched from an old `main`, so a plain diff against `dev` deletes `uuv_joystick_hal`, `uuv_motion_converter` and the other `dev` packages | Rebase or merge onto `dev` before opening a PR, so that only the new package is added |
| sensor-compass, sensor-ext-barometer, sensor-adc, control-state_estimator | Launch files use the namespaces `utux` or `utux/sensor`, so the topics become `/utux/sensor/baro/external/data` and so on. The dashboard subscribes to `/baro/external/data`. | Remove `namespace=` everywhere (the convention above) |
| sensor-imu, sensor-compass | The header stamp is time since the node started, not ROS time | Use `self.get_clock().now().to_msg()`, as the barometer node already does |
| sensor-imu, sensor-compass | They publish zeros when a read fails, which looks like valid data | Skip publishing on a failed read, so that the safety timeout trips |
| sensor-ext-barometer | `frame_id` is `baro_external_link`, but the URDF link is `baro_ext_link`. `uuv_baro_ext_config.yaml` uses `ros_parameters` (missing `__`). | Match the URDF name, and fix the YAML key |
| sensor-compass | `compass_driver_launch.py` is missing a comma after `namespace` (syntax error) | Add the comma |
| control-state_estimator | Does not compile. It has name mismatches (`T_IMUToBase` vs `T_IMUToBase_`, `kalman_filter` vs `kalman_filter_`/`kalman_filer`, `q_bodyToWorld`, `mag_ref`), missing semicolons in the parameter declarations, and assigns Eigen vectors directly to ROS message fields | Fix the build first, then add the barometer and DVL updates |
| dvl-driver | The node file is empty | Implement it as in Section 2 |
| dashboard | Subscribes to `/baro/internal/*`, but no node publishes it | Add an internal barometer driver, or remove those widgets |
| imu, compass, adc, pwm_driver | Four separate processes each call `navigator.init()` | Confirm that the library allows this. If it doesn't, combine them into one Navigator process. |

# Suggested order of work

| # | Task | Depends on | Done when |
| --- | --- | --- | --- |
| 0 | Merge the sensor branches onto `dev` | Integration fixes | IMU, compass, barometer and BMS topics appear with the agreed names |
| 1 | DVL Reading | `feature/dvl-driver` | `dvl_report.md` is written from the bench and tank bags |
| 2 | Simulation | URDF, CAD | The vehicle responds to the joystick in Stonefish |
| 3 | Rover Safety | 0 | Unplugging any sensor, the joystick or a low battery stops the thrusters, both in simulation and on the bench |
| 4 | State estimation, extended | 0, 1 | `/state_estimate` follows `/sim/odometry`, and depth matches the barometer |
| 5 | Adaptive Controller | 2, 4 | It tracks a step and a circle in simulation, then in the pool |
| 6 | Sonar Mapping | 4, Prof. Busheer | A pool map is saved as `.pgm` and `.yaml` |
| 7 | Path Planning | 5, 6 | The vehicle reaches `/goal_pose` around an obstacle |

# Open questions

- **Kogger:** beam angle, the meaning of `FLAGS`, Z1 and Z2, how to enable `DVL_VEL` output, and the rate and baud rate. Ask the vendor, and confirm with the profiler.
- **Navigator library:** can several processes call `navigator.init()` at the same time?
- **Compass units:** what does `navigator.read_mag()` return (µT or T)? The state estimator's references are in µT.
- **ROS 2 distro:** the repo doesn't pin one. The Ping360 driver targets Foxy, so check compatibility with Humble or Jazzy.
- **SLAM:** the method is pending the consultation with Prof. Busheer.

---

# Changes to existing nodes (modified, not new)

Every node in this section **already exists**, either on `dev` or on a feature branch. Each one is marked **CHANGED**. Only the differences from today's behaviour are described: rows tagged **New**, **Renamed** or **Changed** are the edits, and untagged rows stay as they are. For a node's current behaviour, see `uuv-code_dev_nodes.md` (for `dev` nodes) or [Section 0](#0-in-development-nodes-feature-branches) (for branch nodes).

| # | Node | Lives on | Why it changes |
| --- | --- | --- | --- |
| C1 | `joystick_hal` | `dev` | The operator needs to switch between manual and autonomous mode |
| C2 | `motion_converter_node` | `dev` | Accept the controller's wrench, and send output through `safety_monitor` |
| C3 | `thruster_driver_node` | `dev` | A backstop timeout in case `safety_monitor` dies |
| C4 | `state_estimator` | `feature/control-state_estimator` | Fuse the barometer and DVL; broadcast TF |
| C5 | `imu_driver_node`, `compass_driver_node`, `external_barometer_publisher` | `feature/sensor-*` | Topic names, timestamps and failure behaviour that downstream nodes rely on |
| – | `uuv.urdf.xacro` (not a node) | `dev` | Add `dvl_link` and `sonar_link` with their measured mounting offsets |

## C1. Joystick HAL (`joystick_hal`, package `uuv_joystick_hal`): CHANGED

**Subscription(s):** unchanged (`joy`).

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | {$\varnothing$ (fraction of max)} | **Changed:** `mode` ∈ {0, 1} | `mode` was always 0. It now toggles between 0 (manual) and 1 (autonomous). |

**New parameters:** `mode_toggle_button` (button index). Do not use 4 or 5; `motion_converter_node` already uses those to flip heave and yaw.

### Functionality (change):

On each PRESSED event of `mode_toggle_button`, flip an internal `mode` flag and put it in every published `UUVCommand`. The node keeps publishing in autonomous mode, so `/input/command` stays the operator heartbeat that `safety_monitor` watches. Moving the sticks in auto mode has no effect; the operator stops the vehicle by toggling back or by releasing the controller (heartbeat timeout).

## C2. Motion Converter (`motion_converter_node`, package `uuv_motion_converter`): CHANGED

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | {$\varnothing$ (fraction of max)} | axes ∈ [-1.0, 1.0] | Unchanged. It is now also read for `mode`. |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | frame `base_link` | **New:** wrench from `adaptive_controller`, used only when `mode == 1` |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command_raw | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | **Renamed** from `/thruster/command`, so that `safety_monitor` can gate it |

### Functionality (change):

- **mode 0 (manual):** unchanged. The joystick fractions are scaled by the max wrench, then go through the pseudo-inverse, normalisation and mapping.
- **mode 1 (auto):** use the latest `/control/wrench` directly as the wrench. Skip the scaling by max force and moment, because the controller already outputs N and N·m. Then apply the same pseudo-inverse, saturation normalisation and per-thruster mapping as in manual mode.
- **Auto with no wrench:** if no `/control/wrench` has arrived in auto mode, publish zeros.
- **Startup:** start in mode 0 until the first `/input/command` arrives.
- **Dashboard:** the dashboard currently subscribes to `/thruster/command`. That is still correct, because it now shows the post-safety command.

## C3. Thruster Driver (`thruster_driver_node`, package `uuv_motion_converter`): CHANGED

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | Unchanged topic. It is now published by `safety_monitor`, not by `motion_converter_node`. |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/thrusters | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | 0.055 ≤ duty ≤ 0.095 (1100-1900 µs at 50 Hz) | **Changed:** the timeout also publishes neutral (0.075) |

**New parameters:** `command_timeout_s` (0.5)

### Functionality (change):

Add a timer, at 20 Hz for example. If no valid `/thruster/command` has arrived within `command_timeout_s`, publish neutral (0.075) on all 8 channels. This is the last line of defence when `safety_monitor`, or anything upstream of it, crashes.

The 1100-1900 µs pulse limits come from branch `feature/bms-comm`. Keep them when that branch merges.

## C4. State Estimator (`state_estimator`, package `uuv_state_estimator`): CHANGED

Fix the build first; see [Integration fixes](#integration-fixes-for-feature-branches). Then make the changes below.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /imu/data | *sensor_msgs/msg/Imu* | {m/s², rad/s} | – | Unchanged: prediction input |
| /compass/data | *sensor_msgs/msg/MagneticField* | {µT} | – | Unchanged: yaw reference for Mahony |
| /baro/external/data | *sensor_msgs/msg/FluidPressure* | {Pa} | – | **New:** depth update |
| /dvl/twist | *geometry_msgs/msg/TwistWithCovarianceStamped* | {m/s} | frame `dvl_link` | **New:** velocity update |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s} | frame `odom`, child `base_link` | **Changed:** now fills the pose and twist covariance. Every node downstream uses this topic. |
| /tf | `odom → base_link` | – | – | **New:** broadcast so RViz and the mapper can use TF |

**New parameters:** `water_density` (997), `surface_pressure` (Pa, measured at startup), `dvl_noise_floor` (m/s)

### Functionality (additions):

- **Barometer update:** depth z = (P − P_surface) / (ρg). NED convention, so z is positive downward. Call the existing `KalmanFilter::updateBarometer(z)`.
- **DVL update:** add `updateVelocity(v_world, R)`, with observation matrix H = [0 I] on the velocity states.
  - Convert the DVL velocity to the world frame: v_world = R(q) · R_dvl→base · v_dvl, where q is the Mahony orientation.
  - Ignore the lever-arm term ω × r for now; it is small at low turn rates.
  - The measurement noise R comes from the message covariance, which is UNCERTAINTY² (never below `dvl_noise_floor`²).
- **Without DVL:** if `/dvl/twist` goes silent (lost bottom lock), keep predicting from the IMU only. Let the covariance grow so that `safety_monitor` can see it.
- x and y still drift slowly, because the DVL measures velocity, not position. Sonar SLAM corrects this later by publishing `map → odom`.

## C5. Sensor drivers (`imu_driver_node`, `compass_driver_node`, `external_barometer_publisher`): CHANGED

Topics, types and units stay as described in [Section 0](#0-in-development-nodes-feature-branches). The changes are about integration:

| Node | Change | Why |
| --- | --- | --- |
| all three | Remove `namespace=` from the launch files, so the topics are `/imu/data`, `/compass/data` and `/baro/external/data` | The dashboard, `state_estimator`, `safety_monitor` and `sim_bridge` all expect these names |
| `imu_driver_node`, `compass_driver_node` | Stamp headers with `get_clock().now()`, not time since the node started | The state estimator and TF need ROS time; the barometer already does this |
| `imu_driver_node`, `compass_driver_node` | On a failed read, publish nothing instead of zeros | Zeros look like valid data; silence lets the `safety_monitor` timeout trip |
| `compass_driver_node` | Add hard-iron and soft-iron calibration (offset and scale parameters) | The motors and battery distort the field, and Mahony needs a clean heading |
| `external_barometer_publisher` | Set `frame_id` to `baro_ext_link`, and make the fluid density a parameter | Match the URDF; the vehicle is used in both fresh and salt water |
