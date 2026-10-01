# State Estimator (`state_estimator`, package `uuv_state_estimator`, branch `feature/control-state_estimator`)

**Status:** branch, **does not compile yet** (see [Integration fixes](../integration_fixes.md)). The extension that adds the barometer and DVL is in [changes_to_existing/state_estimator.md](../changes_to_existing/state_estimator.md).

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /imu/data | *sensor_msgs/msg/Imu* | {m/s², rad/s} | – | Acceleration and turn rate |
| /compass/data | *sensor_msgs/msg/MagneticField* | {µT} | – | Heading reference |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s} | frame `odom`, child `base_link`; NED axes | Vehicle pose and velocity |

**Parameters:** `frame_id` (odom), `publish_rate` (50 Hz), `lpf_cutoff` (5 Hz), `g_ref` (9.81), `magnetic_ref_hor` and `magnetic_ref_ver` (Toronto defaults, µT)

### Functionality (current):

Works in the NED frame and runs three steps:
1. **IMU corrector:** removes the lever-arm acceleration caused by the IMU being offset from `base_link`.
2. **Mahony filter:** estimates orientation from the accelerometer, gyro and magnetometer.
3. **6-state linear Kalman filter:** `[p, v]`, predicted from acceleration rotated into the world frame.

`KalmanFilter` already has `updateBarometer()` and `updateGPS()` methods, but the node calls neither.

It publishes NED values in the `odom` frame, which breaks the REP-103 [convention](../README.md#conventions-apply-to-every-node); the change file fixes this.
