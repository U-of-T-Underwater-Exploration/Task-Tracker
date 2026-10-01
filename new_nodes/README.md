# uuv-code: New Feature Architecture

These documents describe the nodes that implement the Task-Tracker items. They use the same format as [`nodes.md`](../nodes.md), which documents the `dev` branch.

Each node is marked with one of three statuses:
- **dev:** already merged into `dev`.
- **branch:** in development on a `feature/*` branch.
- **new:** to be written.

Where a feature branch already covers part of a task, the design builds on that branch rather than adding a new node.

## Layout

One directory per task (see the [suggested order of work](#suggested-order-of-work)). Each node has its own file.

| Directory | Contents |
| --- | --- |
| [feature_branches/](feature_branches/) | Nodes already written on `feature/*` branches, as they are today |
| [dvl_reading/](dvl_reading/) | Task: DVL Reading |
| [simulation/](simulation/) | Task: Simulation |
| [rover_safety/](rover_safety/) | Task: Rover Safety |
| [state_estimation/](state_estimation/) | Task: Using the DVL (state estimation) |
| [adaptive_controller/](adaptive_controller/) | Task: Adaptive Controller |
| [sonar_mapping/](sonar_mapping/) | Task: Sonar Mapping |
| [path_planning/](path_planning/) | Task: Path Planning |
| [changes_to_existing/](changes_to_existing/) | Edits to nodes that already exist on `dev` or a branch |
| [uuv_msgs.md](uuv_msgs.md) | New custom messages |
| [integration_fixes.md](integration_fixes.md) | Fixes needed before the feature branches merge into `dev` |

## Conventions (apply to every node)

- Units are SI: m, m/s, rad, N, N·m, Pa, T. Duty cycles and normalised commands are fractions shown as {%} (0-1 or -1 to 1), as in `nodes.md`.
- Frames follow [REP-105](https://www.ros.org/reps/rep-0105.html): `map → odom → base_link`. Sensor frames must match the link names in `uuv.urdf.xacro`: `imu_link`, `compass_link`, `baro_ext_link` and `cam_link` exist already. `dvl_link` and `sonar_link` are new (see [uuv_urdf.md](changes_to_existing/uuv_urdf.md)).
- **Axes follow [REP-103](https://www.ros.org/reps/rep-0103.html) on every topic and in TF:** ENU for `map` and `odom`, FLU for `base_link`. A node may use NED internally (the state estimator and the controller do), but it converts at its inputs and outputs. Because z points up, **depth = −z**.
- Until sonar SLAM exists, the top-level launch file publishes a static identity `map → odom` transform, so nodes working in `map` and nodes working in `odom` agree.
- **Topic names have no namespace prefix** (for example `/imu/data`, not `/utux/sensor/imu/data`). The `dev` nodes and the dashboard already use un-prefixed names. Remove the `namespace=` lines from the branch launch files; see [Integration fixes](integration_fixes.md).
- Every message gets a `std_msgs/Header`, stamped with `get_clock().now()` and the correct `frame_id`.
- New custom messages go in one new package, `uuv_msgs` ([uuv_msgs.md](uuv_msgs.md)).

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
/sonar/echo + TF ─► sonar_mapper [new] ─► /map
/map + /goal_pose + /state_estimate ─► global_planner (A*) [new] ─► /plan
/plan + /state_estimate ─► path_follower [new] ─► /control/reference
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

**In simulation** (`use_sim:=true`), Stonefish and `sim_bridge` replace every hardware node: the sensor drivers, `ping360_node`, `bms_node`, `thruster_driver_node` and `pwm_driver`. Everything from `state_estimator` downward runs unchanged, except that `safety_monitor` disables the checks whose topics have no simulated source (see [safety_monitor.md](rover_safety/safety_monitor.md)).

Several existing nodes have to change to support this design: `joystick_hal`, `motion_converter_node`, `thruster_driver_node`, `state_estimator` and the branch sensor drivers. Each is specified in [changes_to_existing/](changes_to_existing/).

---

## Suggested order of work

| # | Task | Depends on | Existing-node changes | Done when |
| --- | --- | --- | --- | --- |
| 0 | Merge the sensor branches onto `dev` | [Integration fixes](integration_fixes.md) | IMU, compass and barometer driver changes | IMU, compass, barometer and BMS topics appear with the agreed names |
| 1 | [DVL Reading](dvl_reading/) | `feature/dvl-driver` | – | `dvl_report.md` is written from the bench and tank bags |
| 2 | [Simulation](simulation/) | URDF, CAD | – | The vehicle responds to the joystick in Stonefish |
| 3 | [Rover Safety](rover_safety/) | 0 | `motion_converter_node` rename to `/thruster/command_raw`; `thruster_driver_node` timeout | Unplugging any sensor, the joystick or a low battery stops the thrusters, both in simulation and on the bench |
| 4 | [State estimation, extended](state_estimation/) | 0, 1 | `state_estimator` | `/state_estimate` follows the simulator's ground truth, and depth matches the barometer |
| 5 | [Adaptive Controller](adaptive_controller/) | 2, 3, 4 | `joystick_hal` mode toggle; `motion_converter_node` auto mode | It tracks a step and a circle in simulation, then in the pool |
| 6 | [Sonar Mapping](sonar_mapping/) | 4, Prof. Busheer | – | A pool map is saved as `.pgm` and `.yaml` |
| 7 | [Path Planning](path_planning/) | 5, 6 | – | The vehicle reaches `/goal_pose` around an obstacle |

## Open questions

- **Kogger:** beam angle, the meaning of `FLAGS`, Z1 and Z2, how to enable `DVL_VEL` output, and the rate and baud rate. Ask the vendor, and confirm with the profiler.
- **Navigator library:** can several processes call `navigator.init()` at the same time?
- **Compass units:** what does `navigator.read_mag()` return (µT or T)? `sensor_msgs/MagneticField` is defined in tesla, so the driver should convert if needed.
- **ROS 2 distro:** the repo doesn't pin one. The Ping360 driver targets Foxy, so check compatibility with Humble or Jazzy.
- **Ping360 angle units:** the Ping360 protocol counts angle in gradians (400 per turn). Check whether `ping360_sonar` converts to degrees in `SonarEcho`.
- **Stonefish sonar and magnetometer:** what the simulated sonar image contains, and whether a magnetometer can be simulated.
- **SLAM:** the method is pending the consultation with Prof. Busheer.
