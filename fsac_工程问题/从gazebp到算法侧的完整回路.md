## 一、完整闭环总图

```
                         ┌──────── Gazebo 物理世界 ────────┐
                         │                                 │
                         │  Gazebo 积分车辆位置、姿态       │
                         │  x / y / yaw / world velocity   │
                         └──────────────┬──────────────────┘
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 │                                             │
                 │ A. 控制算法测量反馈                          │ B. CAN/安全反馈
                 ▼                                             ▼
       Gazebo OdometryPublisher                    vehicle_dynamics PostUpdate
       /racecar/odometry (GZ)                      /vehicle_dynamics/telemetry
                 │                                             │
                 ▼                                             ├──────────────┐
       ros_gz_bridge                                         虚拟 ECU       │
                 │                                     sim_can_ecu_bridge    │
                 ▼                                             │             │
          ROS /odom                                    CAN 0x100/101/102     │
                 │                                             │             │
        ┌────────┴────────┐                                    ▼             ▼
        ▼                 ▼                                  vcan0    motion_authority
   fsac_planner      control_node                               │      遥测是否新鲜
        │          Pure Pursuit + PID                           ▼             │
        │                 │                          FsacSocketCanSystem       │
        └─centerline──────┘                          ros2_control read()       │
                          │                                    │             │
                          ▼                                    ▼             │
                  /control/request                    state interfaces       │
                          │                                    │             │
                          └──────────────┬─────────────────────┴─────────────┘
                                         ▼
                              command_gate_controller
                       检查请求、授权、VCU模式、CAN健康
                                         │
                              ros2_control command interfaces
                                         │
                                         ▼
                           FsacSocketCanSystem::write()
                                         │
                                   CAN 0x200 命令
                                         │
                                         ▼
                                  sim_can_ecu_bridge
                                         │
                            /vehicle_dynamics/command
                                         │
                                         ▼
                              动力学模型 → Gazebo
                                         │
                                         └────进入下一循环
```

# 二、主控制反馈：Gazebo → `/odom` → control_node

这才是 PID 和 Pure Pursuit 真正使用的车辆运动反馈。

## 1. Gazebo 更新车辆位置

动力学插件把车体速度和横摆角速度写给 Gazebo：

[gazebo_vehicle_dynamics_system.cpp (line 437)](/home/amoy/fsac_driveless/driverless_workspace/workspace/amoyfs_simulation/src/vehicle_dynamics/src/vehicle_dynamics_gz_system/gazebo_vehicle_dynamics_system.cpp:437)

```
canonical_link_.SetLinearVelocity(...);
canonical_link_.SetAngularVelocity(...);
```

之后由 Gazebo Physics 根据：

```
线速度
角速度
仿真时间步
碰撞和约束
```

推进车辆世界位置与姿态。

在 `PostUpdate()` 中，动力学插件又从 Gazebo 读取实际位置：

[gazebo_vehicle_dynamics_system.cpp (line 417)](/home/amoy/fsac_driveless/driverless_workspace/workspace/amoyfs_simulation/src/vehicle_dynamics/src/vehicle_dynamics_gz_system/gazebo_vehicle_dynamics_system.cpp:417)

```
const auto pose = canonical_link_.WorldPose(ecm);
```

然后同步：

```
state_.x_m
state_.y_m
state_.yaw_rad
```

因此当前所有权是：

```
速度：自研动力学计算
位姿：Gazebo 积分
动力学内部位姿：下一周期从 Gazebo 读回
```
## Gazebo 官方插件发布里程计

车模还加载了 Gazebo 官方 `OdometryPublisher`：

[racecar.vehicle_dynamics.xacro (line 104)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/racecar_description/urdf/racecar.vehicle_dynamics.xacro:104)

配置为：

```
<odom_frame>odom</odom_frame>
<robot_base_frame>base_footprint</robot_base_frame>
<odom_topic>/racecar/odometry</odom_topic>
<tf_topic>/racecar/tf</tf_topic>
<odom_publish_frequency>50</odom_publish_frequency>
```

它直接读取 Gazebo 中车辆的实际状态，发布 Gazebo Transport 消息：

```
/racecar/odometry
/racecar/tf
```

也就是说，`/odom` 不是自研动力学插件伪造出来的，而是 Gazebo 根据实际世界状态生成的。

## 3. ros_gz_bridge 转换为 ROS 消息

Simulation bringup 启动 `ros_gz_bridge`：

[simulation_bringup.launch.py (line 197)](/home/amoy/fsac_driveless/driverless_workspace/workspace/amoyfs_simulation/src/simulation_bringup/launch/simulation_bringup.launch.py:197)

