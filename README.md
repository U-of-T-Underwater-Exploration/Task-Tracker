# Task-Tracker



### Simulation
Goal: test control algorithm
Very simple -> just port CAD model and ROS2 inside. 
Stonefish simulator

### Path planning
A*
Potential field navigation
Map format?

### DVL Reading
Goal: read from DVL and test what data is available
- Give the data profile
- Generate a report on what is available for the DVL

link: Kogger specs and Protocol page
https://kogger.tech/product/micro-dvl/
https://github.com/koggertech/Kogger-Protocol

### Using DVL
- Depth detection - avoid hitting seabed
- Odeometry information help with EKF

What the protocol shows. The PDF has no beam-angle or beamwidth field, but it defines ID_DVL_VEL (0x79), version 2, a 68-byte message with these fields:
Velocity: VELOCITY_X, Y, Z, plus Z1 and Z2, all in m/s.
Uncertainty: one value per velocity field, so each velocity comes with its own error estimate.
Distance: DISTANCE_Z, DISTANCE_Z1, DISTANCE_Z2, in meters.
Timing and status: a flags word (bit meanings aren't documented), a timestamp, a delta time, and a latency. There are three beams.

### Sonar Mapping
*Consult Prof. Busheer* for what he uses to do lake mapping.
https://bluerobotics.com/store/sonars/imaging-sonars/ping360-sonar-r1-rp/
underwater SLAM
https://www.mdpi.com/2072-4292/15/10/2496

### Rover safety
Detect sensor or state malfunction and cut out motor commands.

### Adaptive controller
Slotine-Li controller
Implement gradient updates to the system parameters.