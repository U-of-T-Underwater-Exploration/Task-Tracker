# Adaptive Controller (`adaptive_controller`, package `uuv_control`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, rad, m/s, rad/s} | frame `odom`, twist in `base_link` | Actual state η (pose), ν (body velocity) |
| /control/reference | *uuv_msgs/msg/TrajectoryReference* | {m, rad, m/s, m/s²} | frame `map`, all terms in world frame | Desired η_d, η̇_d, η̈_d |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /control/wrench | *geometry_msgs/msg/WrenchStamped* | {N, N·m} | frame `base_link`; clamped to the max wrench | Body-frame force and torque, sent to `motion_converter_node` |
| /control/params | *std_msgs/msg/Float64MultiArray* | mixed | within `theta_min` to `theta_max` | Current estimate θ̂, for logging and tuning |

**Parameters:** `Lambda` (6 diagonal gains), `K_D` (6 diagonal gains), `Gamma` (one entry per parameter), `theta_init`, `theta_min`, `theta_max`, `deadzone` (on ‖s‖), `rate_hz` (50), `adapt_enabled`

### Functionality:

**Frames.** The reference is in `map` and the state is in `odom`. Transform the reference into `odom` with TF (identity until SLAM publishes `map → odom`). Both topics are ENU/FLU; Fossen's model is usually written in NED/FRD, so convert once at the input and use a single convention internally. Convert the output wrench back to FLU before publishing.

**Model.** The vehicle follows Fossen's model: M ν̇ + C(ν)ν + D(ν)ν + g(η) = τ.

The unknown parameters θ appear linearly, so the left side can be written as Y(η, ν, ν_r, ν̇_r) θ. The parameters are:
- Rigid-body and added mass
- Linear and quadratic damping
- Net buoyancy and the offset between the centre of gravity and the centre of buoyancy

**Each step:**
1. Compute the tracking error η̃ = η − η_d and the composite error s = η̃̇ + Λ η̃.
2. Compute the reference velocity η̇_r = η̇_d − Λ η̃. Convert it to the body frame with ν_r = J(η)⁻¹ η̇_r.
3. Control law: τ = Y θ̂ − K_D s, with s converted to the body frame.
4. Gradient update: θ̂̇ = −Γ Yᵀ s.
5. Integrate the update and clamp θ̂ to its bounds (projection).

Skip the update when ‖s‖ < `deadzone`, so that sensor noise does not make the parameters drift.

**Testing.** Tune in simulation first. With adaptation off and θ̂ fixed, the controller is a plain computed-torque controller, which makes a good baseline. Then turn adaptation on and check that θ̂ converges and that tracking error falls. Repeat in the pool.
