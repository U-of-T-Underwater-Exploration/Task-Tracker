# URDF (`uuv.urdf.xacro`, package `uuv_description`): CHANGED

**Lives on:** `dev`. Not a node; it is published by `robot_state_publisher` (see [nodes.md](../../nodes.md#robot-state-publisher-robot_state_publisher-package-uuv_description-launch-only)).

| Change | Why |
| --- | --- |
| **New** link `dvl_link`, fixed to `base_link` at the measured DVL mounting offset and orientation | `/dvl/twist` and `/dvl/altitude` use it; `state_estimator` needs R_dvl→base |
| **New** link `sonar_link`, fixed to `base_link` at the measured Ping360 mounting offset | `/sonar/*` use it; `sonar_mapper` transforms returns through it |

Use the same offsets in the Stonefish scenario ([stonefish_simulator.md](../simulation/stonefish_simulator.md)).
