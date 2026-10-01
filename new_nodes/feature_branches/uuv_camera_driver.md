# Camera Driver (`uuv_camera_driver`, package `camera_ros`, branch `feature/sensor-camera`)

**Status:** branch. Used by the operator, not by autonomy.

**Subscription(s):** none.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /camera/* | *sensor_msgs/msg/Image* (and camera info) | – | 640×480 RGB888, frame `cam_link` | Camera images |

### Functionality:

Wraps `camera_ros` to stream the vehicle camera.
