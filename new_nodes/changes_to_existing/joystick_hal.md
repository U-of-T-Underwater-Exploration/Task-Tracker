# C1. Joystick HAL (`joystick_hal`, package `uuv_joystick_hal`): CHANGED

**Lives on:** `dev`. Current behaviour: [nodes.md](../../nodes.md#joystick-hal-joystick_hal-package-uuv_joystick_hal).

**Subscription(s):** unchanged (`joy`).

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | {$\varnothing$ (fraction of max)} | **Changed:** `mode` ∈ {0, 1} | `mode` was always 0. It now toggles between 0 (manual) and 1 (autonomous). |

**New parameters:** `mode_toggle_button` (button index). Do not use 4 or 5; `motion_converter_node` already uses those to flip heave and yaw.

### Functionality (change):

On each PRESSED event of `mode_toggle_button`, flip an internal `mode` flag and put it in every published `UUVCommand`. The node keeps publishing in autonomous mode, so `/input/command` stays the operator heartbeat that [`safety_monitor`](../rover_safety/safety_monitor.md) watches in both modes. Moving the sticks in auto mode has no effect; the operator stops the vehicle by toggling back or by releasing the controller (heartbeat timeout).
