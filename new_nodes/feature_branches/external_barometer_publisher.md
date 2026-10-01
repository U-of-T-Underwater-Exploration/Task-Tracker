# External Barometer (`external_barometer_publisher`, package `uuv_baro_ext`, branch `feature/sensor-ext-barometer`)

**Status:** branch. Required edits: [changes_to_existing/external_barometer_publisher.md](../changes_to_existing/external_barometer_publisher.md).

**Subscription(s):** none. It reads the Bar30 (MS5837-30BA) over I2C bus 6.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /baro/external/data_raw | *sensor_msgs/msg/FluidPressure* | {Pa} | – | Raw water pressure |
| /baro/external/data | *sensor_msgs/msg/FluidPressure* | {Pa} | – | Low-pass filtered water pressure. **The state estimator uses this for depth.** |
| /baro/external/temperature_raw | *sensor_msgs/msg/Temperature* | {°C} | – | Raw water temperature |
| /baro/external/temperature | *sensor_msgs/msg/Temperature* | {°C} | – | Filtered water temperature |

**Parameters:** `publish_rate` (Hz), `frame_id`, `cutoff_frequency`, `sampling_frequency`. A fluid density is hard-coded to fresh water, but nothing uses it, because this node does not compute depth.

### Functionality:

Reads pressure and temperature, applies an exponential low-pass filter, and publishes. Depth is not computed here; the state estimator converts pressure to depth using its own `water_density` parameter.
