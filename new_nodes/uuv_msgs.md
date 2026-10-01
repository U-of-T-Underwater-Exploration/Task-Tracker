# New messages (`uuv_msgs`)

| Message | Fields | Used by |
| --- | --- | --- |
| `KoggerDvlVel` | `Header header`; `uint32 flags`; `uint32 device_timestamp_ms`; `float32 delta_time, latency`; `float32 vel_x, vel_y, vel_z, vel_z1, vel_z2`; `float32 unc_x, unc_y, unc_z, unc_z1, unc_z2`; `float32 dist_z, dist_z1, dist_z2` | [uuv_dvl_driver](dvl_reading/uuv_dvl_driver.md) → [dvl_profiler](dvl_reading/dvl_profiler.md), [safety_monitor](rover_safety/safety_monitor.md) |
| `TrajectoryReference` | `Header header`; `geometry_msgs/Pose pose`; `geometry_msgs/Twist twist`; `geometry_msgs/Accel accel`. Pose, twist and accel are all expressed in `header.frame_id` (world frame), not the body frame. | [path_follower](path_planning/path_follower.md) → [adaptive_controller](adaptive_controller/adaptive_controller.md) |
| `SafetyStatus` | `Header header`; `uint8 state` (OK = 0, WARN = 1, FAULT = 2); `string[] reasons` | [safety_monitor](rover_safety/safety_monitor.md) → dashboard |
