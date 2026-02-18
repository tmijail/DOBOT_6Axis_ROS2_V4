# ISSUES.md — Bug Analysis for DOBOT_6Axis_ROS2_V4

Automated codebase analysis performed on 2026-02-18.

---

## Summary

| Severity | Count |
|----------|-------|
| Critical | 2     |
| High     | 5     |
| Medium   | 7     |
| Low      | 6     |
| **Total**| **20**|

---

## Critical

### 1. File descriptor leak in `TcpClient::disConnect()` — socket never actually closed

**File:** `dobot_bringup_v4/src/tcp_socket.cpp:45-53`
**Category:** Resource Leak / Memory Safety

```cpp
void TcpClient::disConnect()
{
    if (is_connected_)
    {
        fd_ = -1;          // fd_ is set to -1 FIRST
        is_connected_ = false;
        ::close(fd_);       // then ::close(-1) is called — closes nothing!
    }
}
```

`fd_` is set to `-1` before `::close(fd_)` is called. This means `::close(-1)` is invoked (which fails silently with `EBADF`), and the real file descriptor is **never closed**. Every disconnection leaks a socket file descriptor. Over time, this exhausts the process's file descriptor limit, causing all subsequent socket operations to fail.

**Fix:** Swap the order — call `::close(fd_)` before setting `fd_ = -1`.

---

### 2. `sleep(0.01)` truncates to `sleep(0)` — creates a busy-wait loop

**File:** `dobot_bringup_v4/src/command.cpp:133` (also line 183)
**Category:** Performance / CPU Starvation

```cpp
while (true)
{
    bool err = tcp->tcpRecv(recv_ptr, 1024, has_read, 0);
    if (!err)
    {
        sleep(0.01);   // sleep() takes unsigned int; 0.01 truncates to 0
        continue;
    }
    ...
}
```

The POSIX `sleep()` function accepts `unsigned int`. The value `0.01` is implicitly cast to `0`, so `sleep(0)` returns immediately. Combined with `tcpRecv` having a timeout of `0` (passed as last argument), this loop becomes a **100% CPU busy-wait** whenever the robot doesn't respond immediately. This occurs in both `doTcpCmd()` and `doTcpCmd_f()`.

**Fix:** Use `usleep(10000)` (10ms) or `std::this_thread::sleep_for(std::chrono::milliseconds(10))`.

---

## High

### 3. Missing `kRobotName` prefix for `AccJ` service — service unreachable in multi-robot mode

**File:** `dobot_bringup_v4/src/cr_robot_ros2.cpp:50`
**Category:** Typo / Service Registration

```cpp
std::string serviceAccJ = +"/dobot_bringup_ros2/srv/AccJ";
//                         ^ missing kRobotName, unary + on string literal
```

All other service names use `kRobotName + "/dobot_bringup_ros2/srv/..."`, but `serviceAccJ` uses `+"/dobot_bringup_ros2/srv/AccJ"` (a unary `+` on a `const char*`, which is a no-op). In multi-robot mode (`robotNumber > 1`), this service will have a different namespace than all other services, making `AccJ` unreachable via the expected naming convention.

---

### 4. Typo in `StartPath` service name — extra trailing 't'

**File:** `dobot_bringup_v4/src/cr_robot_ros2.cpp:64`
**Category:** Typo / Service Registration

```cpp
std::string serviceStartPath = kRobotName + "/dobot_bringup_ros2/srv/StartPatht";
//                                                                        ^^^^^^^ "StartPatht" — extra 't'
```

Clients calling the `StartPath` service using the expected name `/dobot_bringup_ros2/srv/StartPath` will fail because the server registers with the misspelled name `StartPatht`.

---

### 5. `DOGroup` service name casing mismatch — "DoGroup" vs "DOGroup"

**File:** `dobot_bringup_v4/src/cr_robot_ros2.cpp:87`
**Category:** Typo / Service Registration

```cpp
std::string serviceDOGroup = kRobotName + "/dobot_bringup_ros2/srv/DoGroup";
//                                                                  ^^^^^^^ lowercase 'o'
```

The service type is `DOGroup` (all caps "DO"), but the registered topic name uses `DoGroup`. ROS2 service names are case-sensitive. Clients using the expected `DOGroup` naming convention will fail to find this service.

---

### 6. `Continue` service never registered — missing service member and registration

**File:** `dobot_bringup_v4/include/dobot_bringup/cr_robot_ros2.h` and `dobot_bringup_v4/src/cr_robot_ros2.cpp`
**Category:** Missing Feature / Dead Code

