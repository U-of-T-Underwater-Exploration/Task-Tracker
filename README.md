# Task-Tracker

This repository tracks the development tasks for the `uuv-code` underwater vehicle stack. For each task it records the goal and the design of the ROS 2 nodes that implement it, in the same format as the existing node documentation.

- [nodes.md](nodes.md): the nodes already on the `dev` branch.
- [new_nodes/](new_nodes/README.md): the design for the tasks below, with conventions, the system diagram and open questions.

## Tasks

Listed in the suggested order of work. The full table, with dependencies and "done when" criteria, is in [new_nodes/README.md](new_nodes/README.md#suggested-order-of-work).

| # | Task | Goal | Nodes |
| --- | --- | --- | --- |
| 0 | [Merge feature branches](new_nodes/feature_branches/README.md) | Bring the sensor, BMS and state-estimator branches onto `dev` with agreed topic names | IMU, compass, barometer, BMS, state estimator ([fixes needed](new_nodes/integration_fixes.md)) |
| 1 | [DVL Reading](new_nodes/dvl_reading/README.md) | Read the Kogger Micro DVL and report what data it actually provides | [uuv_dvl_driver](new_nodes/dvl_reading/uuv_dvl_driver.md), [dvl_profiler](new_nodes/dvl_reading/dvl_profiler.md) |
| 2 | [Simulation](new_nodes/simulation/README.md) | Test the control stack in Stonefish before pool time | [stonefish_simulator](new_nodes/simulation/stonefish_simulator.md), [sim_bridge](new_nodes/simulation/sim_bridge.md) |
| 3 | [Rover Safety](new_nodes/rover_safety/README.md) | Detect sensor or state malfunctions and cut motor commands | [safety_monitor](new_nodes/rover_safety/safety_monitor.md) |
| 4 | [Using the DVL: State Estimation](new_nodes/state_estimation/README.md) | Fuse DVL velocity and depth into the state estimator; avoid the seabed | changes to [state_estimator](new_nodes/changes_to_existing/state_estimator.md) |
| 5 | [Adaptive Controller](new_nodes/adaptive_controller/README.md) | Slotine-Li controller with online gradient updates of the model parameters | [adaptive_controller](new_nodes/adaptive_controller/adaptive_controller.md) |
| 6 | [Sonar Mapping](new_nodes/sonar_mapping/README.md) | Build a map of the pool or lake from the Ping360 | [ping360_node](new_nodes/sonar_mapping/ping360_node.md), [sonar_mapper](new_nodes/sonar_mapping/sonar_mapper.md) |
| 7 | [Path Planning](new_nodes/path_planning/README.md) | Plan with A* on the map and follow the path | [global_planner](new_nodes/path_planning/global_planner.md), [path_follower](new_nodes/path_planning/path_follower.md) |

## Supporting documents

| Document | Contents |
| --- | --- |
| [Changes to existing nodes](new_nodes/changes_to_existing/README.md) | Edits to `joystick_hal`, `motion_converter_node`, `thruster_driver_node`, `state_estimator`, the sensor drivers and the URDF |
| [Integration fixes](new_nodes/integration_fixes.md) | What must be fixed before the feature branches merge into `dev` |
| [New messages](new_nodes/uuv_msgs.md) | The custom messages in `uuv_msgs` |
| [Open questions](new_nodes/README.md#open-questions) | Vendor, library and method questions still to answer |
