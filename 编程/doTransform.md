[[编程]]
这是 ROS 2 中**最常用的坐标变换调用**，用来把一个点云（或其它带 `header` 的消息）从源坐标系变换到目标坐标系。

## 一、整体作用

cpp

tf2::doTransform(*message, cloud_in_target, transform);

含义：**使用 `transform` 这个变换，把 `*message` 变换到目标坐标系，结果存到 `cloud_in_target`。**

一句话：**应用坐标变换，把点云搬到新坐标系。**

## 方向性（关键！）

`tf2::doTransform` 的语义是：

> **把输入消息中的每个点，用 transform 变换一次，输出到目标坐标系。**

而 `transform` 来自：

cpp

auto transform = tf_buffer_.lookupTransform(
    target_frame_,          // 目标坐标系
    message->header.frame_id,  // 源坐标系
    time, timeout);

所以组合起来：

text

源坐标系 (message.frame_id)  ──transform──►  目标坐标系 (target_frame_)


## `tf2::doTransform` 做了什么？

它内部大致做了这些事：

1. **读取 transform**：从 `TransformStamped` 中取出平移和旋转（四元数）
    
2. **构造 4×4 变换矩阵**（隐式）
    
3. **遍历点云中每个点**：
    
    - 取 `(x, y, z)`
        
    - 应用旋转 + 平移
        
    - 写回输出点云
        
4. **更新输出点云的 header**：
    
    - `frame_id` 设为 `target_frame_`
        
    - `stamp` 保持和输入一致
        

**注意**：`tf2::doTransform` 只变换**坐标**，不改变点云的结构（宽度、高度、字段、是否有序等）。