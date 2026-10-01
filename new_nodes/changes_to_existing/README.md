# Changes to existing nodes (modified, not new)

Every node in this directory **already exists**, either on `dev` or on a feature branch. Only the differences from today's behaviour are described: rows tagged **New**, **Renamed** or **Changed** are the edits, and untagged rows stay as they are. For a node's current behaviour, see [`nodes.md`](../../nodes.md) (for `dev` nodes) or [feature_branches/](../feature_branches/) (for branch nodes).

| # | Node | Lives on | Ships with task | Why it changes |
| --- | --- | --- | --- | --- |
| C1 | [joystick_hal](joystick_hal.md) | `dev` | 5. Adaptive Controller | The operator needs to switch between manual and autonomous mode |
| C2 | [motion_converter_node](motion_converter_node.md) | `dev` | 3. Rover Safety (rename), 5. Adaptive Controller (auto mode) | Send output through `safety_monitor`, and accept the controller's wrench |
| C3 | [thruster_driver_node](thruster_driver_node.md) | `dev` | 3. Rover Safety | A backstop timeout in case `safety_monitor` dies |
| C4 | [state_estimator](state_estimator.md) | `feature/control-state_estimator` | 4. State Estimation | Fuse the barometer and DVL; publish in ENU; broadcast TF |
| C5 | [imu_driver_node](imu_driver_node.md) | `feature/sensor-imu` | 0. Merge sensor branches | Topic names, timestamps and failure behaviour that downstream nodes rely on |
| C6 | [compass_driver_node](compass_driver_node.md) | `feature/sensor-compass` | 0. Merge sensor branches | As C5, plus calibration and units |
| C7 | [external_barometer_publisher](external_barometer_publisher.md) | `feature/sensor-ext-barometer` | 0. Merge sensor branches | Topic names and frame name |
| – | [uuv.urdf.xacro](uuv_urdf.md) (not a node) | `dev` | 1. DVL Reading, 6. Sonar Mapping | Add `dvl_link` and `sonar_link` with their measured mounting offsets |
