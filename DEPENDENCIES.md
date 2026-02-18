# DEPENDENCIES.md — Dependency Reference for DOBOT_6Axis_ROS2_V4

This document lists all dependencies for the project, organized by category, and provides
complete installation instructions.

---

## System Requirements

| Requirement | Value |
|-------------|-------|
| Operating System | Ubuntu 22.04 LTS |
| ROS2 Distribution | ROS2 Humble Hawksbill |
| CMake | ≥ 3.22 (MoveIt packages), ≥ 3.16 (dobot_msgs_v4), ≥ 3.5 (others) |
| C++ Standard | C++14 (dobot_bringup_v4), C++17 (dobot_msgs_v4) |
| Python | 3.10 (Ubuntu 22.04 system default) |
| Build tool | colcon |

---

## Quick Install (All Dependencies)

Run the following in order:

```bash
# 1. Install ROS2 Humble (if not already installed)
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
    -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
    http://packages.ros.org/ros2/ubuntu \
    $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
    | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt update
sudo apt install ros-humble-desktop

# 2. Install colcon build tool
sudo apt install python3-colcon-common-extensions

# 3. Install MoveIt
sudo apt install ros-humble-moveit

# 4. Install Gazebo + ROS2 bridge (optional, for simulation)
sudo apt install ros-humble-gazebo-*
echo "source /usr/share/gazebo/setup.bash" >> ~/.bashrc

# 5. Install remaining ROS2 dependencies via rosdep
sudo apt install python3-rosdep
sudo rosdep init      # skip if already done
rosdep update
cd ~/dobot_ws         # your colcon workspace root
rosdep install --from-paths src --ignore-src -r -y

# 6. Install Python tooling
sudo apt install python3-setuptools python3-pytest

# 7. Source ROS2
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Dependencies by Category

### 1. ROS2 Core Packages

These are part of the standard ROS2 Humble desktop install.

| Package | Used by | Role |
|---------|---------|------|
| `rclcpp` | dobot_bringup_v4, dobot_msgs_v4 | C++ ROS2 client library |
| `rclcpp_action` | dobot_bringup_v4 | C++ action client/server |
| `rclpy` | dobot_moveit, dobot_demo, servo_action | Python ROS2 client library |
| `std_msgs` | dobot_bringup_v4, dobot_msgs_v4 | Standard message types |
| `sensor_msgs` | dobot_bringup_v4, dobot_moveit, servo_action | JointState and sensor messages |
| `control_msgs` | dobot_moveit, servo_action | FollowJointTrajectory action |
| `trajectory_msgs` | dobot_moveit, servo_action | JointTrajectory message |
| `builtin_interfaces` | dobot_msgs_v4 | Time and duration types |
| `tf2_ros` | all *_moveit packages | Transform tree |
| `robot_state_publisher` | dobot_rviz, dobot_gazebo, *_moveit | Publishes URDF to /tf |
| `joint_state_publisher` | dobot_rviz, *_moveit | Publishes joint states from URDF |
| `joint_state_publisher_gui` | *_moveit | GUI sliders for joint states |
| `controller_manager` | *_moveit | ros2_control controller manager |
| `ros2launch` | dobot_rviz | Launch system |
| `xacro` | dobot_rviz, dobot_gazebo, *_moveit | URDF macro processor |
| `launch` | dobot_moveit, dobot_gazebo, dobot_rviz, *_moveit | Core ROS2 launch framework (`from launch import LaunchDescription`) |
| `launch_ros` | dobot_moveit, dobot_gazebo, dobot_rviz, *_moveit | ROS2-specific launch actions (`Node`, `ParameterValue`) |
| `ament_index_python` | dobot_moveit, dobot_gazebo, dobot_rviz, *_moveit | Package share directory discovery in launch files |

**Install:**
```bash
sudo apt install ros-humble-desktop
# ros-humble-desktop includes: rclcpp, rclpy, std_msgs, sensor_msgs,
# control_msgs, trajectory_msgs, tf2_ros, robot_state_publisher,
# joint_state_publisher, joint_state_publisher_gui, xacro, rviz2,
# launch, launch_ros, ament_index_python
```

---

### 2. ROS2 Message Generation

| Package | Used by | Role |
|---------|---------|------|
| `rosidl_default_generators` | dobot_msgs_v4 | Generates C++/Python from .srv/.msg files |
| `rosidl_default_runtime` | dobot_msgs_v4 | Runtime IDL support |

> **Note:** `dobot_bringup_v4/package.xml` also declares `<build_depend>message_generation</build_depend>`.
> This is a **ROS1 artifact** — in ROS2, message generation is handled by `rosidl_default_generators`
> (correctly used in `dobot_msgs_v4`). The `message_generation` dependency is harmless but vestigial
> and could be removed.

**Install:** Included with `ros-humble-desktop`.

---

### 3. MoveIt Packages

Required by all `*_moveit` packages (cr3, cr5, cr7, cr10, cr12, cr16, cr20, cr30h, me6, nova2, nova5).

| Package | Role |
|---------|------|
| `moveit_ros_move_group` | MoveGroup server (core planning interface) |
| `moveit_kinematics` | IK solver plugins (KDL, TRAC-IK) |
| `moveit_planners` | Motion planners (OMPL, Pilz) |
| `moveit_simple_controller_manager` | Sends trajectories to ros2_control |
| `moveit_configs_utils` | Python helpers for MoveIt launch files |
| `moveit_ros_visualization` | RViz MoveIt plugin |
| `moveit_ros_warehouse` | MongoDB-backed trajectory warehouse |
| `moveit_setup_assistant` | GUI for generating MoveIt configs |
| `warehouse_ros_mongo` | MongoDB backend for warehouse |

**Install:**
```bash
sudo apt install ros-humble-moveit
```

> `ros-humble-moveit` is a meta-package that installs all of the above.

---

### 4. Gazebo Packages

Required only by `dobot_gazebo` (optional; only needed for simulation).

| Package | Role |
|---------|------|
| `gazebo_plugins` | Gazebo sensor and actuator plugins |
| `gazebo_ros` | Gazebo–ROS2 bridge nodes |
| `gazebo_ros2_control` | ros2_control hardware interface for Gazebo |

**Install:**
```bash
sudo apt install ros-humble-gazebo-*
echo "source /usr/share/gazebo/setup.bash" >> ~/.bashrc
```

**Verify:**
```bash
ros2 launch gazebo_ros gazebo.launch.py
```

---

### 5. RViz and Visualization

Required by `dobot_rviz` and all `*_moveit` packages.

| Package | Role |
|---------|------|
| `rviz2` | 3D visualization application |
| `rviz_common` | RViz plugin base classes |
| `rviz_default_plugins` | Standard RViz displays |

**Install:** Included with `ros-humble-desktop`.

---

### 6. Build and Linting Tools

| Package | Type | Role |
|---------|------|------|
| `ament_cmake` | buildtool | CMake wrapper for ROS2 packages |
| `ament_python` | buildtool | Python package support in ROS2 |
| `ament_lint_auto` | test | Auto-discovers linting tests |
| `ament_lint_common` | test | Common linting checks |
| `ament_copyright` | test | Enforces copyright headers in Python files |
| `ament_flake8` | test | PEP8 style checker via flake8 |
| `ament_pep257` | test | PEP257 docstring checker |
| `python3-pytest` | test | Python test runner |
| `python3-setuptools` | build | Python package installation |

**Install:**
```bash
sudo apt install python3-colcon-common-extensions \
                 python3-setuptools \
                 python3-pytest \
                 ros-humble-ament-lint-auto \
                 ros-humble-ament-lint-common
