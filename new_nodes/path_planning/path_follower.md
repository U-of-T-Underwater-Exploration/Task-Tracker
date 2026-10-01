# Path Follower (`path_follower`, package `uuv_planning`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /plan | *nav_msgs/msg/Path* | {m} | frame `map` | Path from A* |
| /state_estimate | *nav_msgs/msg/Odometry* | {m, m/s} | frame `odom`, transformed into `map` with TF | Current state |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /control/reference | *uuv_msgs/msg/TrajectoryReference* <br>`{Header; Pose pose; Twist twist; Accel accel}` | {m, rad, m/s, m/s²} | frame `map`; speed ≤ `max_speed` | Desired pose, velocity and acceleration for the controller, all in the world frame |

**Parameters:** `lookahead` (m), `max_speed` (m/s), `max_accel` (m/s²), `goal_tolerance` (m), `rate_hz` (10)

### Functionality:

Turns A*'s waypoints into a smooth reference the adaptive controller can track. On each tick:
1. Find the closest point on `/plan`, then the **target point** `lookahead` metres further along the path.
2. The desired velocity points toward the target, with magnitude `max_speed`. It slows down linearly inside the last `lookahead` metres before the goal.
3. Limit the change in velocity to `max_accel`, which gives the acceleration term.
4. Integrate the velocity into the reference pose. Heading points along the velocity, and z comes from the plan.

When the vehicle is within `goal_tolerance` of the goal, it holds the goal pose with zero velocity. A new `/plan` (after a replan) takes effect on the next tick. Because the reference is integrated rather than jumping, the switch is smooth.

Obstacles not yet in the map are handled by replanning: once `sonar_mapper` adds them to `/map`, `global_planner` replans around them.