The `Continue` service callback is declared (header line 187) and implemented (cpp line 770-773), but:
- There is no `kServiceContinue` member variable in the header's `private:` section (line 309 is `kServicePause`, line 310 jumps to `kServiceEnableSafeSkin`)
- The service is never created via `create_service<>()` in `init()` (line 184 creates `kServicePause`, line 185 jumps to `kServiceEnableSafeSkin`)

**Impact:** The `Continue()` ROS2 service endpoint is **never available**. Users cannot resume paused robot programs via the ROS2 interface.

---

### 7. `RelMovJTool` service never registered — service member allocated but unused

**File:** `dobot_bringup_v4/src/cr_robot_ros2.cpp:244-248`
**Category:** Missing Registration

In the `init()` method, after `kServiceStopMoveJog` (line 244), the next registered service is `kServiceRelMovLTool` (line 245). The `kServiceRelMovJTool` member (declared at header line 370) is **never assigned**. The service endpoint for `RelMovJTool` is never created.

**Impact:** `RelMovJTool` (relative move in joint space with tool frame) is unavailable to users despite being fully implemented.

---

## Medium

### 8. Buffer over-read in `doTcpCmd()` — reads up to 2000 bytes past a 1024-byte buffer

**File:** `dobot_bringup_v4/src/command.cpp:141-152`
**Category:** Memory Safety / Undefined Behavior

```cpp
char buf[1024];
// ... recv_ptr points within buf ...
for (int i = 0; i < 2000; i++)  // iterates up to 2000
{
    if (recv_ptr[i] == '{')  // recv_ptr[i] can be past buf[1023]
    ...
}
```

The loop scans up to index 2000 from `recv_ptr`, which points into a 1024-byte stack buffer. This reads uninitialized stack memory and can cause crashes or undefined behavior. The same issue exists in `doTcpCmd_f()` at line 192.

---

### 9. `getErrorID` client uses wrong service name — "GeterrorID" vs "GetErrorID"

**File:** `dobot_bringup_v4/src/cr_robot_ros2.cpp:573`
**Category:** Typo / Service Client

```cpp
std::string name = kRobotName + "/dobot_bringup_ros2/srv/GeterrorID";
//                                                        ^^^^^^^^^^ lowercase 'e'
```

The service server registers as `GetErrorID` (line 81), but the client looks for `GeterrorID`. The client will never connect to the service, so `backendTask()` error reporting is silently broken.

---

### 10. Error messages print literal "%s" instead of actual error details

**File:** `dobot_bringup_v4/src/command.cpp:65,76,90,106,230,245`
**Category:** Logging / Debugging

```cpp
catch (const TcpClientException &err)
{
    std::cout << "tcp recv error :" << std::endl;   // line 65: err.what() not printed
    // ...
    std::cout << "tcp recv Error : %s" << std::endl; // line 76: literal "%s"
    // ...
    std::cout << "%s" << std::endl;                  // line 230: literal "%s"
}
```

Multiple `catch` blocks capture `TcpClientException` but never output `err.what()`. Some print literal `"%s"` as if using printf-style formatting, but `std::cout` doesn't interpret format specifiers. All error context is lost, making debugging connection issues very difficult.

---

### 11. Detached feedback thread has no shutdown mechanism

**File:** `dobot_bringup_v4/src/cr_robot_ros2.cpp:284-285`
**Category:** Thread Safety / Lifecycle

```cpp
threadPubFeedBackInfo = std::thread(&CRRobotRos2::pubFeedBackInfo, this);
threadPubFeedBackInfo.detach();
```

The feedback publishing thread is detached and relies only on `rclcpp::ok()` to exit its loop. When the node is destroyed, `this` may become invalid while the detached thread is still running, causing use-after-free. The thread should be joinable and joined in the destructor.

---

### 12. `PositiveKin` command generates trailing comma — malformed TCP command

**File:** `dobot_bringup_v4/src/parseTool.cpp:213-214`
**Category:** Protocol / Command Formatting

```cpp
ss << request->j6 << ",";    // always appends comma after j6
if (request->user != "")
    ss << ",user=" << request->user;  // adds ANOTHER comma before user
```

When `user` and `tool` are both empty, the command becomes `PositiveKin(j1,j2,j3,j4,j5,j6,)` — note the trailing comma before the closing parenthesis. When `user` is specified, it becomes `PositiveKin(j1,j2,j3,j4,j5,j6,,user=...)` with a double comma. Both are likely rejected by the robot controller.

---

### 13. `GetAO` command has leading space — may be rejected by robot controller

**File:** `dobot_bringup_v4/src/parseTool.cpp:414`
**Category:** Protocol / Command Formatting

```cpp
ss << " GetAO(" << request->index << ")";
//     ^ leading space
```

