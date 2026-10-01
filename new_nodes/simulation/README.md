# 2. Simulation

**Goal:** test the control algorithm before pool time. Keep the setup minimal: import the CAD model and run the existing ROS 2 stack inside [Stonefish](https://github.com/patrykcieslak/stonefish_ros2).

| Node | Status | File |
| --- | --- | --- |
| `stonefish_simulator` | third-party | [stonefish_simulator.md](stonefish_simulator.md) |
| `sim_bridge` | new | [sim_bridge.md](sim_bridge.md) |

With `use_sim:=true`, these two replace every hardware node: the sensor drivers, `ping360_node`, `bms_node`, `thruster_driver_node` and `pwm_driver`. Everything from `state_estimator` downward runs unchanged. `safety_monitor` runs with the sim check list, because there is no simulated battery or PWM hardware.

Until [Rover Safety](../rover_safety/) lands, `motion_converter_node` still publishes `/thruster/command` directly, so `sim_bridge` works without `safety_monitor`.

**Scoring:** plot `/control/reference` against `/sim/ground_truth` to score the controller, and `/state_estimate` against `/sim/ground_truth` to score the state estimator.
