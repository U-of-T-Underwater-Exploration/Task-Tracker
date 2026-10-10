# Adaptive Controller (`adaptive_controller`, package `uuv_control`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s, rad/s} | frame `odom`, twist in `base_link` | Actual state $\eta$ (pose), $\nu$ (body velocity) |
| /control/reference | *uuv_msgs/msg/TrajectoryReference* | {m, rad, m/s, m/s²} | frame `map`, all terms in world frame | Desired $\eta_d$, $\dot{\eta}_d$, $\ddot{\eta}_d$ |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | frame `base_link`; clamped to the max wrench | Body-frame force and torque, sent to `motion_converter_node` |
| /control/params | *std_msgs/msg/Float64MultiArray* | mixed | within `theta_min` to `theta_max` | Current estimate $\hat{\theta}$, for logging and tuning |

**Parameters:** `Lambda` (6 diagonal gains), `K_D` (6 diagonal gains), `Gamma` (one entry per parameter), `theta_init`, `theta_min`, `theta_max`, `deadzone` (on $\lVert s \rVert$), `rate_hz` (50), `adapt_enabled`

### Functionality:

**Frames.** The reference is in `map` and the state is in `odom`. Transform the reference into `odom` with TF (identity until SLAM publishes `map → odom`). Both topics are ENU/FLU; Fossen's model is usually written in NED/FRD, so convert once at the input and use a single convention internally. Convert the output wrench back to FLU before publishing.

**Model.** The vehicle follows Fossen's model:

$$
M \dot{\nu} + C(\nu)\nu + D(\nu)\nu + g(\eta) = \tau
$$

The unknown parameters $\theta$ appear linearly, so the left side can be written as $Y(\eta, \nu, \nu_r, \dot{\nu}_r)\,\theta$. The parameters are:
- Rigid-body and added mass
- Linear and quadratic damping
- Net buoyancy and the offset between the centre of gravity and the centre of buoyancy

**Each step:**
1. Compute the tracking error and the composite error:
   $$
   \tilde{\eta} = \eta - \eta_d, \qquad s = \dot{\tilde{\eta}} + \Lambda \tilde{\eta}
   $$
2. Compute the reference velocity and convert it to the body frame:
   $$
   \dot{\eta}_r = \dot{\eta}_d - \Lambda \tilde{\eta}, \qquad \nu_r = J(\eta)^{-1} \dot{\eta}_r
   $$
3. Control law, with $s$ converted to the body frame:
   $$
   \tau = Y \hat{\theta} - K_D\, s
   $$
4. Gradient update:
   $$
   \dot{\hat{\theta}} = -\Gamma\, Y^\top s
   $$
5. Integrate the update and clamp $\hat{\theta}$ to its bounds (projection).

Skip the update when $\lVert s \rVert <$ `deadzone`, so that sensor noise does not make the parameters drift.

**Testing.** Tune in simulation first. With adaptation off and $\hat{\theta}$ fixed, the controller is a plain computed-torque controller, which makes a good baseline. Then turn adaptation on and check that $\hat{\theta}$ converges and that tracking error falls. Repeat in the pool.
