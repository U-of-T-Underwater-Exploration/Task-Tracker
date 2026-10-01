# Compass Driver (`compass_driver_node`, package `uuv_compass_driver`, branch `feature/sensor-compass`)

**Status:** branch. Required edits: [changes_to_existing/compass_driver_node.md](../changes_to_existing/compass_driver_node.md).

**Subscription(s):** none. It reads the Navigator magnetometer.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /compass/data_raw | *sensor_msgs/msg/MagneticField* | {as returned by the library; check whether µT or T} | frame `compass_link` | Raw magnetic field |
| /compass/data | *sensor_msgs/msg/MagneticField* | same | frame `compass_link` | Low-pass filtered magnetic field |

**Parameters:** `timer_period` (0.1 s), `cutoff_frequency` (25 Hz), `coordinate_frame` (±1 per axis)

### Functionality:

Reads the magnetometer, low-pass filters it, applies the axis sign flips, and publishes. There is no hard-iron or soft-iron calibration yet. The state estimator needs that calibration for a usable heading near the vehicle's motors and battery.

`sensor_msgs/MagneticField` is defined in tesla. Whether the library returns µT or T is an [open question](../README.md#open-questions).
