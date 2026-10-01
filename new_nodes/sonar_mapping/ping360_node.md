# Ping360 Driver (`ping360_node`, package `ping360_sonar`)

**Status:** third-party, branch `ros2` of [CentraleNantesRobotics/ping360_sonar](https://github.com/CentraleNantesRobotics/ping360_sonar).

**Subscription(s):** none. It connects over UDP, by default 192.168.2.2:9092, or over serial at 115200 baud.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /sonar/echo | *ping360_sonar_msgs/msg/SonarEcho* <br>`{angle, gain, number_of_samples, transmit_frequency, speed_of_sound, range, intensities[]}` | {deg (check: the Ping360 protocol uses gradians), m, 0-255} | frame `sonar_link` | One beam: intensity against range at one angle |
| /sonar/scan | *sensor_msgs/msg/LaserScan* | {rad, m} | frame `sonar_link` | First strong return per angle. No node in this design consumes it yet; kept for RViz and future obstacle avoidance |
| /sonar/image | *sensor_msgs/msg/Image* | – | – | Polar sweep image, for the operator |

**Parameters:** `angle_sector` (60-360°), `angle_step` (1-20°), `range_max` (1-50 m), `frequency` (650-850 kHz)

### Functionality:

This node already exists upstream. The driver was written for Foxy, so check that it builds on our ROS 2 distro. Remap its topics into `/sonar/*` and set `frame_id = sonar_link`.

In simulation, [sim_bridge](../simulation/sim_bridge.md) publishes `/sonar/echo` and `/sonar/scan` instead.
