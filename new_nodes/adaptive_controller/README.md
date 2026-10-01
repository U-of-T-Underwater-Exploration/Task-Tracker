# 5. Adaptive Controller

**Goal:** a Slotine-Li adaptive controller whose model parameters are learned online with gradient updates.

| Node | Status | File |
| --- | --- | --- |
| `adaptive_controller` | new | [adaptive_controller.md](adaptive_controller.md) |

Existing-node changes that ship with this task:
- [joystick_hal](../changes_to_existing/joystick_hal.md): a button toggles between manual and autonomous mode.
- [motion_converter_node](../changes_to_existing/motion_converter_node.md): in autonomous mode, allocate `/control/wrench` instead of the joystick.

Until [Path Planning](../path_planning/) exists, feed `/control/reference` from a small test script (a step and a circle).
