# Safety Monitor (`safety_monitor`, package `uuv_safety`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command_raw | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | Output of `motion_converter_node`, to be gated |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | – | `mode` ∈ {0, 1} | Operator heartbeat (both modes), and the current mode |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | – | Controller heartbeat (autonomous mode only) |
| /imu/data, /compass/data, /baro/external/data, /state_estimate | as in their nodes | – | – | Sensor and state health |
| /dvl/raw | *uuv_msgs/msg/KoggerDvlVel* | {m/s} | – | DVL driver alive, and DVL uncertainty. `/dvl/twist` is not used, because it drops samples whenever bottom lock is lost. |
| /dvl/altitude | *sensor_msgs/msg/Range* | {m} | – | Seabed clearance |
| /bms/data, /bms/temperature | *sensor_msgs/msg/BatteryState*, *std_msgs/msg/Float32MultiArray* | {V, °C} | – | Battery health |
| /pwm/generated | *pwm_msg/msg/ThrustersPWM* | {%, $\varnothing$} | – | Hardware PWM validity flags |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | `command_raw` passed through when OK or WARN, **all zeros** on FAULT |
| /safety/status | *uuv_msgs/msg/SafetyStatus* <br>`{Header; uint8 state; string[] reasons}` | – | OK = 0, WARN = 1, FAULT = 2 | Current safety state and why; can be shown on the dashboard |

**Service(s):** `/safety/reset` (*std_srvs/srv/Trigger*) clears a latched FAULT, if every check passes.

**Parameters:** `rate_hz` (20), `sensor_topics` (list of topics for the alive check), a per-topic `timeout_s`, `disabled_checks` (list, empty on the vehicle), `min_altitude` (m), `max_depth` (m), `max_tilt` (rad), `max_dvl_uncertainty` (m/s), `min_cell_voltage` (V), `max_battery_temp` (°C), `max_position_std` (m)

### Functionality:

Runs on a **timer**, not on message callbacks. Because of this, a silent publisher still trips a timeout. The mode comes from the latest `/input/command`.

| Check | Condition | Result |
| --- | --- | --- |
| Operator heartbeat | No `/input/command` for more than its timeout, in either mode | FAULT |
| Controller heartbeat | In auto mode (`mode == 1`), no `/control/wrench` for more than its timeout | FAULT |
| Sensor alive | Any topic in `sensor_topics` silent for more than its timeout | FAULT |
| Invalid values | NaN or Inf in any command or state | FAULT |
| Seabed | `/dvl/altitude` < `min_altitude` | FAULT |
| Depth | depth (−z of `/state_estimate`) > `max_depth` | FAULT |
| Attitude | \|roll\| or \|pitch\| > `max_tilt` | FAULT |
| Estimator health | Position standard deviation in `/state_estimate` > `max_position_std` (in auto mode). This is what catches a long loss of DVL bottom lock. | FAULT |
| DVL quality | Uncertainty in `/dvl/raw` > `max_dvl_uncertainty` | WARN |
| Battery | Any cell < `min_cell_voltage`, any temperature > `max_battery_temp`, or `power_supply_health` not GOOD | FAULT |
| PWM hardware | Any `pwm_valid` is false on channels 0-7 | FAULT |

A FAULT **latches**: the node outputs zero thrust (neutral pulse) until `/safety/reset` is called.

The node sits after `motion_converter_node`, so it covers both manual and autonomous modes. The [thruster_driver_node timeout](../changes_to_existing/thruster_driver_node.md) is the backstop if `safety_monitor` itself dies.

**In simulation** the launch file sets `disabled_checks` to `[battery, pwm, dvl_quality]` and replaces `/dvl/raw` with `/dvl/twist` in `sensor_topics` (it also drops `/compass/data`, if Stonefish has no magnetometer). Using `/dvl/twist` for the alive check is fine because the simulated DVL does not lose lock unless the scenario makes it.
