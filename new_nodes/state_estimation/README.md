# 4. Using the DVL: State Estimation

**Goal:** fuse DVL odometry into the state estimator, and use altitude to avoid hitting the seabed.

**Approach:** extend the existing `uuv_state_estimator` on branch `feature/control-state_estimator`. Do not add a third-party EKF. There are no new nodes in this task.

| Part | Where it is specified |
| --- | --- |
| Current estimator | [feature_branches/state_estimator.md](../feature_branches/state_estimator.md) |
| Barometer and DVL fusion, TF, ENU output | [changes_to_existing/state_estimator.md](../changes_to_existing/state_estimator.md) |
| DVL input | [dvl_reading/uuv_dvl_driver.md](../dvl_reading/uuv_dvl_driver.md) (`/dvl/twist`) |
| Seabed avoidance | **Seabed** check in [rover_safety/safety_monitor.md](../rover_safety/safety_monitor.md) (`/dvl/altitude`) |

**Done when:** `/state_estimate` follows `/sim/ground_truth` in simulation, and depth matches the barometer.
