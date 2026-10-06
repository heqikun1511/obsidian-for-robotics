## `ros2_control` 到底是什么？

`ros2_control` 是 ROS 2 的机器人控制框架。它的核心目标不是替你发明控制算法，而是把不同硬件（Gazebo、CAN 电机、串口舵机、EtherCAT）统一成相同的**状态接口**和**命令接口**，并由 `controller_manager` 按固定周期调度。

```
Controller（算法/命令分配）
  read command topic/action
  write joint command interfaces
           │
controller_manager（生命周期、资源占用、update 循环）
           │
Hardware interface
  read():  实车反馈 / Gazebo 状态 → state interfaces
  write(): command interfaces → CAN/串口/Gazebo
```


## Joint interfaces：控制器和硬件说的“共同语言”

常见 interface：

| 接口                 | 含义         | 单位          | 车辆中的典型用途     |
| ------------------ | ---------- | ----------- | ------------ |
| `position` command | 目标位置 / 角度  | m 或 rad     | 前轮转向角、云台、机械臂 |
| `velocity` command | 目标速度 / 角速度 | m/s 或 rad/s | 轮电机角速度       |
| `effort` command   | 力 / 扭矩     | N 或 N·m     | 更真实的驱动轮、力控   |
| `position` state   | 实际位置 / 角度  | m 或 rad     | 编码器角度、舵机反馈   |
| `velocity` state   | 实际速度       | m/s 或 rad/s | 轮速、转向速度      |
| `effort` state     | 实际力矩 / 力   | N 或 N·m     | 电机电流估算、力传感器  |


**同一 joint 可以有多个 state interface，但命令接口必须按控制模式谨慎设计。**

例如前转向关节：

```
<joint name="front_left_steering_joint">
  <command_interface name="position"/>
  <state_interface name="position"/>
  <state_interface name="velocity"/>
</joint>
```

后驱轮：

```
<joint name="rear_left_wheel_joint">
  <command_interface name="velocity"/>
  <state_interface name="position"/>
  <state_interface name="velocity"/>
</joint>
```

## 生命周期与 update 循环

典型控制周期：

```
每个周期 t_k：
  1. hardware.read(t_k, dt)
     将最新反馈写入 state interfaces
  2. controller_manager.update(t_k, dt)
     各 active controller 读取 state、计算、写 command interfaces
  3. hardware.write(t_k, dt)
     将 command interfaces 发给执行器
```

Controller 通常有生命周期：

```
unconfigured → inactive → active
                    ↑        │
                    └────────┘
```

- `configure`：读取参数、校验 joint 名称与接口；
    
- `activate`：开始 claim 接口并输出命令；
    
- `deactivate`：停止占有接口；
    
- `cleanup`：释放资源。
    

为什么要理解 claim？因为**一个 command interface 在同一时刻不能被两个普通控制器同时写**。例如 `front_left_steering_joint/position` 不应同时被 `forward_position_controller` 和 `ackermann_steering_controller` 占有。


## 重要控制器速查

| Controller                      | 输入                           | 输出 / 作用                 | 无人车用途             |
| ------------------------------- | ---------------------------- | ----------------------- | ----------------- |
| `joint_state_broadcaster`       | 无                            | 发布 `/joint_states`      | 必备，供 TF/RViz/诊断使用 |
| `ackermann_steering_controller` | 车体速度与转向命令                    | 左右转向角、驱动轮速度、odom        | Ackermann 车辆主控制器  |
| `diff_drive_controller`         | `/cmd_vel`                   | 左右轮速度、odom              | 差速车；不用于赛车         |
| `joint_trajectory_controller`   | FollowJointTrajectory action | 多 joint 位置/速度/努力轨迹      | 云台、机械臂、复杂舵机组      |
| `forward_command_controller`    | 数组命令 topic                   | 直接透传到指定 joint interface | Bring-up、单独测轮子/舵机 |
| `pid_controller`                | 参考值与状态                       | PID 控制量                 | 需要显式 PID 结构时      |
| `gpio_controllers`              | GPIO command/state           | 灯、继电器、离散 I/O            | 安全灯、蜂鸣器等          |


### `joint_state_broadcaster`

它没有“运动控制”能力，只把所有已注册的 state interfaces 汇总发布为 `sensor_msgs/msg/JointState`。但它几乎总是第一个启动：

```
joint_state_broadcaster:
  type: joint_state_broadcaster/JointStateBroadcaster
```

启动后检查：

```
ros2 control list_controllers
ros2 topic echo /joint_states --once
```

### `forward_command_controller`

它适合 Bring-up，而不是最终无人车控制。它把一个 ROS topic 的数组直接写进指定接口。