转换关系是：

```
Gazebo /racecar/odometry
    gz.msgs.Odometry
        ↓
ROS /odom
    nav_msgs/msg/Odometry
```

同时转换：

```
/racecar/tf           → /tf
/racecar/joint_states → /joint_states
```

其中：

- `/odom`：Planning 和控制算法使用。
- `/tf`：RViz、坐标变换、规划使用。
- `/joint_states`：车辆可视化和 `robot_state_publisher` 使用，不直接参与 PID。

## 4. control_node 订阅 `/odom`

`control_node` 默认值是 `/vehicle/odometry`，但当前联合仿真配置把它改成了 `/odom`：

[gazebo_path_control.yaml (line 53)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_control_node/config/gazebo_path_control.yaml:53)

```
trajectory_topic: /planning/centerline
odometry_topic: /odom
path_in_vehicle_frame: true
control_request_topic: /control/request
```

订阅代码位于：

[control_node.cpp (line 97)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_control_node/src/control_node.cpp:97)

```
/planning/centerline → nav_msgs/Path
/odom                → nav_msgs/Odometry
```

每次收到 `/odom`，它保存最新消息，并更新输入 watchdog。

## 5. 控制周期怎样使用 `/odom`

控制周期位于：

[control_node.cpp (line 172)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_control_node/src/control_node.cpp:172)

首先检查：

```
中心线是否收到
里程计是否收到
时间戳是否新鲜
/clock 是否持续前进
```

任何条件失败都会：

```
longitudinal_pid_.reset();
publish_stop();
```

随后从 odometry 读取：

```
const auto& pose = odometry->pose.pose;
const double speed = odometry->twist.twist.linear.x;
```

纵向 PID 使用：

```
误差 = 目标速度 - /odom 实际纵向速度
```

对应代码：

[control_node.cpp (line 241)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_control_node/src/control_node.cpp:241)

```
const auto lateral = pure_pursuit_.compute(vehicle, points);

const auto longitudinal =
  longitudinal_pid_.compute(planned_speed_mps, speed, 1.0 / control_rate_hz_);
```

最后生成：

```
steering_angle = Pure Pursuit 结果
acceleration   = PID 结果
speed          = 曲率速度规划结果
```

发布到：

```
/control/request
ackermann_msgs/msg/AckermannDriveStamped
```

### 当前配置的一个重要细节

因为：

```
path_in_vehicle_frame: true
```

Planning 发布的中心线已经位于 `base_footprint` 坐标系中，所以 Pure Pursuit 计算时把车辆视为：

```
x = 0
y = 0
yaw = 0
```

因此 `/odom` 对 `control_node` 的直接作用主要是：

- 提供实际车速；
- 提供输入时间戳和存活性；
- 为动态速度规划提供平面速度。

车辆世界位置则先被 Planning 用来生成局部 `base_footprint` 中心线，再间接影响控制器。




# CAN反馈：Gazebo动力学 → 虚拟ECU → ros2_control

这条链路不直接给 PID 提供速度，而是模拟实车 VCU、电机控制器和 BMS 的反馈。

## 1. 动力学插件发布 telemetry

插件在每次 `PostUpdate()` 中发布：

[gazebo_vehicle_dynamics_system.cpp (line 221)](/home/amoy/fsac_driveless/driverless_workspace/workspace/amoyfs_simulation/src/vehicle_dynamics/src/vehicle_dynamics_gz_system/gazebo_vehicle_dynamics_system.cpp:221)

```
/vehicle_dynamics/telemetry
amoyfs_interfaces/msg/VehicleDynamicsTelemetry
```

包含：

```
x_m / y_m / yaw_rad
vx_mps / vy_mps / yaw_rate_radps
四轮轮速
四轮轮端扭矩
油门和制动
左右实际转角
电机转速
纵向和横向加速度
```

其中：

- `x/y/yaw` 已经从 Gazebo 世界位姿同步。
- `vx/vy/yaw_rate` 来自动力学状态。
- 转角、电机转速、扭矩来自执行器插件。

## 2. sim_can_ecu_bridge 接收 telemetry

节点入口：

[sim_can_ecu_bridge.cpp (line 44)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_sim_can_bridge/src/sim_can_ecu_bridge.cpp:44)

它订阅：

```
/vehicle_dynamics/telemetry
```

并检查 telemetry 是否：

- 已收到；
- 数值有限；
- 在 200 ms 内仍然新鲜。

## 3. 转换为三类 CAN 反馈

### 0x100：整车/VCU反馈

