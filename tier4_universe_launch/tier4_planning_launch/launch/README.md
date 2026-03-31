# planning.launch.xml

This document describes the arguments of `planning.launch.xml` and how they affect vehicle behavior.

## Overview

`planning.launch.xml` is the top-level launch file for the Autoware planning stack. It launches the following subsystems:

- **Mission Planning** — route planning and manual lane change handling
- **Scenario Planning** — lane driving (behavior + motion planning) and parking
- **Planning Validator** — trajectory validation before publishing to control
- **Planning Evaluator** — metrics evaluation of planned trajectories
- **Remaining Distance/Time Calculator** (optional) — mission progress estimation

## Arguments

### Parameter File Paths

#### `mission_planner_param_path`

Path to the YAML parameter file for the **Mission Planner** node (`autoware_mission_planner_universe`).

This file controls:

- **Arrival detection** — how the planner determines the vehicle has reached its goal:
  - Angle alignment threshold
  - Lateral/longitudinal distance tolerances
  - Duration the vehicle must remain near the goal
- **Reroute behavior** — when and how the vehicle can request a new route:
  - Minimum time between reroute requests
  - Minimum route length for reroute
  - Whether rerouting is allowed in autonomous mode
- **Goal validation** — angle threshold for accepting a goal pose, footprint-inside-lanes check, goal pose correction

#### `freespace_planner_param_path`

Path to the YAML parameter file for the **Freespace Planner** node (`autoware_freespace_planner`), used in the parking scenario.

This file controls:

- **Planning algorithm selection** — A\* (default) or RRT\* for freespace path generation
- **Waypoint velocity** — speed at which the vehicle executes parking maneuvers
- **Replanning triggers**:
  - Whether to replan when an obstacle is found on the current path
  - Whether to replan when the vehicle deviates from the planned course
  - Debounce time for obstacle detection
- **Search parameters** — time limit, turning ratio, angle discretization, cost weights for curves/reverse/direction changes
- **Goal tolerances** — angle, lateral, and longitudinal tolerance for reaching the parking goal
- **Vehicle shape margin** — safety margin added to the vehicle footprint during planning

#### `planning_validator_param_path`

Path to the YAML parameter file for the **Planning Validator** node (`autoware_planning_validator`).

This file controls the overall validation behavior:

- **Handling strategy when validation fails**:
  - `PUBLISH_AS_IT_IS` (type 0) — publish the invalid trajectory with a warning
  - `USE_PREVIOUS_RESULT` (type 1) — publish the last valid trajectory instead
  - `USE_PREVIOUS_RESULT_WITH_SOFT_STOP` (type 2) — publish a decelerating trajectory to bring the vehicle to a stop
- **Diagnostic publishing** — whether to publish ROS 2 diagnostics and terminal warnings
- **Error count threshold** — number of consecutive failures before triggering an error

#### `planning_validator_latency_checker_param_path`

Path to the YAML parameter file for the **Latency Checker** plugin of the Planning Validator.

This file controls:

- **Trajectory freshness check** — threshold (in seconds) for how stale a trajectory timestamp can be before it is considered invalid. If the trajectory is older than this threshold, it triggers the configured handling strategy (e.g., use previous result or soft stop).

#### `planning_validator_trajectory_checker_param_path`

Path to the YAML parameter file for the **Trajectory Checker** plugin of the Planning Validator.

This file controls validation thresholds for trajectory quality. Each check can independently trigger the handling strategy:

| Check | Description |
|-------|-------------|
| Interval | Maximum distance between consecutive trajectory points |
| Curvature | Maximum allowed curvature (relates to steering capability) |
| Relative angle | Maximum yaw change between consecutive points |
| Lateral acceleration | Maximum lateral acceleration |
| Longitudinal acceleration | Maximum/minimum longitudinal acceleration (accel/decel limits) |
| Lateral jerk | Maximum lateral jerk |
| Steering / steering rate | Maximum steering angle and rate of change |
| Distance deviation | Maximum lateral distance from ego position |
| Longitudinal distance deviation | Maximum longitudinal distance from ego position |
| Velocity deviation | Maximum velocity difference from ego velocity |
| Yaw deviation | Maximum yaw difference from ego orientation |
| Forward trajectory length | Minimum trajectory length (based on deceleration profile) |
| Trajectory shift | Detects sudden lateral/longitudinal shifts in trajectory |

#### `planning_validator_intersection_collision_checker_param_path`

Path to the YAML parameter file for the **Intersection Collision Checker** plugin of the Planning Validator.

This file controls:

