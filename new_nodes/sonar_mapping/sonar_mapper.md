# Sonar Mapper (`sonar_mapper`, package `uuv_sonar`)

**Status:** new.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sonar/echo | *ping360_sonar_msgs/msg/SonarEcho* | {deg, m} | frame `sonar_link` | Raw beams |
| /tf, /tf_static | `map → odom → base_link → sonar_link` | {m, rad} | – | Sonar pose at the time each beam fired. `odom → base_link` comes from `state_estimator`. |

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /map | *nav_msgs/msg/OccupancyGrid* | {m/cell} | -1 = unknown, 0 = free, 100 = occupied | 2D map at the operating depth, frame `map` |

**Parameters:** `resolution` (0.2 m), `map_size` (m), `intensity_threshold`, `min_range` (to ignore near-field ringing)

### Functionality:

For each beam:
1. Threshold the intensity samples, ignoring those closer than `min_range`.
2. Mark the first strong return as occupied and the cells before it as free.
3. Transform the cells into `map` using the TF lookup at the beam's timestamp.
4. Update the cells with a log-odds update.

The Ping360 is a slowly rotating single beam, so the vehicle moves during a sweep. Always use the pose at the exact beam timestamp, never the pose at the end of the sweep.
