# Dashboard (`uuv_dashboard`, PyQt6 GUI, branch `feature/dashboard`)

**Status:** branch. Used by the operator, not by autonomy.

**Subscription(s):**

| Topic | Type | Description |
| --- | --- | --- |
| /baro/external/data, /baro/external/temperature | *sensor_msgs/msg/FluidPressure*, *sensor_msgs/msg/Temperature* | Water pressure and temperature |
| /baro/internal/data, /baro/internal/temperature | same | Hull pressure and temperature. **No node publishes these yet** (see [Integration fixes](../integration_fixes.md)). |
| /bms/data, /bms/temperature | *sensor_msgs/msg/BatteryState*, *std_msgs/msg/Float32MultiArray* | Battery state |
| /thruster/command | *std_msgs/msg/Float32MultiArray* | Thruster commands. After the [motion_converter_node change](../changes_to_existing/motion_converter_node.md) this is the post-safety command, which is still what the operator should see. |

**Publish(s):** none.

### Functionality:

Displays the topics above for the operator. A suggested addition is a widget for `/safety/status` from [safety_monitor](../rover_safety/safety_monitor.md).
