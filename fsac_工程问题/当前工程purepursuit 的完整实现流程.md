[[[Purepursuit]]


# 第一步：验证输入

Pure Pursuit 首先检查：

[pure_pursuit.cpp (line 57)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/lateral_Pure_Pursuit/src/pure_pursuit.cpp:57)

- 参数是否合法；
- 路径至少有两个点；
- 车辆位置是否为有限值；
- 航向是否为有限值；
- 速度是否为有限值；
- 所有路径点是否为有限值。

任何一点包含：

```
NaN
+Inf
-Inf
```

算法都会返回：

```
valid = false;
```

控制节点随后：

```
longitudinal_pid_.reset();
publish_stop();
```

因此 Pure Pursuit 计算失败不仅会把转向置零，还会让纵向控制一起停车。

---

# 5. 第二步：计算动态前视距离

代码：

[pure_pursuit.cpp (line 71)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/lateral_Pure_Pursuit/src/pure_pursuit.cpp:71)

公式：

\[ L_d = \operatorname{clamp} \left( L_{\min}+K_v|v|, L_{\min}, L_{\max} \right) \]

当前参数：

\[ L_d = \operatorname{clamp} \left( 2.0+0.6|v|, 2.0, 8.0 \right) \]

不同速度下：

|速度|计算前视距离|
|---|---|
|0 m/s|2.0 m|
|1 m/s|2.6 m|
|2 m/s|3.2 m|
|4 m/s|4.4 m|
|5 m/s|5.0 m|
|8 m/s|6.8 m|
|10 m/s|8.0 m|
|15 m/s|仍为 8.0 m|

这里使用了：

```
std::abs(vehicle.speed_mps)
```

因此倒车时也会根据速度绝对值增大前视距离。不过当前算法本身并没有专门实现倒车 Pure Pursuit。

## 为什么速度越快，前视距离越长

低速时使用短前视距离：

```
优点：更贴合路径
缺点：方向变化比较敏感
```

高速时使用长前视距离：

```
优点：转向更平滑、更稳定
缺点：可能切弯、跟踪精度降低
```

这相当于在：

```
跟踪精度
和
高速稳定性
```

之间做动态折中。

---

# 6. 第三步：选择前视目标点

算法从路径第一个点开始扫描：

[pure_pursuit.cpp (line 76)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/lateral_Pure_Pursuit/src/pure_pursuit.cpp:76)

对于每个点，计算它到车辆的欧氏距离平方：

\[ d_i^2 = (x_i-x_v)^2+(y_i-y_v)^2 \]

找到第一个满足：

\[ d_i\geq L_d \]

的路径点，将它作为前视目标点。

伪代码：
代码：

```
selected = path.back();

for each point in path:
  if distance(vehicle, point) >= desired_lookahead:
    selected = point;
    break;
```

如果整条局部路径都没有点达到期望前视距离，就使用最后一个路径点。

## 当前实现的关键特点

它不是：

- 从圆和路径求精确交点；
- 按路径弧长寻找前视点；
- 在两个路径点之间插值；
- 先寻找离车辆最近点再向前推进。

它只是：

> 从 Path 数组头部开始，选择第一个“欧氏距离大于前视距离”的离散点。

所以它依赖 Planning 保证：

1. 路径点按照车辆前进方向排序；
2. Path 开头靠近车辆；
3. 不应该先出现距离较远的车辆后方点；
4. 路径点间距不能过大；
5. 交叉赛道附近不能混入另一段路线。

---

# 7. 第四步：把目标点转换到车辆坐标系

对于目标点：

$\[ \Delta x=x_t-x_v \]\[ \Delta y=y_t-y_v \]$

然后计算目标点在车辆横向方向上的坐标：

$\[ y_L = -\sin(\psi)\Delta x+\cos(\psi)\Delta y \]$

代码：

[pure_pursuit.cpp (line 88)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/lateral_Pure_Pursuit/src/pure_pursuit.cpp:88)

这里的 \(y_L\) 就是代码里的：

```
lateral_error
```

它并不等于传统意义上的“车辆到整条路径的最近横向误差”，而是：

> 前视目标点在车辆坐标系中的横向坐标。

## 当前仿真的简化

当前路径已经在 `base_footprint` 下，并且传入：

```
vehicle.x   = 0
vehicle.y   = 0
vehicle.yaw = 0
```

所以公式退化为：

$\[ \Delta x=x_t \]\[ \Delta y=y_t \]\[ y_L=y_t \]$

即当前实现中：


# 第五步：计算目标曲率

Pure Pursuit 假设车辆沿一个圆弧，从当前位置驶向前视目标点。

目标点距离：

\[ L_{\text{actual}}^2 = \Delta x^2+\Delta y^2 \]

路径曲率：

\[ \kappa = \frac{2y_L}{L_{\text{actual}}^2} \]

对应代码：

[pure_pursuit.cpp (line 90)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/lateral_Pure_Pursuit/src/pure_pursuit.cpp:90)

注意这里分母使用的是：

```
实际选中点的距离平方
```

不是前面计算出的理论前视距离平方。

因为当前选的是离散路径点，实际距离可能不是严格的 `Ld`。

## 曲率符号

```
目标点在左侧：
  y_L > 0
  κ > 0
  左转

目标点在右侧：
  y_L < 0
  κ < 0
  右转

目标点正前方：
  y_L = 0
  κ = 0
  直行
```

---

# 9. 第六步：曲率转换为转向角

Pure Pursuit 使用运动学自行车模型：

$\[ \kappa=\frac{\tan\delta}{L} \]$

其中：

- \(\kappa\)：车辆路径曲率；
- \(\delta\)：等效前轮转向角；
- \(L\)：车辆轴距。

反解：

$\[ \delta = \arctan(L\kappa) \]$

代码：

[pure_pursuit.cpp (line 97)](/home/amoy/fsac_driveless/driverless_workspace/workspace/autonomous_system/stacks/rt_control/src/lateral_Pure_Pursuit/src/pure_pursuit.cpp:97)

当前轴距：

\[ L=1.6m \]

然后限制：

$\[ \delta_{\text{output}} = \operatorname{clamp} (\delta,-0.45,0.45) \]$



