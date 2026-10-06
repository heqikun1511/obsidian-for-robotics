
## 话题名参数（Topic Names）

| 参数名                       | 默认值                                 | 含义                         |
| ------------------------- | ----------------------------------- | -------------------------- |
| `emergency_stop_topic`    | `/perception/emergency_stop`        | 发布**紧急停车**布尔信号的话题          |
| `forward_clearance_topic` | `/perception/forward_clearance`     | 发布**前方可通行距离**（米）的话题        |
| `speed_scale_topic`       | `/perception/speed_scale`           | 发布**速度缩放系数**（0~1）的话题，用于减速  |
| `turn_bias_topic`         | `/perception/recommended_turn_bias` | 发布**推荐转向偏置**（弧度）的话题，用于避障转向 |
| `state_topic`             | `/perception/safety_state`          | 发布**安全状态**（枚举/整数）的话题       |


|   |
|---|
|发布**安全状态**（枚举/整数）的话题|

> 这些都声明为 `std::string`，默认带 `/perception/` 前缀，说明属于感知层输出，供规划/控制层订阅。

---

## 二、时间与频率参数

|参数名|默认值|含义|
|---|---|---|
|`tf_timeout_sec`|`0.05`|TF 查询超时（秒），太大会拖慢回调|
|`cloud_timeout_sec`|`0.20`|点云超时（秒），超过则认为数据过旧、触发看门狗|
|`watchdog_rate_hz`|`15.0`|看门狗定时器频率（Hz），周期性检查点云是否新鲜|

> `tf_timeout_sec = 0.05` 很短，说明要求 TF 必须及时可用；`cloud_timeout_sec = 0.2` 表示 200ms 没收到点云就判定"感知失效"。

---

## 三、点云空间范围参数（ROI）

| 参数名               | 默认值     | 含义                           |
| ----------------- | ------- | ---------------------------- |
| `min_range_m`     | `0.20`  | 最小水平距离（米），比这更近的点忽略（可能是自车/噪声） |
| `max_range_m`     | `2.50`  | 最大水平距离（米），超出范围的远处点忽略         |
| `min_forward_x_m` | `0.05`  | 前方 x 方向最小距离（米），只关心车头前方       |
| `max_lateral_y_m` | `1.50`  | 横向 y 方向最大范围（米），左右各 1.5 米     |
| `min_height_z_m`  | `-0.40` | 高度 z 方向最小值（米），过滤地面以下         |
| `max_height_z_m`  | `1.20`  | 高度 z 方向最大值（米），过滤高空           |
|                   |         |                              |

> 这组参数共同定义了**感兴趣区域（ROI）**——一个以机器人为中心的长方体空间，只在这个范围内的点才参与安全判断。正好呼应你之前问的 ROI 概念。

---

## 四、点云预处理参数

| 参数名                 | 默认值    | 含义                          |
| ------------------- | ------ | --------------------------- |
| `voxel_leaf_size_m` | `0.03` | 体素滤波的叶大小（米），3cm 立方体降采样，减少点数 |
| `min_valid_points`  | `80`   | 有效点云的最小点数，少于这个数认为数据不可靠      |

> **体素滤波**：把空间划成 3cm 的小立方体，每个立方体只保留一个点，用来降采样、加速后续计算。

---

## 五、自车过滤（Self Filter）参数

|参数名|默认值|含义|
|---|---|---|
|`self_filter_enabled`|`false`|是否启用自车过滤|
|`self_box_min`|`{-0.20, -0.35, -0.30}`|自车包围盒最小值 `(x, y, z)`|
|`self_box_max`|`{0.35, 0.35, 0.35}`|自车包围盒最大值 `(x, y, z)`|

> 传感器可能扫到机器人自身（车体、机械臂等），用一个长方体盒子把这部分点滤掉。盒子范围由 `min` 和 `max` 定义。

---

## 六、地面过滤（Ground Filter）参数

|参数名|默认值|含义|
|---|---|---|
|`ground_filter_enabled`|`true`|是否启用地面分割|
|`ground_distance_threshold_m`|`0.035`|点到地面平面的距离阈值（米），3.5cm 内算地面|
|`ground_min_points`|`120`|拟合地面所需的最少点数|
|`constrain_ground_normal`|`true`|是否约束地面法向量（防止把斜面当地面）|
|`ground_max_tilt_deg`|`20.0`|地面最大倾斜角（度），超过 20° 不算地面|

> 用 RANSAC 拟合地面平面，把地面点分离出去（`ground_pub_` 发布），剩下的才是障碍物点（`obstacle_pub_`）。

---

## 七、扇区与安全判断参数

|参数名|默认值|含义|
|---|---|---|
|`sector_count`|`7`|把前方视场分成 7 个扇区（左、中、右等）|
|`sector_half_fov_deg`|`55.0`|扇区半视场角（度），总共 110° 视野|
|`forward_corridor_half_width_m`|`0.28`|正前方走廊的半宽（米），用于判断"正前方"是否畅通|
|`stop_distance_m`|`0.35`|停车距离（米），障碍物进入此范围立即停车|
|`slow_distance_m`|`0.75`|减速距离（米），进入此范围开始减速|
|`clear_distance_m`|`0.85`|畅通距离（米），超过此距离认为无障碍|
|`confirm_frames`|`2`|连续 N 帧检测到障碍才确认（防抖动）|
|`clear_frames`|`3`|连续 N 帧检测到畅通才解除（防抖动）|
|`min_speed_scale`|`0.25`|最小速度缩放（0.25 = 最低降到 25% 速度）|
|`max_turn_bias_rad`|`0.22`|最大转向偏置（弧度），约 12.6°|

