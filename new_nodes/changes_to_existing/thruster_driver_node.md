# C3. Thruster Driver (`thruster_driver_node`, package `uuv_motion_converter`): CHANGED

**Lives on:** `dev`. Current behaviour: [nodes.md](../../nodes.md#thruster-driver-thruster_driver_node-package-uuv_motion_converter).

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | Unchanged topic. It is now published by `safety_monitor`, not by `motion_converter_node`. |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/thrusters | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | **Changed:** 0.055 ≤ duty ≤ 0.095 (1100-1900 µs at 50 Hz) | **Changed:** the timeout also publishes neutral (0.075) |

**New parameters:** `command_timeout_s` (0.5)

### Functionality (change):

Add a timer, at 20 Hz for example. If no valid `/thruster/command` has arrived within `command_timeout_s`, publish neutral (0.075) on all 8 channels. This is the last line of defence when `safety_monitor`, or anything upstream of it, crashes.

The 1100-1900 µs pulse limits (instead of `dev`'s 1000-2000 µs) come from branch `feature/bms-comm`. Keep them when that branch merges.
