---
tags:
  - 无人车算法
---
# Pure Pursuit 路径跟踪

## 1. Pure Pursuit 要解决什么问题？

输入：

1. 当前车辆位姿 $((x,y,\psi))$，通常在 `map` 或 `odom` 坐标系；
    
2. 一串有序路径点 ($P_0,P_1,\ldots,P_n)$；
    
3. 当前车速 (v)；
    
4. 前视距离（$lookahead distance）(L_d)。$
    

输出：

- 曲率 ($\kappa$)，单位 `1/m`；
    
- 或前轮等效转向角 $(\delta)$，单位 `rad`。
    

核心思想很直观：**不盯着车正下方的路径点，而是找车前方距离约为 (L_d) 的目标点；令车辆沿一段圆弧驶向该点。**

## 2. 推导：从目标点到曲率

### 2.1 把目标点变换到车体坐标系

设当前位姿为 ((x_r,y_r,\psi))，全局路径上选中的目标点为 ((x_t,y_t))。

先得到全局相对位置：

$[$  
$\Delta x=x_t-x_r,\qquad \Delta y=y_t-y_r$  
$]$

再绕 $(-\psi)$ 旋转到 `base_link` 坐标系（ROS 中通常规定：x 向前、y 向左）：
$[xL​yL​​]=旋转矩阵，转 −ψ[cosψ−sinψ​sinψcosψ​]​​[ΔxΔy​]$

- 左边 (xL,yL)(xL​,yL​)：**目标点在车眼里的位置**
    
- 右边 (Δx,Δy)(Δx,Δy)：**目标点在地图上的位置偏差**
    
- 中间那个矩阵：**把地图视角旋转到车视角**

其中：

- (x_L>0)：目标在车前方；
    
- (y_L>0)：目标在左方，应左转；
    
- (y_L<0)：目标在右方，应右转。
    

若目标刚好位于 `base_link` 的 x 轴上，(y_L=0)，车辆直行。


### 2.2 圆弧几何


![[Pasted image 20260916091820.png]]

要让车辆从原点、初始朝向沿 x 正方向，走一段圆弧到达 ((x_L,y_L))。圆的圆心在 y 轴上，半径为 (R)。由几何关系可得：

$[$  
$R=\frac{x_L^2+y_L^2}{2y_L}$  
$]$

因此曲率：

$[$  
$\boxed{\kappa=\frac{1}{R}=\frac{2y_L}{x_L^2+y_L^2}}$  
$]$

若你的目标点严格取在半径为 (L_d) 的圆上，即 $(x_L^2+y_L^2=L_d^2)$，也常写为：

[  
$\boxed{\kappa=\frac{2y_L}{L_d^2}}$  
]

这两个式子本质一致。代码中推荐用前一个式子，因为实际路径离散、目标点插值和定位误差会让距离不严格等于 (L_d)。


## 3. 目标点应怎样选？

不要简单取“数组中第一个距离大于 (L_d) 的点”。这个写法能跑，但在低分辨率路径、急弯、闭环赛道或车辆已经偏离路径时容易跳点。

推荐流程：

1. 找到车辆到路径折线的最近投影点，并记录该点的累计弧长 (s_0)；
    
2. 取 ($s_t=s_0+L_d)$；
    
3. 在包含 (s_t) 的两条离散路径点之间做线性插值，得到目标点；
    
4. 如果路径是闭环，到末尾后按总弧长取模；
    
5. 将目标点变到 `base_link`，若出现 ($x_L\le0)$，说明目标选择或最近点索引出现跳跃，应重新限制搜索窗口。

### 为什么要用弧长而不是欧氏距离？

在发卡弯或自交赛道附近，几何距离最近的点可能属于赛道另一段。弧长索引保证目标点沿着“当前路径前进方向”选择。

工程上可维护一个 `last_nearest_index`，每个周期只在它前后有限窗口内找最近投影点，避免一次定位噪声使 index 跳到赛道另一圈。



## 一、先理解问题：为什么不能"简单取第一个距离 > L_d 的点"

最朴素的做法是：

> 从路径数组里，从头往后扫，找到第一个到车辆距离大于前视距离 LdLd​ 的点，就把它当目标点。

这个写法能跑，但很脆弱，原因有几个：

### 1. 路径点分辨率问题

如果路径点很稀疏（比如每 2 米一个点），那"第一个距离 > LdLd​"的点可能距离车 2.1 米，也可能距离车 3.9 米。目标点忽远忽近，转角就忽大忽小，车会抖。

### 2. 急弯 / 发卡弯

在发卡弯处，路径折回来，几何上离车很近的点，可能在路径的另一段。如果按欧氏距离找，可能选到一个"空间上近、但路径上还很远"的点，车就会往错误方向打方向。

### 3. 闭环赛道 / 自交赛道

赛道自己交叉时，交叉点附近有两段路径在空间上重合。按欧氏距离，可能选到另一圈的路径点上，导致目标点"跳圈"。

### 4. 车辆已经偏离路径

如果车偏离路径较远，最近点可能不在预期位置，简单的"第一个 > LdLd​"会选到很前面的点，造成大转角。


## 二、推荐流程逐步解释

### 第 1 步：找最近投影点，记录累计弧长 s0s0​

不是找"最近的路径点"，而是找**车辆到路径折线的最近投影点**。

- 路径是一段段线段组成的折线。
    
- 对每一段线段，求车辆位置到该线段的垂足（投影点）。
    
- 在所有投影点里，找距离车辆最近的那个。
    
- 这个投影点可能落在某条线段中间，不一定是离散路径点。
    
