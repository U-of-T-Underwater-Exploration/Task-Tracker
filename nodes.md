# uuv-code (`dev` branch): Node Overview

Data flow:

```
joy → joystick_hal → /input/command → motion_converter_node → /thruster/command
    → thruster_driver_node → /pwm/thrusters → pwm_driver → Navigator PWM ch 0-7

/pwm/servo → servo_converter → /pwm/command → pwm_driver → Navigator PWM ch 8-15
                                              pwm_driver → /pwm/generated
```

---

# Joystick HAL (`joystick_hal`, package `uuv_joystick_hal`)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| joy | *sensor_msgs/msg/Joy* | $\varnothing$ | axes ∈ [-1.0, 1.0]; buttons ∈ {0, 1} | Raw controller state from an external `joy_node`. Reliable QoS, depth 10. |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* <br>`{float surge, sway, heave, roll, pitch, yaw; ActionCommand[] actions; uint8 mode}` | {$\varnothing$ (fraction of max)} | each axis ∈ [-1.0, 1.0]; `mode` = 0 | 6-DOF command plus the list of button events |

### Functionality:

Maps joystick axes to a 6-DOF command: axis 1 → surge, 3 → sway, 2 → heave, 0 → roll, 4 → pitch, 5 → yaw. A missing axis is sent as 0.0.

The node keeps the previous button state. For each button it emits an `ActionCommand {action = button index, state}` with state PRESSED (0 → 1), HELD (1 → 1) or RELEASED (1 → 0). Buttons that are INACTIVE are left out of the list.

The axis and button mapping is hard-coded and is marked in the code as still to be confirmed for the controller in use.

---

# Motion Converter (`motion_converter_node`, package `uuv_motion_converter`)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /input/command | *uuv_joystick_msgs/msg/UUVCommand* | {$\varnothing$ (fraction of max)} | axes ∈ [-1.0, 1.0] | Desired 6-DOF motion and button actions |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {$\varnothing$ (fraction of max thrust)} | -1.0 ≤ cmd ≤ 1.0 | Normalised command per thruster, indexed by thruster `id` |

### Functionality:

Thruster allocation. At startup the node reads the 8 thrusters' settings from `config/params.yaml`: `id`, `p_motor_offset` (position), `r_motor_dir` (thrust direction), `max_forward_thrust`, `max_reverse_thrust`, `enabled` and `inverted`.

It builds a 6×8 allocation matrix. Rows 0-2 hold the force direction, and rows 3-5 hold the moment (position × direction). It takes the pseudo-inverse (complete orthogonal decomposition). It also computes the maximum achievable force or moment on each axis.

On each `/input/command`:
1. Scale each axis fraction by that axis's maximum force or moment to get a wrench.
2. Multiply by the pseudo-inverse to get the thrust per thruster.
3. If any thruster would exceed its limit, divide all thrusts by the worst ratio so that the direction is preserved.
4. Map thrust to [-1, 1] using the forward or reverse maximum.
5. Flip the sign for `inverted` thrusters and clamp to [-1, 1].
6. Write each result at index `id`.

Button action 4 flips the sign of heave, and action 5 flips the sign of yaw.

---

# Thruster Driver (`thruster_driver_node`, package `uuv_motion_converter`)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /thruster/command | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {$\varnothing$ (fraction of max thrust)} | length = 8; -1.0 ≤ cmd ≤ 1.0 | Normalised thrust command per thruster |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/thrusters | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | 0.05 ≤ duty ≤ 0.10 | PWM duty cycle per thruster |

### Functionality:

Converts normalised thrust to PWM duty cycle at 50 Hz. The mapping is linear, from -1.0 → 1000 µs (duty 0.05) to +1.0 → 2000 µs (duty 0.10), so 0.0 → 1500 µs (duty 0.075).

Values outside [-1, 1] are clamped and an error is logged. An array whose length is not 8 is rejected and nothing is published. The frequency and pulse widths are hard-coded.

---

# PWM Driver (`pwm_driver`, package `pwm_driver`)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/command | *pwm_msg/msg/CommandPWM* <br>`{float pwm_values, int8 pwm_channelnumber}` | {%, $\varnothing$} | 8 ≤ n ≤ 15 <br> 0 ≤ duty ≤ 1.0 | Channel to command and its duty cycle. (0 ⇒ Channel 1) |
| /pwm/thrusters | *std_msgs/msg/Float32MultiArray* <br>`{float[8]}` | {%} | length = 8 <br> 0 ≤ duty ≤ 1.0 | 8-array to set all thruster channels (0-7) at the same time |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/generated | *pwm_msg/msg/ThrustersPWM* <br>`{float[16] pwm_values, bool[16] pwm_valid}` | {%, $\varnothing$} | pwm_values ∈ [0, 1.0] | The currently generated PWM signals, and whether each one is valid |

