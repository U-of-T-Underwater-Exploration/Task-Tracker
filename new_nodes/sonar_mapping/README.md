# 6. Sonar Mapping

**Goal:** build a map of the lake or pool from the [Ping360](https://bluerobotics.com/store/sonars/imaging-sonars/ping360-sonar-r1-rp/). **Before choosing a SLAM method, ask Prof. Busheer what he uses for lake mapping.** Phase 1 below is odometry-based mapping, which works whatever SLAM method comes later. See the [underwater SLAM survey](https://www.mdpi.com/2072-4292/15/10/2496) for options.

| Node | Status | File |
| --- | --- | --- |
| `ping360_node` | third-party | [ping360_node.md](ping360_node.md) |
| `sonar_mapper` | new | [sonar_mapper.md](sonar_mapper.md) |

**Map format decision:**
- Use `nav_msgs/OccupancyGrid`, the standard Nav2 type. Save it with `ros2 run nav2_map_server map_saver_cli`, which writes a `.pgm` and a `.yaml`.
- The map is 2D because the vehicle holds depth with the barometer. Move to a 3D OctoMap only if multi-depth mapping is needed.
- Phase 1 uses the static identity `map → odom` from the top-level launch file. Phase 2 (SLAM) replaces it, publishing a real `map → odom` to correct the DVL drift.

`sonar_link` must be added to the URDF ([uuv_urdf.md](../changes_to_existing/uuv_urdf.md)).