[sim_can_ecu_bridge.cpp (line 268)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_sim_can_bridge/src/sim_can_ecu_bridge.cpp:268)

转换字段：

```
telemetry.vx_mps
    → vehicle_speed_mps

左右实际转角平均值
    → steering_angle_deg

telemetry.brake × 最大制动压力
    → brake_pressure_bar

VirtualVcu.drive_enabled
    → drive_enabled

VirtualVcu.vehicle_mode
    → vehicle_mode
```

发布频率：100 Hz。

### 0x102：电机反馈

```
telemetry.motor_speed_radps
    → motor_speed_rpm

后轴轮端总扭矩 ÷ 传动比 ÷ 传动效率
    → motor_torque_nm
```

发布频率：100 Hz。

### 0x101：BMS反馈

当前是虚拟值和简化计算：

```
电池电压       → 配置常量
SOC            → 配置常量
温度           → 配置常量
电池电流       → 电机机械功率 / 母线电压
fault_flags    → 0
```

发布频率：10 Hz。

三类反馈通过 `vcan0` 发送。

## 4. SocketCAN hardware 接收反馈

`FsacSocketCanSystem` 的后台线程从 `vcan0` 接收：

```
0x100
0x101
0x102
```

接收和解码入口：

[fsac_socketcan_system.cpp (line 448)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_socketcan_system/src/fsac_socketcan_system.cpp:448)

解码结果保存到线程安全快照中，不直接在接收线程操作 ros2_control 接口。

## 5. controller_manager 调用 read()

`controller_manager` 当前配置为 250 Hz：

[ros2_controllers.simulation.yaml (line 2)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/racecar_description/config/ros2_controllers.simulation.yaml:2)

每个周期调用：

[fsac_socketcan_system.cpp (line 276)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_socketcan_system/src/fsac_socketcan_system.cpp:276)

```
FsacSocketCanSystem::read()
```

它把 CAN 快照写入 ros2_control state interfaces：

```
vehicle_feedback/vehicle_speed_mps
vehicle_feedback/steering_angle_rad
vehicle_feedback/brake_pressure_bar
vehicle_feedback/drive_enabled
vehicle_feedback/vehicle_mode
vehicle_feedback/communication_ok

bms_feedback/...
motor_feedback/...
```

`communication_ok` 不只是“收到过报文”，还检查：

- 报文是否超时；
- AliveCounter 是否连续；
- 数据是否成功解码。

---

# 四、CAN反馈最后进入哪里

## 1. 进入 command_gate_controller

command gate 只读取与安全决策有关的接口：

[command_gate_controller.cpp (line 41)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_command_gate_controller/src/command_gate_controller.cpp:41)

```
vehicle_feedback/communication_ok
vehicle_feedback/drive_enabled
vehicle_feedback/vehicle_mode
bms_feedback/communication_ok
bms_feedback/fault_flags
motor_feedback/communication_ok
motor_feedback/fault_flags
```

它不读取 `vehicle_speed_mps` 来运行 PID。PID 使用的是 `/odom`。

command gate 综合三种输入：

```
/control/request             算法请求
/safety/motion_authority     运动授权
ros2_control state interfaces  VCU、BMS、电机反馈
```

生成：

```
request_valid
authority
feedback_healthy
enable_confirmed
explicit_fault
emergency_requested
```

然后推进状态机：

[command_gate_controller.cpp (line 269)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_command_gate_controller/src/command_gate_controller.cpp:269)

```
DISARMED
   ↓ 反馈健康
REQUESTING_ENABLE
   ↓ drive_enabled=true 且 vehicle_mode=AUTONOMOUS
ACTIVE
   ↓
放行 Pure Pursuit/PID 的转角与加速度
```

具体的 `REQUESTING_ENABLE → ACTIVE` 代码仍在：

[command_gate_state_machine.cpp (line 85)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_command_gate_controller/src/command_gate_state_machine.cpp:85)

```
if (inputs.enable_confirmed)
{
  state_ = GateState::kActive;
}
```

## 2. 进入 vehicle_state_broadcaster

第二个消费者是只读 broadcaster：

[vehicle_state_broadcaster.cpp (line 237)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/fsac_vehicle_state_broadcaster/src/vehicle_state_broadcaster.cpp:237)

它把 ros2_control state interfaces 转成 ROS 消息：

```
/vehicle/status
/vehicle/bms_status
/vehicle/motor_status
/diagnostics
```

这部分主要给：

- 上位机；
- Foxglove；
- 故障诊断；
- 运行日志；

使用，不参与 Pure Pursuit/PID 数学计算。

---