The command string starts with a space: `" GetAO(1)"` instead of `"GetAO(1)"`. The Dobot TCP/IP protocol may not tolerate leading whitespace.

---

### 14. Shared `RealTimeData` accessed from multiple threads without full synchronization

**File:** `dobot_bringup_v4/src/command.cpp:43-44` and `cr_robot_ros2.cpp:288-553`
**Category:** Thread Safety / Data Race

`real_time_data_` is a `shared_ptr<RealTimeData>` that is:
- Written to directly via `tcpRecv` into its raw memory (`reinterpret_cast<uint8_t*>(real_time_data_.get())`) in `recvTask()` (command.cpp:43)
- Read from `pubFeedBackInfo()` which publishes all its fields as JSON (cr_robot_ros2.cpp:297-546)
- Only `current_joint_` and `tool_vector_` are protected by `mutex_`

The `pubFeedBackInfo()` thread reads dozens of fields from `real_time_data_` without any lock, while `recvTask()` overwrites the entire struct via raw memory copy. This is a data race (undefined behavior) and can produce torn/inconsistent readings.

---

## Low

### 15. `demo.py` calls non-existent method `spin_until_future_complete` on Node

**File:** `dobot_demo/dobot_demo/demo.py:23,28,42,54,64`
**Category:** Python API Misuse

```python
node.spin_until_future_complete(response)
```

`rclpy.node.Node` does not have a `spin_until_future_complete` method. The correct call is:
```python
rclpy.spin_until_future_complete(node, response)
```

This will raise `AttributeError` at runtime, making the demo completely non-functional.

---

### 16. Truncated `PI` constant reduces kinematic precision

**File:** `dobot_bringup_v4/include/dobot_bringup/command.h:125`
**Category:** Precision

```cpp
static constexpr double PI = 3.1415926;
```

Uses only 7 decimal places. The error (~5.4e-8 radians) accumulates over trajectory segments. Should use `3.14159265358979323846` or `M_PI`.

Similarly, `action_move_server.py:47` uses `3.14159` (5 decimal places) for radian-to-degree conversion.

---

### 17. Launch files crash with `None` when environment variables are unset

**File:** `dobot_bringup_v4/launch/dobot_bringup_ros2.launch.py:27-29`
**Category:** Error Handling

```python
ip_address = os.getenv("IP_address")   # returns None if unset
robot_type = os.getenv("DOBOT_TYPE")   # returns None if unset
```

These values are passed directly as ROS2 parameters. If the environment variables are not set, `None` is passed, which may cause confusing errors downstream instead of a clear message at launch time.

The same issue exists in `dobot_moveit/launch/dobot_moveit.launch.py:7` and `dobot_moveit/dobot_moveit/action_move_server.py:21`.

---

### 18. `dobot_bringup_v4` CMakeLists.txt project name mismatch with directory

**File:** `dobot_bringup_v4/CMakeLists.txt:2`
**Category:** Build System / Naming

```cmake
project(cr_robot_ros2)
```

The package directory is `dobot_bringup_v4/` but the CMake project is named `cr_robot_ros2`. The executable installs to `lib/cr_robot_ros2/` and launch files reference `package='cr_robot_ros2'`. While this works, the naming inconsistency creates confusion for developers.

---

### 19. Hardcoded include path to install directory in CMakeLists.txt

**File:** `dobot_bringup_v4/CMakeLists.txt:29`
**Category:** Build System / Portability

```cmake
include_directories(
    include
    ../../../install/dobot_msgs_v4/include   # hardcoded relative path
)
```

This hardcodes a relative path to the colcon install directory. This breaks if the workspace layout changes, if using merged install, or if building in isolation. The `ament_target_dependencies` on line 39 should handle this automatically.

---

### 20. `action_move_server.py` fire-and-forget ServoJ calls — no error checking

**File:** `dobot_moveit/dobot_moveit/action_move_server.py:54-66`
**Category:** Robustness

```python
for ii in Positions:
    self.ServoJ_C(ii[0],ii[1],ii[2],ii[3],ii[4],ii[5])
    time.sleep(0.18)  # hardcoded delay
```

The `ServoJ_C` method calls `call_async()` but never checks the result (the `spin_until_future_complete` is commented out). If a ServoJ command fails, the trajectory continues blindly. Combined with the hardcoded 0.18s sleep (rather than using the trajectory's `time_from_start`), this can cause the robot to fall behind or skip waypoints.

---

## Notes

- `InverseSolution.srv` and `TcpDashboard.srv` are registered in `dobot_msgs_v4/CMakeLists.txt` but have no corresponding service servers in the C++ code. They may be intended for future use.
- The `dobot_bringup_v4/config/param.json` is read at launch file parse time (not at runtime), so changes require re-sourcing the workspace.
