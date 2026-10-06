# 这段代码的作用与为什么"不能用最新姿态替代历史姿态"

cpp

// 查询点云采样时刻的 TF，不能用最新姿态替代历史姿态。
const auto transform = tf_buffer_.lookupTransform(
    target_frame_, message->header.frame_id, rclcpp::Time(message->header.stamp),
    rclcpp::Duration::from_seconds(tf_timeout_sec_));
tf2::doTransform(*message, cloud_in_target, transform);

这段代码做了两件事：

1. **按点云的时间戳**查询 TF 变换
    
2. **应用变换**把点云转到 `target_frame_`
    

注释里那句"不能用最新姿态替代历史姿态"是**整个坐标变换的关键原则**。下面详细解释。


### 步骤

text

1. lookupTransform(target, source, stamp, timeout)
   → 查询 stamp 时刻，source → target 的变换
2. tf2::doTransform(*message, cloud_in_target, transform)
   → 把点云中每个点应用这个变换，得到 target 坐标系下的点云

---

## 一、这段代码在做什么？

### 输入

- `message`：一帧点云，带 `header.stamp`（采样时间）和 `header.frame_id`（传感器坐标系）
    
- `target_frame_`：目标坐标系（如 `base_link`、`map`）
    
- `tf_buffer_`：TF 缓冲区，存着历史坐标变换

### 场景：机器人一边移动一边扫点云

假设激光雷达装在机器人上，机器人一边前进一边扫描：

text

时刻 t₁：雷达位置 A，扫描得到点云 P₁
时刻 t₂：雷达位置 B（机器人移动了），扫描得到点云 P₂
时刻 t₃：雷达位置 C，扫描得到点云 P₃

现在你想把 P₁ 转到机器人基座 `base_link`：

### ❌ 错误做法：用最新姿态

cpp

// 用当前时刻的变换
auto transform = tf_buffer_.lookupTransform(
    "base_link", "laser", this->now());   // ← 用 now()，错误！
tf2::doTransform(P1, P1_in_base, transform);

问题：

- `P₁` 是 t₁ 时刻采集的，雷达在位置 A
    
- 但 `now()` 时刻雷达可能在位置 C
    
- 用 C 时刻的变换去变换 A 时刻的数据 → **点云被"错位"**


## 为什么必须用 `message->header.stamp`？

cpp

rclcpp::Time(message->header.stamp)   // ← 点云采样时刻

### `header.stamp` 是什么？

每条 ROS 消息都带 `std_msgs/Header`：

cpp

struct Header {
    builtin_interfaces::msg::Time stamp;   // 时间戳
    std::string frame_id;                   // 坐标系
};

- `stamp`：**数据产生时刻**（传感器采样时间）
    
- `frame_id`：**数据所在的坐标系**
    

### 这个时间戳由谁填？

| 来源    | 说明             |
| ----- | -------------- |
| 传感器驱动 | 激光雷达在采样瞬间打时间戳  |
| 相机驱动  | 图像曝光时刻         |
| 深度相机  | 深度帧生成时刻        |
| 消息转发  | 保持原始时间戳（不应重新打） |

**关键**：`stamp` 是**数据本身的时刻**，不是"收到消息的时刻"。

### 时间戳链条

text

传感器采样（t₁）→ 打时间戳 stamp=t₁ → 发布 → 传输延迟 → 节点收到（t₂）
                                                              │
                                                              ▼
                                          用 stamp=t₁ 查询 TF（正确）
                                          用 now()=t₂ 查询 TF（错误）

网络传输、队列等待都会带来延迟，如果用"收到时刻"查 TF，就会错位。

---


## TF2 的时间语义

TF2 的设计就是为了支持**历史时刻查询**：

### TF 缓冲区

cpp

tf2_ros::Buffer tf_buffer_(this->get_clock());

- 默认缓存 **10 秒**的历史变换
    
- 每个变换带时间戳
    
- 支持**任意时刻**查询（只要在缓存范围内）
    

### `lookupTransform` 的时间参数

cpp

lookupTransform(target, source, time, timeout)
                              ↑
                        查哪个时刻的变换

- 传 `stamp`：查历史时刻（**正确**）
    
- 传 `now()`：查最新时刻（**错误**）
    
- 传 `Time(0)`：查最新可用（**也不推荐用于传感器数据**）

## 为什么"运动补偿"必须用历史姿态？

### 核心概念：运动补偿（Motion Compensation）

传感器扫描需要时间：

|传感器|扫描时间|
|---|---|
|2D 激光雷达|几十 ms（转一圈）|
|3D 激光雷达|100 ms（16 线转一圈）|
|深度相机|一帧曝光时间|
|滚动快门相机|逐行曝光|

**在扫描期间，机器人还在运动**。所以：

text

一帧点云里，不同点的采样时刻其实不同！
3D 激光雷达：
  t=0ms   → 第 1 个点
  t=10ms  → 第 100 个点
  t=50ms  → 第 500 个点
  t=100ms → 最后一个点