# Global Planner: A* (`global_planner`, package `uuv_planning`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /map | *nav_msgs/msg/OccupancyGrid* | {m/cell} | frame `map` | Static or sonar-built map |
| /state_estimate | *nav_msgs/msg/Odometry* | {m} | frame `odom`, transformed into `map` with TF | Start position |
| /goal_pose | *geometry_msgs/msg/PoseStamped* | {m, rad} | frame `map`; z = −(target depth) | Goal, for example clicked in RViz or sent by a mission script |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /plan | *nav_msgs/msg/Path* | {m} | frame `map`; every waypoint has the goal's z | Collision-free list of waypoints |

**Parameters:** `robot_radius` (m, for inflating obstacles), `occupied_threshold` (50), `allow_unknown` (false), `replan_period` (s)

### Functionality:

1. Inflate obstacles by `robot_radius`.
2. Run A* on the 8-connected grid. The cost is the step distance and the heuristic is the Euclidean distance to the goal.
3. Remove waypoints that have line of sight to each other, to shorten the path.
4. Publish the result.

It replans when a new goal arrives, when the map changes along the path, or every `replan_period`. If there is no path, it logs an error and publishes an empty `Path`.
