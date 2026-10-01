# DVL Profiler (`dvl_profiler`, package `uuv_dvl_driver`)

**Status:** new. A bench and pool test tool.

**Subscription(s):**

| Topic | Type | Units | Range/Constraints | Description |
| --- | --- | --- | --- | --- |
| /dvl/raw | *uuv_msgs/msg/KoggerDvlVel* | as in [uuv_dvl_driver](uuv_dvl_driver.md) | – | Raw DVL data |
| /dvl/status | *diagnostic_msgs/msg/DiagnosticArray* | – | – | Driver statistics |

**Publish(s):** none. It writes `dvl_report.md` when it shuts down.

### Functionality:

Produces the **data profile** this task asks for:
- Which message IDs and versions appeared, and at what rate (Hz).
- Minimum, maximum, mean and standard deviation of each field, plus the NaN and zero counts.
- Each `FLAGS` bit pattern observed, alongside the test condition of that recording (bottom lock or no lock, in air, out of range).
- Typical uncertainty against altitude.

**Test plan:** record a `ros2 bag` for each case below and run the profiler on it.
1. In air.
2. In a tank, sitting still.
3. Pushed at a known speed.
4. At several heights above the bottom, down to the minimum range.

From these runs, infer what Z1, Z2 and the FLAGS bits mean. Ask Kogger directly for the beam angle.