```

---

### 7. System Libraries

| Library | Used by | Install |
|---------|---------|---------|
| `libpthread` | dobot_bringup_v4 | Pre-installed on Ubuntu (`libc6`) |
| POSIX sockets (`sys/socket.h`, `arpa/inet.h`) | dobot_bringup_v4 | Pre-installed on Ubuntu (`libc6`) |

No additional installation needed — these are part of the standard Ubuntu `libc` package.

---

### 8. Embedded C++ Libraries (Vendored)

These are bundled directly in the source tree and require no separate installation.

| Library | Location | Version | Purpose |
|---------|----------|---------|---------|
| `nlohmann/json` | `dobot_bringup_v4/include/nlohmann/json.hpp` | single-header | JSON command string parsing |

No installation needed — already present in the repository.

---

### 9. Python Runtime Dependencies

Used implicitly by Python nodes. These are resolved through ROS2 workspace packages or the Ubuntu
system Python installation.

> **Warning:** These dependencies are currently **not declared** in the `package.xml` files for
> `dobot_moveit`, `dobot_demo`, and `servo_action`. They should be added as `<exec_depend>` entries.
> See [Missing `package.xml` Declarations](#missing-packagexml-declarations) below for details.

| Package | Used by | Install |
|---------|---------|---------|
| `numpy` | dobot_moveit, servo_action | `sudo apt install python3-numpy` |
| `rclpy` | dobot_moveit, dobot_demo, servo_action | Included with `ros-humble-desktop` |

---

## Per-Package Dependency Summary

| Package | Build deps | Exec deps | Test deps |
|---------|-----------|-----------|-----------|
| `dobot_bringup_v4` | ament_cmake, rclcpp, rclcpp_action, std_msgs, sensor_msgs, dobot_msgs_v4 | rclcpp, rclcpp_action, dobot_msgs_v4, pthread | ament_lint_auto, ament_lint_common |
| `dobot_msgs_v4` | ament_cmake, rosidl_default_generators, rclcpp, std_msgs, builtin_interfaces | rosidl_default_runtime, rclcpp | ament_lint_auto, ament_lint_common |
| `dobot_moveit` | ament_python, setuptools | rclpy, control_msgs, trajectory_msgs, sensor_msgs, dobot_msgs_v4, numpy | ament_copyright, ament_flake8, ament_pep257, python3-pytest |
| `dobot_demo` | ament_python, setuptools | rclpy, dobot_msgs_v4 | ament_copyright, ament_flake8, ament_pep257, python3-pytest |
| `servo_action` | ament_python, setuptools | rclpy, control_msgs, trajectory_msgs, sensor_msgs, numpy | ament_copyright, ament_flake8, ament_pep257, python3-pytest |
| `dobot_gazebo` | ament_cmake | gazebo_plugins, gazebo_ros, gazebo_ros2_control, robot_state_publisher, xacro, cra_description | — |
| `dobot_rviz` | ament_cmake | rviz2, xacro, robot_state_publisher, joint_state_publisher, ros2launch | ament_lint_auto, ament_lint_common |
| `cra_description` | ament_cmake | — | — |
| `cr*_moveit`, `me6_moveit`, `nova*_moveit` | ament_cmake | moveit_ros_move_group, moveit_kinematics, moveit_planners, moveit_simple_controller_manager, moveit_configs_utils, moveit_ros_visualization, moveit_ros_warehouse, moveit_setup_assistant, warehouse_ros_mongo, robot_state_publisher, joint_state_publisher, joint_state_publisher_gui, tf2_ros, xacro, controller_manager, rviz2, rviz_common, rviz_default_plugins, dobot_rviz | — |

---

## Missing `package.xml` Declarations

The following dependencies are used in source code and launch files but are **not declared** in
their respective `package.xml` files. This means `rosdep install` will not automatically resolve
them. They are documented here for completeness; the fixes below should be applied to the
corresponding `package.xml` files.

### `dobot_bringup_v4` (package name: `cr_robot_ros2`)

`sensor_msgs` is used in `CMakeLists.txt` (`find_package(sensor_msgs REQUIRED)`) and in source
files (`main.cpp`, `cr_robot_ros2.cpp`, `cr_robot_ros2.h`) but is **not declared** in
`package.xml`.

**Missing entry:**
```xml
<depend>sensor_msgs</depend>
```

Additionally, `message_generation` is declared as a `<build_depend>` but is a ROS1 package with
no effect in ROS2. It can safely be removed.

### `dobot_moveit`

All runtime dependencies are missing from `package.xml`. The package imports `rclpy`,
`control_msgs`, `trajectory_msgs`, `sensor_msgs`, `dobot_msgs_v4`, and `numpy` at runtime.

**Missing entries:**
```xml
<exec_depend>rclpy</exec_depend>
<exec_depend>control_msgs</exec_depend>
<exec_depend>trajectory_msgs</exec_depend>
<exec_depend>sensor_msgs</exec_depend>
<exec_depend>dobot_msgs_v4</exec_depend>
```

### `dobot_demo`

All runtime dependencies are missing from `package.xml`. The package imports `rclpy` and
`dobot_msgs_v4` at runtime.

**Missing entries:**
```xml
<exec_depend>rclpy</exec_depend>
<exec_depend>dobot_msgs_v4</exec_depend>
```

### `servo_action`

All runtime dependencies are missing from `package.xml`. The package imports `rclpy`,
`control_msgs`, `trajectory_msgs`, and `sensor_msgs` at runtime.

**Missing entries:**
```xml
<exec_depend>rclpy</exec_depend>
<exec_depend>control_msgs</exec_depend>
<exec_depend>trajectory_msgs</exec_depend>
<exec_depend>sensor_msgs</exec_depend>
```

### Impact

Without these declarations:
- `rosdep install --from-paths src` will not install missing dependencies for these packages
- `colcon build` may still succeed if the dependencies happen to be installed for other reasons
  (e.g., via `ros-humble-desktop`), masking the issue
- Deploying to a minimal ROS2 installation (e.g., `ros-humble-ros-base`) will fail

---

## Building the Workspace

```bash
# Create and enter workspace
mkdir -p ~/dobot_ws/src
cd ~/dobot_ws/src
git clone https://github.com/Dobot-Arm/DOBOT_6Axis_ROS2_V4.git

