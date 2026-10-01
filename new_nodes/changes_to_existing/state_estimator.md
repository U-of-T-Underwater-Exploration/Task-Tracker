# C4. State Estimator (`state_estimator`, package `uuv_state_estimator`): CHANGED

**Lives on:** `feature/control-state_estimator`. Current behaviour: [feature_branches/state_estimator.md](../feature_branches/state_estimator.md).

Fix the build first; see [Integration fixes](../integration_fixes.md). Then make the changes below.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /imu/data | *sensor_msgs/msg/Imu* | {m/s², rad/s} | – | Unchanged: prediction input |
| /compass/data | *sensor_msgs/msg/MagneticField* | **Changed:** {T} | – | Yaw reference for Mahony. Now in tesla, as the message defines (see [compass change](compass_driver_node.md)). |
| /baro/external/data | *sensor_msgs/msg/FluidPressure* | {Pa} | – | **New:** depth update |
| /dvl/twist | *geometry_msgs/msg/TwistWithCovarianceStamped* | {m/s} | frame `dvl_link` | **New:** velocity update |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s} | frame `odom`, child `base_link`; **Changed:** ENU/FLU | **Changed:** converted from internal NED to ENU/FLU, and now fills the pose and twist covariance. Every node downstream uses this topic. |
| /tf | `odom → base_link` | – | – | **New:** broadcast so RViz, the mapper and the planners can use TF |

**New parameters:** `water_density` (997 kg/m³), `surface_pressure` (Pa; if unset, averaged from the first barometer samples at startup), `dvl_noise_floor` (m/s)

**Changed parameters:** `magnetic_ref_hor` and `magnetic_ref_ver` are now in tesla.

### Functionality (additions):

- **Output convention:** keep NED internally, and convert pose, twist and covariance to ENU (`odom`) / FLU (`base_link`) when publishing `/state_estimate` and TF.
- **Barometer update:** depth d = (P − P_surface) / (ρg), positive downward, which is the internal NED z. Call the existing `KalmanFilter::updateBarometer(d)`.
- **DVL update:** add `updateVelocity(v_world, R)`, with observation matrix H = [0 I] on the velocity states.
  - Convert the DVL velocity to the world frame: v_world = R(q) · R_dvl→base · v_dvl, where q is the Mahony orientation and R_dvl→base comes from TF.
  - Ignore the lever-arm term ω × r for now; it is small at low turn rates.
  - The measurement noise R comes from the message covariance, which is UNCERTAINTY² (never below `dvl_noise_floor`²).
- **Without DVL:** if `/dvl/twist` goes silent (lost bottom lock), keep predicting from the IMU only. Let the covariance grow, so that `safety_monitor`'s estimator-health check can see it.
- x and y still drift slowly, because the DVL measures velocity, not position. Sonar SLAM corrects this later by publishing `map → odom`.