- **Intersection collision detection** — checks whether the ego trajectory will collide with obstacles at intersections during right/left turns
- **Detection range** — how far ahead to check for potential collisions
- **Time-to-collision threshold** — minimum safe time-to-collision before declaring invalid
- **Temporal filtering** — on/off time buffers to prevent flickering between valid/invalid states
- **Per-turn-direction enable** — can independently enable/disable checks for right and left turns

#### `planning_validator_rear_collision_checker_param_path`

Path to the YAML parameter file for the **Rear Collision Checker** plugin of the Planning Validator.

This file controls:

- **Rear collision detection** — checks for obstacles approaching from behind the vehicle using point cloud data with velocity estimation
- **Time-to-collision margin** — safety margin for rear collision time calculation
- **Ego reaction model** — assumed reaction time and maximum deceleration capability
- **Temporal filtering** — on/off time buffers for stable collision state transitions
- **Point cloud filtering** — voxel grid size, crop box dimensions, velocity estimation parameters

### Behavior Arguments

#### `enable_all_modules_auto_mode`

Controls whether behavior planning modules operate in **automatic mode** or require **external authorization** via the RTC (Runtime Cooperation) interface.

| Value | Effect |
|-------|--------|
| `true` | All planning modules (lane change, avoidance, goal planner, etc.) activate automatically based on their internal safety assessment. No external command is needed. |
| `false` | Each module publishes its decision status and waits for an explicit execution command from an external system (e.g., operator HMI) before activating. |

This affects both the **Behavior Path Planner** (path-level decisions like lane changes, avoidance, start/goal planning) and the **Behavior Velocity Planner** (velocity-level decisions at intersections, crosswalks, traffic lights, etc.).

#### `is_simulation`

Controls safety fallback behavior in the **traffic light module** of the Behavior Velocity Planner when no traffic signal recognition data is available.

| Value | Effect |
|-------|--------|
| `true` | The vehicle **passes through** traffic lights when no signal data is available. This prevents the vehicle from stopping indefinitely in simulation environments where traffic light perception may not be running. |
| `false` | The vehicle **stops** at traffic lights when no signal data is available. This is the safe default for real-world operation, protecting against perception failures or map errors. |

When traffic signal data is available, the vehicle uses the actual signal state regardless of this flag.

#### `launch_remaining_distance_time_calculator`

**Default:** `false`

Controls whether to launch the **Remaining Distance/Time Calculator** node.

| Value | Effect |
|-------|--------|
| `true` | Launches a node that continuously calculates and publishes the remaining distance (meters) and estimated remaining time (seconds) to the goal along the planned route. Used for HMI display and mission progress tracking. Only active during lane driving (not parking). |
| `false` | The node is not launched. No remaining distance/time information is published. |

### Input Topic Arguments

#### `input_objects_topic_name`

The ROS topic providing **detected dynamic objects** (vehicles, pedestrians, cyclists, etc.) from the perception stack. This topic is consumed by:

- **Behavior Path Planner** — for object-aware path decisions (lane change safety, obstacle avoidance, goal/start planning)
- **Behavior Velocity Planner** — for velocity decisions at crosswalks, intersections, blind spots, etc.
- **Motion Velocity Planner** — for velocity adjustments based on nearby dynamic objects
- **Surround Obstacle Checker** — for detecting obstacles around the stopped vehicle
- **Planning Validator** — for collision checking in intersection and rear collision checkers

#### `input_pointcloud_topic_name`

The ROS topic providing **ground-filtered point cloud** data from the perception stack. This topic is consumed by:

- **Behavior Velocity Planner** — for occlusion detection and detailed obstacle sensing
- **Motion Velocity Planner** — for obstacle detection in velocity planning
- **Surround Obstacle Checker** — for point-cloud-based obstacle detection around the vehicle
- **Planning Validator** — for point-cloud-based collision checking (intersection and rear collision)

## Subsystem Launch Structure

```
planning.launch.xml
├── mission_planning/
│   ├── manual_lane_change_handler
│   └── mission_planning.launch.xml
│       └── Mission Planner + Route Selector
├── scenario_planning/
│   ├── scenario_selector
│   ├── velocity_smoother
│   ├── external_velocity_limit_selector
│   ├── hazard_lights_selector
│   ├── lane_driving.launch.xml
│   │   ├── behavior_planning/
│   │   │   ├── Behavior Path Planner
│   │   │   └── Behavior Velocity Planner
│   │   └── motion_planning/
│   │       └── Motion Velocity Planner + Obstacle Stop/Slowdown
│   └── parking.launch.xml
│       ├── Costmap Generator
│       └── Freespace Planner
├── Planning Validator
├── Planning Evaluator
└── Remaining Distance/Time Calculator (optional)
```
