# Sim Bridge (`sim_bridge`, package `uuv_sim`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | The same input the real `thruster_driver_node` would receive |
| /sim/dvl | *stonefish_ros2/msg/DVL* | {m/s} | – | Simulated DVL |
| /sim/sonar | *sensor_msgs/msg/Image* | {intensity} | – | Simulated sonar |
| /sim/odometry | *nav_msgs/msg/Odometry* | {m, rad, m/s} | NED | Ground truth |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sim/thrusters | *std_msgs/msg/Float64MultiArray* | {%} | -1.0 ≤ cmd ≤ 1.0 | Reordered from thruster `id` to scenario order |
| /dvl/twist | *geometry_msgs/msg/TwistWithCovarianceStamped* | {m/s} | frame `dvl_link` | Same topic as the real DVL driver |
| /sonar/echo | *ping360_sonar_msgs/msg/SonarEcho* | as [ping360_node](../sonar_mapping/ping360_node.md) | frame `sonar_link` | One beam, same topic as the real sonar driver |
| /sonar/scan | *sensor_msgs/msg/LaserScan* | {rad, m} | frame `sonar_link` | First strong return per angle, same as the real sonar driver |
| /sim/ground_truth | *nav_msgs/msg/Odometry* | {m, rad, m/s} | frame `odom`, child `base_link`; ENU/FLU | Ground truth in the same convention as `/state_estimate`, for scoring |

### Functionality:

Adapts the simulator to the real topic names and types, so the stack cannot tell it is in simulation. It:
- converts Float32 to Float64 and reorders the thrusters;
- converts the DVL message to a twist with covariance;
- converts the sonar image into per-beam `SonarEcho` messages and a `LaserScan`. Check what one Stonefish sonar image holds (one beam or the whole fan); if it is the whole fan, publish only the newest beam each time;
- converts ground truth from NED to ENU/FLU.

Topics whose type already matches need only a launch-file remap: `/sim/imu → /imu/data`, `/sim/pressure → /baro/external/data` and `/sim/dvl/altitude → /dvl/altitude`.

**Not simulated:** `/bms/*`, `/pwm/generated` and `/dvl/raw`. The sim launch file tells `safety_monitor` to skip the checks that need them (see [safety_monitor.md](../rover_safety/safety_monitor.md)).

Check whether Stonefish can simulate a magnetometer for `/compass/data`. If it can't, the state estimator's yaw will drift in simulation, and `/compass/data` must also be removed from `safety_monitor`'s sim topic list.