- 记录这个投影点在整条路径上的**累计弧长** s0s0​。
    

**为什么要弧长？** 因为弧长是"沿着路径走了多远"的度量，是路径的**内在坐标**，不受空间几何折叠的影响。

---

### 第 2 步：取 st=s0+Ldst​=s0​+Ld​

从当前投影点沿路径前进 LdLd​ 的弧长，得到目标点的弧长坐标 stst​。

这一步是核心：**目标点是按"沿路径前进 LdLd​"选的，不是按"空间距离 LdLd​"选的。**

### 第 3 步：在包含 stst​ 的两条离散路径点之间做线性插值

路径是离散的点，stst​ 通常落在两个离散点之间。用线性插值算出目标点的实际坐标：

$Pt=Pi+st−sisi+1−si(Pi+1−Pi)Pt​=Pi​+si+1​−si​st​−si​​(Pi+1​−Pi​)$
[[线性插值]]

这样目标点是连续的，不会因为路径点稀疏而跳变。

---

### 第 4 步：闭环路径按总弧长取模

如果路径是闭环，stst​ 超过总弧长 StotalStotal​ 时，取：

st=stmod  Stotalst​=st​modStotal​

这样车跑到终点后会自然回到起点继续，不会越界。

---

### 第 5 步：变到 `base_link`，检查 xL≤0xL​≤0

把目标点用上一轮讲的旋转矩阵变到车辆坐标系，得到 (xL,yL)(xL​,yL​)。

- 正常情况下，目标点应该在车前方，即 xL>0xL​>0。
    
- 如果出现 xL≤0xL​≤0，说明目标点跑到车后面去了，这通常意味着：
    
    - 最近点索引跳变了
        
    - 或者目标点选错了段
        
- 这时应该**重新限制搜索窗口**，重新找最近点。

## 4. 前视距离 (L_d)：最关键的参数

前视距离决定控制器“看多远”。

| (L_d) 太小             | (L_d) 合适   | (L_d) 太大        |
| -------------------- | ---------- | --------------- |
| 紧贴路径、对噪声敏感、蛇形振荡、转向频繁 | 路径误差小且转向连续 | 轨迹被切弯、弯道外扩、反应太迟 |




## 5. 限幅、低速与倒车

### 5.1 转向限幅

任何情况下都必须限制：

$[$  
$\delta_{cmd}=\operatorname{clip}(\delta,-\delta_{max},\delta_{max})$  
$]$

不要只依赖下游驱动器限幅；上层限幅能避免规划异常将大量无效命令送进系统。

还应限制转向角速度：

$$
\dot{\delta}_{cmd} = \operatorname{clip}\left(
\frac{\delta_{raw} - \delta_{prev}}{\Delta t},
-\dot{\delta}_{max},\ \dot{\delta}_{max}
\right)
$$

随后积分得到新的转向命令。这对舵机 / EPS 的真实动态尤为重要。


### 5.2 低速处理

Pure Pursuit 本身不除以速度，因此低速仍可工作；但定位航向噪声在低速时常占主导。建议：

- 对 (|$v|<v_{min})$ 使用固定的较短 (L_d)；
    
- 车辆静止时保持当前转向或切换到专门的停车逻辑；
    
- 不要让目标点落在车辆后方。
## 6. 最小可用伪代码

```
def pure_pursuit(path, pose_map, speed_mps, last_s):
    # 1) 找最近弧长位置（搜索范围要受 last_s 约束）
    s_near = closest_projection_arc_length(path, pose_map.xy, last_s)

    # 2) 速度自适应前视距离
    ld = clamp(ld_min + ld_gain * abs(speed_mps), ld_min, ld_max)
    target_map = interpolate_path_at_arc_length(path, s_near + ld)

    # 3) 目标点 map -> base_link
    dx = target_map.x - pose_map.x
    dy = target_map.y - pose_map.y
    x_l =  cos(pose_map.yaw) * dx + sin(pose_map.yaw) * dy
    y_l = -sin(pose_map.yaw) * dx + cos(pose_map.yaw) * dy

    # 4) 防御式检查
    dist_sq = x_l * x_l + y_l * y_l
    if x_l <= 0.0 or dist_sq < 1e-6:
        return safe_fallback_command()

    # 5) 横向几何控制
    kappa = 2.0 * y_l / dist_sq
    delta_raw = atan(wheelbase_m * kappa)
    delta_cmd = clamp(delta_raw, -max_steer_rad, max_steer_rad)
    delta_cmd = rate_limit(delta_cmd, delta_previous, max_steer_rate, dt)

    return delta_cmd, target_map, s_near


```





# fsac 最小实现的工程问题：
## 世界 → 车辆：旋转 −yaw−yaw

车辆坐标系相对世界坐标系旋转了 yawyaw。要从世界坐标转到车辆坐标，需要反向旋转 −yaw−yaw：

$$\begin{bmatrix} x_v \\ y_v \end{bmatrix}
= R(-yaw)
\begin{bmatrix} x_w \\ y_w \end{bmatrix}$$



其中旋转矩阵：

$$R(\theta) =
\begin{bmatrix}
\cos\theta & -\sin\theta \\
\sin\theta & \cos\theta
\end{bmatrix}$$

### 代入 −yaw−yaw 的结果

latex

$$R(-yaw) =
\begin{bmatrix}
\cos(-yaw) & -\sin(-yaw) \\
\sin(-yaw) & \cos(-yaw)
\end{bmatrix}
=
\begin{bmatrix}
\cos(yaw) & \sin(yaw) \\
-\sin(yaw) & \cos(yaw)
\end{bmatrix}$$