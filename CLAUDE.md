# CLAUDE.md — AI Assistant Guide for DOBOT_6Axis_ROS2_V4

This document provides context, conventions, and workflows for AI assistants working on this repository.

---

## Project Overview

This is a **ROS2 SDK** for controlling Dobot collaborative robotic arms (6-axis and similar) via TCP/IP. It provides:

- Real-time hardware communication via TCP/IP sockets
- Full MoveIt motion planning integration
- Gazebo simulation support
- RViz visualization
- Support for 11 distinct Dobot robot models

**Target environment:** Ubuntu 22.04 + ROS2 Humble

---

## Repository Structure

```
DOBOT_6Axis_ROS2_V4/
├── dobot_bringup_v4/        # Core hardware interface node (C++)
├── dobot_msgs_v4/           # Custom ROS2 messages and services
├── dobot_moveit/            # MoveIt integration (Python)
├── dobot_demo/              # Demo client application (Python)
├── dobot_gazebo/            # Gazebo simulation
├── dobot_rviz/              # RViz config, URDF files, meshes
├── cra_description/         # Shared robot description (URDF/meshes)
├── servo_action/            # Action server for servo control
├── cr3_moveit/              # MoveIt config for CR3
├── cr5_moveit/              # MoveIt config for CR5
├── cr7_moveit/              # MoveIt config for CR7
├── cr10_moveit/             # MoveIt config for CR10
├── cr12_moveit/             # MoveIt config for CR12
├── cr16_moveit/             # MoveIt config for CR16
├── cr20_moveit/             # MoveIt config for CR20
├── cr30h_moveit/            # MoveIt config for CR30H
├── me6_moveit/              # MoveIt config for ME6
├── nova2_moveit/            # MoveIt config for Nova2
├── nova5_moveit/            # MoveIt config for Nova5
├── V4新增指令/              # V4 protocol changelog and PDF docs
├── README.md                # Chinese user guide
└── README_EN.md             # English user guide
```

---

## Supported Robot Models

| Model  | Description                  |
|--------|------------------------------|
| CR3    | CR-series, lighter payload   |
| CR5    | CR-series, 5kg payload       |
| CR7    | CR-series, 7kg payload       |
| CR10   | CR-series, 10kg payload      |
| CR12   | CR-series, 12kg payload      |
| CR16   | CR-series, 16kg payload      |
| CR20   | CR-series, 20kg payload      |
| CR30H  | CR-series, heavy payload     |
| ME6    | ME-series                    |
| Nova2  | Nova-series, 2kg payload     |
| Nova5  | Nova-series, 5kg payload     |

Each model has its own `*_moveit/` package containing:
- `config/joint_limits.yaml` — per-joint velocity/acceleration limits
- `config/kinematics.yaml` — IK solver settings
- `config/moveit_controllers.yaml` — MoveIt controller definitions
- `config/ros2_controllers.yaml` — ROS2 controller configuration
- `config/pilz_cartesian_limits.yaml` — Cartesian motion safety limits
- `config/initial_positions.yaml` — Default joint start positions
- `launch/` — MoveIt launch files for real hardware and simulation

---

## Build System

**Build tool:** `colcon` + `ament_cmake` (standard ROS2)

**C++ standards:**
- `dobot_bringup_v4`: C++14
- `dobot_msgs_v4`: C++17

**Build commands:**
```bash
# From the workspace root (parent of this repo, or from repo root)
colcon build
source install/local_setup.sh
```

**Compiler flags used:** `-Wall -Wextra -Wpedantic`

**External C++ libraries:**
- `nlohmann/json.hpp` — JSON command parsing (header-only)
- `pthread` — multi-threaded robot communication

---

## Key Packages and Source Files

### `dobot_bringup_v4/` — Hardware Interface (C++)

The central package. All robot communication happens here.

| File | Purpose |
|------|---------|
| `src/main.cpp` | ROS2 node entry point; creates publishers for joint states and robot status |
| `src/cr_robot_ros2.cpp` | Core robot class; registers 100+ service servers; handles all command callbacks |
| `src/command.cpp` | Command string construction and dispatch |
| `src/parseTool.cpp` | Tool coordinate frame parsing |
| `src/tcp_socket.cpp` | TCP/IP socket wrapper for robot controller communication |
| `include/dobot_bringup/cr_robot_ros2.h` | Class declaration; exposes all service callbacks |
| `include/dobot_bringup/command.h` | Command interface |
| `include/dobot_bringup/parseTool.h` | Parse tool declarations |
| `include/dobot_bringup/tcp_socket.h` | Socket class declarations |

**Published topics:**
- `joint_states_robot` (`sensor_msgs/JointState`) — live joint angles from robot
- Robot status (`dobot_msgs_v4/msg/RobotStatus`)
- Tool position (`dobot_msgs_v4/msg/ToolVectorActual`)

**Default publish rate:** 10.0 Hz (configurable via `JointStatePublishRate` env var)

### `dobot_msgs_v4/` — Custom Messages and Services

Contains all custom ROS2 message and service definitions.

