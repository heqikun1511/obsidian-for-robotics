
这是 ROS 2 中 **TF2 坐标变换**的核心调用，用来查询两个坐标系之间的变换关系。下面逐行拆解。

## 一、整体作用

cpp

const auto transform = tf_buffer_.lookupTransform(
    target_frame_,                              // 目标坐标系
    message->header.frame_id,                   // 源坐标系
    rclcpp::Time(message->header.stamp),        // 查询时间
    rclcpp::Duration::from_seconds(tf_timeout_sec_));  // 超时

含义：**在时间 `stamp`，查询从 `message->header.frame_id` 到 `target_frame_` 的坐标变换**，并把它存到 `transform` 里。

## 二、参数逐个解析

### 参数 1：`target_frame_`

- 类型：`std::string`，成员变量（通常末尾下划线表示成员）
    
- **目标坐标系**，即"我要变换到哪个坐标系"
    
- 例如：`"base_link"`（机器人基座）、`"map"`（地图坐标系）

### 参数 2：`message->header.frame_id`

- 类型：`std::string`
    
- **源坐标系**，即"这个点云/数据当前在哪个坐标系"
    
- 例如：`"camera_link"`、`"velodyne"`、`"laser"`
### 参数 3：`rclcpp::Time(message->header.stamp)`

- 查询的**时间点**
    
- `message->header.stamp` 是消息自带的时间戳，类型是 `builtin_interfaces::msg::Time`
    
- 用 `rclcpp::Time(...)` 转换后，才能和 TF 缓冲区的时间戳比较
    
- **为什么要用消息时间戳？** 因为机器人是运动的，不同时刻坐标系之间的相对位置不同。用消息时间戳能保证：**数据产生时刻的坐标变换**，而不是当前时刻的
### 参数 4：`rclcpp::Duration::from_seconds(tf_timeout_sec_)`

- **超时时间**
    
- 如果 TF 数据还没到齐（比如变换需要多个坐标系串联，中间某个还没发布），`lookupTransform` 会**等待**最多 `tf_timeout_sec_` 秒
    
- 超时后抛出 `tf2::LookupException` 异常
    
- 常见取值：`0.1` ~ `1.0` 秒