# C5. IMU Driver (`imu_driver_node`, package `uuv_imu_driver`): CHANGED

**Lives on:** `feature/sensor-imu`. Current behaviour: [feature_branches/imu_driver_node.md](../feature_branches/imu_driver_node.md).

Topics, types and units stay the same. The changes are about integration:

| Change | Why |
| --- | --- |
| Make sure the launch file has no `namespace=`, so the topic is `/imu/data` | The dashboard, `state_estimator`, `safety_monitor` and `sim_bridge` all expect this name |
| Stamp headers with `get_clock().now()`, not time since the node started | The state estimator and TF need ROS time; the barometer already does this |
| On a failed read, publish nothing instead of zeros | Zeros look like valid data; silence lets the `safety_monitor` timeout trip |
| Choose `timer_period` and `cutoff_frequency` together, so the sample rate is more than twice the cutoff | At 10 Hz sampling, a 25 Hz low-pass filter does nothing |
