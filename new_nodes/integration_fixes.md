# Integration fixes for feature branches

These need fixing before the branches merge into `dev` and the new nodes can rely on them. The node-level changes that follow from them are in [changes_to_existing/](changes_to_existing/).

| Branch | Issue | Fix |
| --- | --- | --- |
| all feature branches | They branched from an old `main`, so a plain diff against `dev` deletes `uuv_joystick_hal`, `uuv_motion_converter` and the other `dev` packages | Rebase or merge onto `dev` before opening a PR, so that only the new package is added |
| sensor-compass, sensor-ext-barometer, sensor-adc, control-state_estimator | Launch files use the namespaces `utux` or `utux/sensor`, so the topics become `/utux/sensor/baro/external/data` and so on. The dashboard subscribes to `/baro/external/data`. In `compass_driver_launch.py` the `namespace` line is also missing a comma (syntax error). | Remove `namespace=` everywhere (see the [conventions](README.md#conventions-apply-to-every-node)); this also removes the syntax error |
| sensor-imu, sensor-compass | The header stamp is time since the node started, not ROS time | Use `self.get_clock().now().to_msg()`, as the barometer node already does |
| sensor-imu, sensor-compass | They publish zeros when a read fails, which looks like valid data | Skip publishing on a failed read, so that the safety timeout trips |
| sensor-ext-barometer | `frame_id` is `baro_external_link`, but the URDF link is `baro_ext_link`. `uuv_baro_ext_config.yaml` uses `ros_parameters` (missing `__`). | Match the URDF name, and fix the YAML key |
| control-state_estimator | Does not compile. It has name mismatches (`T_IMUToBase` vs `T_IMUToBase_`, `kalman_filter` vs `kalman_filter_`/`kalman_filer`, `q_bodyToWorld`, `mag_ref`), missing semicolons in the parameter declarations, and assigns Eigen vectors directly to ROS message fields | Fix the build first, then add the barometer and DVL updates |
| bms-comm | The package is named `bms-comm`; ROS 2 package names may not contain hyphens | Rename the package to `bms_comm` |
| dvl-driver | The node file is empty | Implement it as in [uuv_dvl_driver.md](dvl_reading/uuv_dvl_driver.md) |
| dashboard | Subscribes to `/baro/internal/*`, but no node publishes it | Add an internal barometer driver, or remove those widgets |
| imu, compass, adc, pwm_driver | Four separate processes each call `navigator.init()` | Confirm that the library allows this. If it doesn't, combine them into one Navigator process. |
