# Stonefish Simulator (`stonefish_simulator`, package `stonefish_ros2`, third-party)

**Status:** third-party.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sim/thrusters | *std_msgs/msg/Float64MultiArray* <br>`{double[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | Thruster setpoints, in the order the thrusters appear in the scenario file |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sim/imu | *sensor_msgs/msg/Imu* | {rad/s, m/s²} | frame `imu_link` | Simulated IMU |
| /sim/dvl | *stonefish_ros2/msg/DVL* | {m/s, m} | – | Simulated DVL velocity and beam ranges |
| /sim/dvl/altitude | *sensor_msgs/msg/Range* | {m} | frame `dvl_link` | Altitude above the seabed |
| /sim/pressure | *sensor_msgs/msg/FluidPressure* | {Pa} | frame `baro_ext_link` | Simulated Bar30 |
| /sim/odometry | *nav_msgs/msg/Odometry* | {m, rad, m/s} | Stonefish world frame (NED) | **Ground truth**, for scoring only. `sim_bridge` republishes it in ENU. |
| /sim/sonar | *sensor_msgs/msg/Image* | {intensity} | frame `sonar_link` | Mechanical scanning imaging sonar (Ping360-like) |

**Parameters:** scenario file `uuv_sim/scenarios/uuv.scn`, simulation rate, graphics on or off

### Functionality:

Runs the physics: buoyancy, hydrodynamic drag, thrusters and sensors. The robot is defined in an XML scenario file:
- **Meshes:** export two `.obj` files from CAD, a detailed visual mesh and a simplified physics mesh.
- **Mass properties:** mass, centre of gravity and inertia, taken from CAD.
- **Thrusters:** positions and directions copied from `uuv_motion_converter/config/params.yaml`, so that the simulated robot matches the allocation matrix.
- **Sensors:** each one gets a `<ros_publisher topic=...>` tag. Name each sensor's frame after its URDF link (as in the table above), so the topics that are only remapped carry the right `frame_id`. Mount the sensors at the same offsets as the URDF.

Stonefish's world frame is NED. Check the IMU's body axes against what the real `imu_driver_node` publishes after its sign flips, and correct any difference in `sim_bridge`.

Graphics need OpenGL 4.3 or newer. The Stonefish library version must match the `stonefish_ros2` version.