**Custom messages (`msg/`):**
- `RobotStatus.msg` — `bool is_enable`, `bool is_connected`
- `ToolVectorActual.msg` — `float64 x, y, z, rx, ry, rz` (Cartesian pose)

**Custom services (`srv/`):** 152+ services organized into categories:

| Category | Example Services |
|----------|-----------------|
| Robot control | `EnableRobot`, `DisableRobot`, `PowerOn`, `ResetRobot`, `EmergencyStop` |
| Joint motion | `MovJ`, `RelMovJUser`, `RelMovJTool`, `RelJointMovJ` |
| Cartesian motion | `MovL`, `RelMovLUser`, `RelMovLTool`, `Circle`, `Arc` |
| Velocity/acceleration | `VelJ`, `VelL`, `AccJ`, `AccL`, `CP` |
| Real-time servo | `ServoJ`, `ServoP` |
| Digital IO | `DO`, `DOInstant`, `DI`, `ToolDO`, `ToolDOInstant`, `DOGroup`, `DOGroupDEC` |
| Analog IO | `AO`, `AOInstant`, `AI`, `ToolAI` |
| Kinematics | `PositiveKin`, `InverseKin`, `InverseSolution`, `CalcUser`, `CalcTool` |
| State queries | `GetAngle`, `GetPose`, `GetError`, `GetErrorID`, `RobotMode` |
| Coordinate frames | `User`, `Tool`, `SetUser`, `SetTool` |
| Force/torque | `EnableFTSensor`, `GetForce`, `ForceDriveMode`, `FCForceMode` |
| Force feedback | `FCSetForce`, `FCSetStiffness`, `FCSetDamping`, `FCCollisionSwitch` |
| Safety | `SetCollisionLevel`, `SetSafeWallEnable`, `SetBackDistance` |
| Drag teaching | `StartDrag`, `StopDrag`, `DragSensivity` |
| Program control | `Stop`, `Pause`, `Continue`, `RunScript` |
| Modbus | `ModbusCreate`, `ModbusRTUCreate`, `ModbusClose`, `GetHoldRegs`, `SetHoldRegs` |
| Trajectory | `StartPath`, `MoveJog`, `StopMoveJog`, `RunTo` |
| RT offset | `StartRTOffset`, `EndRTOffset` |
| Conveyor | `CnvInit`, `CnvMovL`, `CnvMovC`, `StartSyncCnv`, `StopSyncCnv` |
| Variable access | `GetInputBool/Int/Float`, `SetOutputBool/Int/Float` |

**Service request/response pattern:**
```
# Request fields vary per service; example for MovJ.srv:
bool mode
float64 a
float64 b
float64 c
float64 d
float64 e
float64 f
string[] param_value
---
string robot_return
int32 res
```

### `dobot_moveit/` — MoveIt Integration (Python)

| File | Purpose |
|------|---------|
| `dobot_moveit/action_move_server.py` | Action server implementing `FollowJointTrajectory`; converts MoveIt trajectories to `ServoJ` service calls |
| `dobot_moveit/joint_states.py` | Joint state management utilities |
| `launch/moveit_demo.launch.py` | MoveIt planning demo (simulation/offline) |
| `launch/dobot_moveit.launch.py` | Real hardware MoveIt integration |
| `launch/moveit_gazebo.launch.py` | Gazebo + MoveIt integration |

### `dobot_demo/` — Demo Client (Python)

`dobot_demo/demo.py` demonstrates basic robot control via service calls:
- `EnableRobot` → `MovJ` → `MovL` → `DO` (digital output)

### `dobot_gazebo/` — Simulation

| File | Purpose |
|------|---------|
| `worlds/cr.world` | Gazebo world file |
| `launch/dobot_gazebo.launch.py` | Launch Gazebo alone |
| `launch/gazebo_moveit.launch.py` | Gazebo + MoveIt combined |

### `dobot_rviz/` — Visualization

- `urdf/` — URDF files for all 11 robot models (`cr3_robot.urdf`, etc.)
- `rviz/` — RViz configuration files
- `meshes/` — 3D mesh files for visualization
- `launch/dobot_rviz.launch.py` — Launch RViz

### `cra_description/`

Shared robot description resources (URDF/Xacro and meshes) used by MoveIt packages.

---

## Communication Architecture

```
ROS2 Application / MoveIt
         |
  ROS2 Service Calls (152+ services)
         |
 cr_robot_ros2_node (C++ ROS2 node)
         |
   tcp_socket.cpp
         |
  TCP/IP (port varies)
         |
 Dobot Robot Controller
```

The robot controller exposes a TCP/IP API documented in:
- `V4新增指令/Dobot TCP_IP二次开发接口文档_V4.6.5_20251015_cn.pdf`
- Protocol version: **V4.6.5** (as of 2025-10-15)

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `IP_address` | `192.168.5.1` | Robot controller IP (wired); use `192.168.1.6` for wireless |
| `DOBOT_TYPE` | — | Robot model: `cr3`, `cr5`, `cr7`, `cr10`, `cr12`, `cr16`, `cr20`, `cr30h`, `me6`, `nova2`, `nova5` |
| `JointStatePublishRate` | `10.0` | Joint state publish frequency (Hz) |