> 这组是**安全决策的核心**：
> 
> - 障碍物 < 0.35m → 停车
>     
> - 0.35~0.75m → 减速
>     
> - > 0.85m → 畅通
>     
> - 用 `confirm_frames` / `clear_frames` 做**时序确认**，避免一帧误检就急停
>     

---

## 八、代价地图（Costmap）栅格参数

|参数名|默认值|含义|
|---|---|---|
|`grid_resolution_m`|`0.05`|栅格分辨率（米），5cm/格|
|`grid_min_x_m`|`-0.30`|栅格地图 x 方向起点（米）|
|`grid_min_y_m`|`-1.50`|栅格地图 y 方向起点（米）|
|`grid_width_m`|`3.00`|地图宽度（米）|
|`grid_height_m`|`3.00`|地图高度（米）|

> 把障碍物点云投影成一张 `nav_msgs::msg::OccupancyGrid` 代价地图，供规划层使用。地图覆盖 x ∈ [-0.3, 2.7]，y ∈ [-1.5, 1.5]，分辨率 5cm。

---

## 九、启动时的安全校验（重要）

代码里做了三组校验，**错误直接抛异常终止启动**，这是安全关键系统的常见做法：

cpp

// 1. 自车包围盒必须是 3 维
if (self_box_min_.size() != 3U || self_box_max_.size() != 3U) {
    throw std::runtime_error("self_box_min 和 self_box_max 必须包含 x、y、z 三个值");
}
// 2. 距离阈值必须递增：stop < slow < clear
if (stop_distance_m_ >= slow_distance_m_ || slow_distance_m_ >= clear_distance_m_) {
    throw std::runtime_error("必须满足 stop_distance_m < slow_distance_m < clear_distance_m");
}
// 3. 频率、体素、栅格、扇区必须为正
if (watchdog_rate_hz_ <= 0.0 || voxel_leaf_size_m_ <= 0.0 ||
    grid_resolution_m_ <= 0.0 || sector_count_ < 3) {
    throw std::runtime_error("watchdog、体素、栅格和扇区参数必须为有效正值");
}

**为什么直接崩溃而不是警告？** 因为参数错误会导致安全逻辑失效（比如 stop > slow 时永远不减速），在实机上可能造成事故。**启动时失败比运行时失控更安全**。

---

## 十、发布者与订阅者

### 发布者（都用 `reliable` QoS）

|发布者|类型|话题|
|---|---|---|
|`obstacle_pub_`|`PointCloud2`|障碍物点云|
|`ground_pub_`|`PointCloud2`|地面点云|
|`costmap_pub_`|`OccupancyGrid`|局部代价地图|
|`emergency_stop_pub_`|`Bool`|紧急停车信号|
|`clearance_pub_`|`Float32`|前方可通行距离|
|`speed_scale_pub_`|`Float32`|速度缩放|
|`turn_bias_pub_`|`Float32`|转向偏置|
|`state_pub_`|`UInt8`|安全状态|

### 订阅者（用 `SensorDataQoS`）

cpp

cloud_sub_ = create_subscription<sensor_msgs::msg::PointCloud2>(
    input_topic_, rclcpp::SensorDataQoS().keep_last(1), ...);

- **输入**：点云，用 `SensorDataQoS().keep_last(1)`——高频传感器数据，**只保留最新一帧**，丢旧帧没关系
    
- **输出**：安全指令、代价地图，用 `reliable`——**必须送达**，不能丢
    

> 这正好呼应你之前问的 QoS：**传感器输入用 best_effort/keep_last(1)，安全输出用 reliable**。

---

## 十一、看门狗定时器

cpp

const auto watchdog_period = std::chrono::duration_cast<std::chrono::milliseconds>(
    std::chrono::duration<double>(1.0 / watchdog_rate_hz_));
watchdog_timer_ = create_wall_timer(watchdog_period, ...);

- 以 `watchdog_rate_hz_`（15Hz）周期调用 `watchdogCallback`
    
- 作用：**周期检查点云是否超时**（超过 `cloud_timeout_sec` 没收到），如果超时则触发安全停车
    
- 这是**失效安全（fail-safe）**设计——感知失效时不能"假装没事"，必须主动停车
    

---

## 十二、整体架构总结

text

       点云输入 (SensorDataQoS)
              │
              ▼
    ┌─────────────────────┐
    │  1. ROI 裁剪         │ ← min/max_range, forward/lateral/height
    │  2. 体素降采样        │ ← voxel_leaf_size_m
    │  3. 自车过滤          │ ← self_filter_*
    │  4. 地面分割          │ ← ground_filter_*
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  5. 扇区分析          │ ← sector_count, sector_half_fov
    │  6. 距离判断          │ ← stop/slow/clear_distance
    │  7. 时序确认          │ ← confirm_frames, clear_frames
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  8. 输出              │
    │   - 紧急停车 (Bool)   │
    │   - 可通行距离 (Float)│
    │   - 速度缩放 (Float)  │
    │   - 转向偏置 (Float)  │
    │   - 代价地图 (Grid)   │
    └─────────────────────┘
              ▲
              │
       看门狗定时器 (15Hz) ← 监控点云是否超时

---

## 十三、一句话总结

> 这组参数把一个**点云安全节点**的所有可调项都暴露出来：**输入范围（ROI）、预处理（体素/自车/地面）、安全阈值（停/慢/清）、输出话题、时序防抖、代价地图尺寸、看门狗频率**。并且在启动时对参数做**严格校验**，错误直接抛异常——这是安全关键系统的标准做法。发布安全指令用 `reliable` QoS，订阅传感器用 `SensorDataQoS`，兼顾可靠性与实时性。