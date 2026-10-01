# DVL Driver (`uuv_dvl_driver`, package `uuv_dvl_driver`, branch `feature/dvl-driver`)

**Status:** branch, to be completed. Only the payload parser is written today: `parse_dvl_vel()` decodes the 68-byte `ID_DVL_VEL` payload into a map of field names to values. The node file `uuv_dvl_driver.cpp` is empty.

**Subscription(s):** none. It reads the serial port directly.

**Publish(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /dvl/raw | *uuv_msgs/msg/KoggerDvlVel* <br>`{uint32 flags, device_timestamp_ms; float32 delta_time, latency, vel_x/y/z/z1/z2, unc_x/y/z/z1/z2, dist_z/z1/z2}` | {$\varnothing$, ms, s, m/s, m} | – | Every field of `ID_DVL_VEL` v2, unmodified (see [uuv_msgs.md](../uuv_msgs.md)) |
| /dvl/twist | *geometry_msgs/msg/TwistWithCovarianceStamped* | {m/s} | frame `dvl_link` | VELOCITY_X/Y/Z, with covariance on the diagonal equal to UNCERTAINTY². The state estimator reads this. |
| /dvl/altitude | *sensor_msgs/msg/Range* | {m} | min/max range from the datasheet | DISTANCE_Z, the height above the seabed |
| /dvl/status | *diagnostic_msgs/msg/DiagnosticArray* | – | – | Frame rate, checksum errors, unknown-ID counts, raw FLAGS value |

**Parameters:** `port` (/dev/ttyUSB0), `baudrate`, `frame_id` (dvl_link), `max_valid_uncertainty` (m/s)

### Functionality:

Reads serial bytes and finds `0xBB 0x55`. It checks LENGTH and the Fletcher-16 checksum, then dispatches frames by ID. For 0x79 it reuses the existing `parse_dvl_vel()`. Replacing that function's `std::map` output with a packed struct would be faster and avoid string lookups.

`ID_DVL_VEL` v2 is published on all three DVL data topics. Unknown IDs are counted on `/dvl/status`, so the profiler can report them.

Samples are dropped from `/dvl/twist` and `/dvl/altitude` when the value is NaN or the uncertainty exceeds `max_valid_uncertainty`. `/dvl/raw` always publishes every sample, so downstream nodes can still tell that the driver is alive when bottom lock is lost.

The timestamp is the receive time minus `LATENCY`.

`dvl_link` must be added to the URDF ([uuv_urdf.md](../changes_to_existing/uuv_urdf.md)).
