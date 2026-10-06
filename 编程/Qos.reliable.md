

# `rclcpp::QoS(1).reliable()` 解析

这是 ROS 2 (C++) 中构造 **QoS（服务质量）配置**的常见写法，用于订阅者、发布者等。下面详细拆解。

## 逐部分解析

### 1. `rclcpp::QoS`

- `rclcpp::QoS` 是 ROS 2 中封装服务质量策略的类
    
- 它同时包含了 **可靠性（Reliability）**、**持久性（Durability）**、**历史（History）**、**队列深度（Depth）** 等策略
    
- 构造时可以传一个整数，表示**历史队列深度（history depth）**
    

### 2. `QoS(1)`

- 括号里的 `1` 是 **队列深度（depth）**
    
- 默认历史策略是 `KEEP_LAST`，即"只保留最近的 N 条消息"
    
- 所以 `QoS(1)` 等价于：
    
    - History = `KEEP_LAST`
        
    - Depth = `1`
        
    - 也就是**只保留最新 1 条消息**，旧的会被丢弃
        

### 3. `.reliable()`

- 这是一个**链式调用**，设置可靠性策略为 `RELIABLE`
    
- 即：**保证消息送达**（底层通过重传机制实现，类似 TCP）
    
- 与之相对的是 `.best_effort()`（尽力而为，可能丢包，类似 UDP）