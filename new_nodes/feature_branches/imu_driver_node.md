# IMU Driver (`imu_driver_node`, package `uuv_imu_driver`, branch `feature/sensor-imu`)

**Status:** branch. Required edits: [changes_to_existing/imu_driver_node.md](../changes_to_existing/imu_driver_node.md).

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

Note: with `timer_period` at 0.1 s the node samples at 10 Hz, below twice the 25 Hz `cutoff_frequency`, so the filter has no effect at that rate. Choose the two together.
