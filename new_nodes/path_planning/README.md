# 7. Path Planning

**Goal:** plan a collision-free route on the sonar map with A*, then turn it into a smooth reference for the controller.

| Node | Status | Role | File |
| --- | --- | --- | --- |
| `global_planner` | new | A* on `/map` → `/plan` | [global_planner.md](global_planner.md) |
| `path_follower` | new | Follows `/plan` with a lookahead point → `/control/reference` | [path_follower.md](path_follower.md) |

The map format is decided in [Sonar Mapping](../sonar_mapping/README.md): `nav_msgs/OccupancyGrid`. The global planner also accepts a static map loaded with `nav2_map_server`.
