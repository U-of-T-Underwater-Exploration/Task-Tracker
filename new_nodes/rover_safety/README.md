# 3. Rover Safety

**Goal:** detect sensor or state malfunctions and cut motor commands.

| Node | Status | File |
| --- | --- | --- |
| `safety_monitor` | new | [safety_monitor.md](safety_monitor.md) |

Existing-node changes that ship with this task:
- [motion_converter_node](../changes_to_existing/motion_converter_node.md): rename its output to `/thruster/command_raw`, so `safety_monitor` sits between it and `thruster_driver_node`.
- [thruster_driver_node](../changes_to_existing/thruster_driver_node.md): a backstop timeout in case `safety_monitor` dies.

Seabed avoidance from the DVL altitude (part of the "Using the DVL" task) is the **Seabed** check in `safety_monitor`.
