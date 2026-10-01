# C7. External Barometer (`external_barometer_publisher`, package `uuv_baro_ext`): CHANGED

**Lives on:** `feature/sensor-ext-barometer`. Current behaviour: [feature_branches/external_barometer_publisher.md](../feature_branches/external_barometer_publisher.md).

Topics, types and units stay the same.

| Change | Why |
| --- | --- |
| Remove `namespace=` from the launch file, so the topic is `/baro/external/data` | The dashboard, `state_estimator`, `safety_monitor` and `sim_bridge` all expect this name |
| Set `frame_id` to `baro_ext_link` | Match the URDF link name |
| Fix the YAML key `ros_parameters` → `ros__parameters` in `uuv_baro_ext_config.yaml` | Otherwise the parameters are not loaded |
| Remove the hard-coded fluid density | It is unused: depth is computed in `state_estimator`, whose `water_density` parameter covers fresh and salt water |