---

## Launch Workflows

### 1. RViz Visualization Only
```bash
ros2 launch dobot_rviz dobot_rviz.launch.py
```

### 2. Real Hardware Bringup
```bash
# Set robot IP and type first
export IP_address=192.168.5.1
export DOBOT_TYPE=cr5
ros2 launch dobot_bringup_v4 dobot_bringup_ros2.launch.py
```

### 3. MoveIt with Real Hardware
```bash
ros2 launch dobot_moveit dobot_moveit.launch.py
```

### 4. Gazebo Simulation
```bash
ros2 launch dobot_gazebo dobot_gazebo.launch.py
```

### 5. Gazebo + MoveIt
```bash
ros2 launch dobot_gazebo gazebo_moveit.launch.py
# or
ros2 launch dobot_moveit moveit_gazebo.launch.py
```

### 6. MoveIt Demo (no hardware required)
```bash
ros2 launch dobot_moveit moveit_demo.launch.py
```

---

## Testing

Tests are in the Python packages (`dobot_demo`, `dobot_moveit`, `servo_action`).

**Test types (standard ament Python tests):**
- `test_copyright.py` — checks copyright headers
- `test_flake8.py` — PEP8 style compliance
- `test_pep257.py` — docstring convention compliance

**Run tests:**
```bash
colcon test
colcon test-result --verbose
```

**Linting tools:** `ament_flake8`, `ament_pep257`, `ament_copyright`

---

## Code Conventions

### C++ (dobot_bringup_v4)

- Standard: C++14
- Compiler warnings: `-Wall -Wextra -Wpedantic` — fix all warnings
- Naming: class members use camelCase; service callbacks follow ROS2 service naming patterns
- The main robot class `CRRobotROS2` (in `cr_robot_ros2.h/cpp`) registers all service servers in its constructor
- TCP commands are assembled as formatted strings matching the Dobot TCP/IP protocol
- JSON parsing uses the `nlohmann/json` header-only library

### Python (dobot_moveit, dobot_demo, servo_action)

- Follow PEP8 (enforced by flake8)
- Follow PEP257 for docstrings
- Include copyright headers in all files (enforced by ament_copyright)
- ROS2 Python nodes use `rclpy`

### Service Definitions

- All services follow the pattern: request fields + `---` separator + `string robot_return` + `int32 res`
- `res == 0` typically indicates success
- `robot_return` contains the raw string response from the robot controller

### Adding a New Service

1. Add `NewService.srv` to `dobot_msgs_v4/srv/`
2. Register it in `dobot_msgs_v4/CMakeLists.txt` under `rosidl_generate_interfaces`
3. Add a callback method declaration to `cr_robot_ros2.h`
4. Implement the callback in `cr_robot_ros2.cpp`
5. Register the service server in the node constructor

### Adding a New Robot Model

1. Create `<model>_moveit/` package by copying an existing `*_moveit/` package
2. Update `package.xml` and `CMakeLists.txt` with the new model name
3. Update all YAML config files for the new robot's kinematics and limits
4. Add the corresponding URDF to `dobot_rviz/urdf/`
5. Add meshes to `dobot_rviz/meshes/` and `cra_description/meshes/`

---

## Key Architectural Decisions

1. **All robot commands are ROS2 services** — not actions or topics. This means commands are synchronous request/response by design. Only `FollowJointTrajectory` uses the ROS2 action interface (via `dobot_moveit`).

2. **TCP/IP passthrough** — The `cr_robot_ros2_node` is essentially a ROS2 service-to-TCP-command bridge. Service callbacks format command strings per the Dobot TCP/IP protocol and send them over a socket.

3. **Per-model MoveIt packages** — Each robot model has its own MoveIt configuration package rather than a unified parameterized package. This keeps configurations explicit and independent.

4. **Joint state publishing is separate from command execution** — The main node runs a continuous loop publishing joint states at `JointStatePublishRate` Hz independent of incoming service calls.

5. **ServoJ is the real-time control interface** — For trajectory following, MoveIt sends trajectories to the `action_move_server.py`, which splits them into `ServoJ` service calls executed sequentially.

---

## Protocol Reference

The Dobot TCP/IP protocol is documented in:
- `V4新增指令/Dobot TCP_IP二次开发接口文档_V4.6.5_20251015_cn.pdf`

Recent protocol additions (V4.6.5, 2025-10-15):
- `DOGroupDEC` / `DIGroupDEC` / `getDOGroupDEC` — decimal-encoded IO group commands
- Conveyor belt integration commands (`CnvInit`, `CnvMovL`, `CnvMovC`, etc.)
- Real-time offset commands (`StartRTOffset`, `EndRTOffset`)

---

## Maintainer

- **Organization:** Dobot Arm
- **Contact:** futingxing@dobot-robot.com
- **Protocol version:** V4.6.5 (2025-10-15)
- **Last significant update:** April 2025