# Source ROS2
source /opt/ros/humble/setup.bash

# Install all rosdep-managed dependencies automatically
cd ~/dobot_ws
rosdep install --from-paths src --ignore-src -r -y

# Build
colcon build

# Source the workspace
source install/local_setup.sh
```

### Persistent Environment Setup

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
echo "source ~/dobot_ws/install/local_setup.sh" >> ~/.bashrc
echo "source /usr/share/gazebo/setup.bash" >> ~/.bashrc   # only if using Gazebo

# Robot connection settings
echo "export IP_address=192.168.5.1" >> ~/.bashrc          # wired; use 192.168.1.6 for wireless
echo "export DOBOT_TYPE=cr5" >> ~/.bashrc                  # set to your robot model

source ~/.bashrc
```

### Supported `DOBOT_TYPE` Values

`cr3` | `cr5` | `cr7` | `cr10` | `cr12` | `cr16` | `cr20` | `cr30h` | `me6` | `nova2` | `nova5`

---

## Running Tests

```bash
colcon test
colcon test-result --verbose
```

Tests cover:
- **Copyright headers** — all Python files must have a license header
- **PEP8 compliance** — enforced by flake8
- **PEP257 docstrings** — enforced by pep257

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| `Package 'dobot_msgs_v4' not found` | Workspace not sourced | `source install/local_setup.sh` |
| `Cannot connect to robot` | Wrong IP or not on same subnet | `ping $IP_address`; check network settings |
| `gazebo: command not found` | Gazebo not installed | `sudo apt install ros-humble-gazebo-*` |
| `moveit_ros_move_group not found` | MoveIt not installed | `sudo apt install ros-humble-moveit` |
| `ModuleNotFoundError: numpy` | numpy missing | `sudo apt install python3-numpy` |
| `colcon: command not found` | colcon not installed | `sudo apt install python3-colcon-common-extensions` |
