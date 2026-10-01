# C2. Motion Converter (`motion_converter_node`, package `uuv_motion_converter`): CHANGED

**Lives on:** `dev`. Current behaviour: [nodes.md](../../nodes.md#motion-converter-motion_converter_node-package-uuv_motion_converter).

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | {$\varnothing$ (fraction of max)} | axes ∈ [-1.0, 1.0] | Unchanged. It is now also read for `mode`. |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | frame `base_link` | **New:** wrench from `adaptive_controller`, used only when `mode == 1` |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command_raw | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | -1.0 ≤ cmd ≤ 1.0 | **Renamed** from `/thruster/command`, so that `safety_monitor` can gate it |

### Functionality (change):

- **mode 0 (manual):** unchanged. The joystick fractions are scaled by the max wrench, then go through the pseudo-inverse, normalisation and mapping.
- **mode 1 (auto):** use the latest `/control/wrench` directly as the wrench. Skip the scaling by max force and moment, because the controller already outputs N and N·m. Then apply the same pseudo-inverse, saturation normalisation and per-thruster mapping as in manual mode.
- **Auto with no wrench:** if no `/control/wrench` has arrived in auto mode, publish zeros. A wrench that stops arriving later is caught by `safety_monitor`'s controller heartbeat.
- **Startup:** start in mode 0 until the first `/input/command` arrives.
- **When to rename:** land the rename in the same change as `safety_monitor` (task 3). Until then, nothing would publish `/thruster/command`.
- **Dashboard:** the dashboard subscribes to `/thruster/command`. That is still correct, because it now shows the post-safety command.
