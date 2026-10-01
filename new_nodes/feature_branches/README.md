# 0. In-development nodes (feature branches)

This directory documents what is already written on `feature/*` branches, as it is **today**, so that new work plugs into it. The edits these nodes need are in [changes_to_existing/](../changes_to_existing/), and the merge blockers are in [integration_fixes.md](../integration_fixes.md).

| Node | Package | Branch | Used by autonomy |
| --- | --- | --- | --- |
| [imu_driver_node](imu_driver_node.md) | `uuv_imu_driver` | `feature/sensor-imu` | yes |
| [compass_driver_node](compass_driver_node.md) | `uuv_compass_driver` | `feature/sensor-compass` | yes |
| [external_barometer_publisher](external_barometer_publisher.md) | `uuv_baro_ext` | `feature/sensor-ext-barometer` | yes |
| [uuv_dvl_driver](../dvl_reading/uuv_dvl_driver.md) | `uuv_dvl_driver` | `feature/dvl-driver` | yes. Only the payload parser is written; the full design is in the DVL Reading task. |
| [state_estimator](state_estimator.md) | `uuv_state_estimator` | `feature/control-state_estimator` | yes |
| [bms_node](bms_node.md) | `bms-comm` | `feature/bms-comm` | yes (safety) |
| [adc_publisher](adc_publisher.md) | `uuv_adc_driver` | `feature/sensor-adc` | no (operator) |
| [uuv_camera_driver](uuv_camera_driver.md) | `camera_ros` | `feature/sensor-camera` | no (operator) |
| [uuv_dashboard](uuv_dashboard.md) | `uuv_dashboard` | `feature/dashboard` | no (operator) |