**Parameters:** `pwm_frequency_hz` (default 50)

### Functionality:

Hardware interface to the Blue Robotics Navigator (`bluerobotics_navigator`). At startup it initialises the Navigator, sets the PWM frequency and enables PWM output.

Both subscriptions do the same thing: they store the channel and duty cycle in the node's internal state, write them to the Navigator, then publish the full 16-channel state on `/pwm/generated`.

`/pwm/command` is for general-purpose, one-at-a-time setting of the auxiliary channels (8-15). `/pwm/thrusters` is for setting all 8 thruster channels (0-7) at once.

An out-of-range duty is replaced with 0.0 and its channel is marked invalid. A channel outside 8-15 on `/pwm/command` is rejected. Navigator errors are logged and the channel is marked invalid.

---

# Servo Converter (`servo_converter`, package `pwm_driver`)

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/servo | *pwm_msg/msg/ServoPWM* <br>`{float servo_angle, int8 pwm_channelnumber}` | {deg, $\varnothing$} | `servo_min_angle` ≤ angle ≤ `servo_max_angle` (default 0-180) | Servo angle and the PWM channel it is on |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /pwm/command | *pwm_msg/msg/CommandPWM* <br>`{float, int8}` | {%, $\varnothing$} | 0 ≤ duty ≤ 1.0 | Duty cycle for that channel, passed on to `pwm_driver` |

**Parameters:** `servo_min_pulse_width` (500 µs), `servo_max_pulse_width` (2500 µs), `servo_min_angle` (0), `servo_max_angle` (180), `pwm_frequency_hz` (50)

### Functionality:

Converts a servo angle to a duty cycle. The angle is clamped to the min/max angle (a warning is logged). It is mapped linearly to a pulse width between the min and max pulse widths, then multiplied by `pwm_frequency_hz` to get the duty cycle.

If the angle range is zero, it logs an error and outputs a duty of 0.0.

---

# Robot State Publisher (`robot_state_publisher`, package `uuv_description`, launch only)

**Subscription(s):** `/joint_states` (standard, if any joints are published)

**Publish(s):** `/tf`, `/tf_static`, `/robot_description`

### Functionality:

Publishes the transforms and robot description from `urdf/uuv.urdf.xacro`. It is started by `uuv_tf.launch.py`.

---

# Messages

| Message | Fields |
| --- | --- |
| `uuv_joystick_msgs/UUVCommand` | `float32 surge, sway, heave, roll, pitch, yaw`; `ActionCommand[] actions`; `uint8 mode` |
| `uuv_joystick_msgs/ActionCommand` | `uint16 action` (button index); `uint8 state` (INACTIVE=0, PRESSED=1, HELD=2, RELEASED=3) |
| `pwm_msg/CommandPWM` | `float32 pwm_values`; `int8 pwm_channelnumber` |
| `pwm_msg/ServoPWM` | `float32 servo_angle`; `int8 pwm_channelnumber` |
| `pwm_msg/ThrustersPWM` | `float32[16] pwm_values`; `bool[16] pwm_valid` |

# Launch files

- `uuv_motion_converter/launch/ultimate.launch.py`: top-level. Includes the motion-converter `bringup`, `joystick_hal`, `uuv_tf` and the pwm_driver `bringup`.
- `uuv_motion_converter/launch/bringup.launch.py`: `motion_converter_node` and `thruster_driver_node`.
- `pwm_driver/launch/bringup.launch.py`: `pwm_driver` and `servo_converter`.

# Observations

- Nothing in the repo publishes `joy` (expects a separate `joy_node`) or `/pwm/servo`.
- The `pwm_driver` launch files reference package `uuv_pwm_driver`, but the directory is `pwm_driver`. Check that it matches `package.xml`.
- `config/params.yaml` starts with the node key `utux:` rather than `/**` or a node name, which may stop the parameters from loading.
- `thruster_driver_node` hard-codes 50 Hz and 1000-2000 µs, while `pwm_driver` and `servo_converter` take the frequency as a parameter.
- `thruster_driver_node` leaves a channel at 0.0 duty if its input is NaN, because neither range branch matches.