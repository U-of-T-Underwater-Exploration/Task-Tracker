# Battery Management (`bms_node`, package `bms-comm`, branch `feature/bms-comm`)

**Status:** branch. The package must be renamed to `bms_comm` (see [Integration fixes](../integration_fixes.md)).

**Subscription(s):** none. It polls the JK-BMS over UART (`/dev/ttyS0`, 115200 baud).

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /bms/data | *sensor_msgs/msg/BatteryState* | {V, A, %} | 8 cells (8S) | Pack voltage, current, state of charge, cell voltages and health |
| /bms/temperature | *std_msgs/msg/Float32MultiArray* <br>`{float[3]}` | {°C} | – | [0] MOSFET, [1] left probe, [2] right probe |

**Parameters:** `serial_port`, `baud_rate`, `publish_frequency` (2 Hz)

### Functionality:

Parses the BMS serial protocol and publishes battery state for the dashboard. In this design it also feeds [`safety_monitor`](../rover_safety/safety_monitor.md) (low voltage and over-temperature checks).

The same branch also changes the thruster pulse limits to 1100-1900 µs and makes `pwm_driver` start at a neutral duty of 0.075. [thruster_driver_node](../changes_to_existing/thruster_driver_node.md) keeps those limits.