典型用途：

- 单独验证“左后驱轮速度命令是否能转”；
    
- 验证前轮转向正负与限位；
    
- 暂时替代未完成的车辆控制器。
    

测试时务必先设置保守命令和软件限位；实车轮子应悬空或使用安全支架。

### `ackermann_steering_controller`

它是车辆运动学控制器。它接收车体层面的速度与转向需求，再利用：

- 轴距（wheelbase）；
    
- 前后轮距（track width）；
    
- 轮半径；
    
- joint 名称与正方向；
    

计算左右转向关节和驱动轮的命令。

你应把它理解成一个**运动学分配器**：它不负责判断赛车是否该避开锥桶，也不负责从路径中挑 lookahead point。

对于真正的四轮独立驱动或复杂差速器，务必检查你当前 `ros2_controllers` 版本的参数接口是否正好符合车辆结构；若不完全匹配，保持上层 `AckermannDrive` 契约不变，并实现你自己的车辆级 controller / 分配器通常更干净




## Controller YAML 的结构

一个典型的 controller 配置有两层：

```
controller_manager:
  ros__parameters:
    update_rate: 250

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    vehicle_controller:
      type: ackermann_steering_controller/AckermannSteeringController

vehicle_controller:
  ros__parameters:
    # 这里填写当前 controller 版本要求的车轮、转向关节与几何参数
```

`controller_manager` 那层决定“加载哪些 controller、它们的类型、控制周期”；具体 controller 自己的参数写在同级另一节。
## 从控制器到 Gazebo：`gz_ros2_control`

Gazebo Sim 中的标准接口是 `gz_ros2_control`：

```
ROS 2 controller
    ↓ command/state interface
gz_ros2_control/GazeboSimSystem
    ↓
Gazebo Sim joints
```

URDF/Xacro 中必须同时具备两部分。

### ros2_control 硬件声明

```
<ros2_control name="GazeboSimSystem" type="system">
  <hardware>
    <plugin>gz_ros2_control/GazeboSimSystem</plugin>
  </hardware>

  <joint name="rear_left_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>

  <joint name="front_left_steering_joint">
    <command_interface name="position"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
</ros2_control>
```



### Gazebo 系统插件加载

```
<gazebo>
  <plugin filename="libgz_ros2_control-system.so"
          name="gz_ros2_control::GazeboSimROS2ControlPlugin">
    <parameters>$(find-pkg-share racecar_control)/config/controllers.yaml</parameters>
    <ros>
      <namespace>car_01</namespace>
    </ros>
  </plugin>
</gazebo>
```

然后在 launch 中 spawn 机器人，加载并 activate controller。多车时每辆车应有独立 namespace、独立 controller manager 与关节命名；避免所有车辆都向全局 `/joint_states`、`/tf`、`/odom` 发布。

## 不要与 Gazebo 原生 Ackermann 插件混用

Gazebo 的：

```
gz::sim::systems::AckermannSteering
```

是**另一套独立控制链路**：它订阅 Gazebo Transport 的 `cmd_vel`，自己计算并直接控制 Gazebo joints。

若你采用：

```
Pure Pursuit → ROS Ackermann Controller → gz_ros2_control → joints
```

则不要同时让 `gz::sim::systems::AckermannSteering` 控制相同的轮子和转向 joints。否则两个系统同时写同一关节，结果会是命令争夺、抖动或 odom/TF 重复发布。

选型：

- 快速纯 Gazebo baseline：可使用 Gazebo 原生 `AckermannSteering` + `ros_gz_bridge`；
    
- 仿真—实车一致的无人车开发：使用 `gz_ros2_control` + ROS controller，推荐本项目采用这一条。
### `read()`：从底层读取状态

建议从 STM32 / VCU 获得并验证：

- 四轮编码器位置、轮速；
    
- 左右前轮实际转向角；
    
- 电机状态、电流、母线电压、温度；
    
- 心跳、CAN 通信状态、故障码；
    
- MCU 时间戳与数据新鲜度。
    

将 SI 单位写到 state interfaces：rad、rad/s、N·m 等。不要把 CAN 原始计数直接暴露给上层 controller。


### `write()`：向底层发命令

在进入 CAN / SPI 之前执行：

- 数值是否有限（拒绝 `NaN` / `Inf`）；
    
- 位置、速度、扭矩幅值限幅；
    
- 角速度 / 加速度变化率限幅；
    
- 安全状态是否允许执行；
    
- 底层通信是否超时。
    

并定义明确的失联行为：例如当上层命令超过 100 ms 未更新时，速度目标归零、转向保持或回中（具体策略依安全设计决定）。