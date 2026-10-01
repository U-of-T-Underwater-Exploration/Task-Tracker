# C6. Compass Driver (`compass_driver_node`, package `uuv_compass_driver`): CHANGED

**Lives on:** `feature/sensor-compass`. Current behaviour: [feature_branches/compass_driver_node.md](../feature_branches/compass_driver_node.md).

Topics and types stay the same.

| Change | Why |
| --- | --- |
| Remove `namespace=` from the launch file (this also fixes the missing-comma syntax error), so the topic is `/compass/data` | The dashboard, `state_estimator`, `safety_monitor` and `sim_bridge` all expect this name |
| Stamp headers with `get_clock().now()`, not time since the node started | The state estimator and TF need ROS time; the barometer already does this |
| On a failed read, publish nothing instead of zeros | Zeros look like valid data; silence lets the `safety_monitor` timeout trip |
| Publish in tesla, converting from what `navigator.read_mag()` returns | `sensor_msgs/MagneticField` is defined in tesla; the [state estimator change](state_estimator.md) expects it |
| Add hard-iron and soft-iron calibration (offset and scale parameters) | The motors and battery distort the field, and Mahony needs a clean heading |
| Choose `timer_period` and `cutoff_frequency` together, so the sample rate is more than twice the cutoff | At 10 Hz sampling, a 25 Hz low-pass filter does nothing |
