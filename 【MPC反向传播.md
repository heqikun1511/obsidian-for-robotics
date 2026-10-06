

**这是 MPC 反向传播的一部分：计算代价函数对某个控制量的梯度贡献——`tracking` 项（跟踪误差）通过"敏感度"（控制量对状态的影响）传递到梯度上。**

## 一、逐部分拆解

`cpp`

`{`
    `const std::size_t distance = prediction - control;`
    `const double sensitivity =`
        `input_gain * std::pow(config_.servo_pole, distance);`
    `value +=`
        `2.0 * config_.tracking_weight *`
        `(state[prediction + 1U] - reference_rad[joint][prediction]) *`
        `sensitivity;`
`}`

|部分|含义|
|---|---|
|`prediction`|预测步（受影响的未来时刻）|
|`control`|控制步（当前算梯度的控制量）|
|`distance = prediction - control`|控制步到预测步的"距离"|
|`sensitivity`|**敏感度**——控制量对预测状态的影响|
|`value`|累积的梯度值|
|`state[prediction+1] - ref[...]`|**跟踪误差**|
|`2.0 * tracking_weight * ...`|代价函数导数的系数|



## 二、数学本质：链式法则

### 代价函数

MPC 的跟踪代价：

$J=∑k=0N−1w⋅(xk+1−rk)2J=k=0∑N−1​w⋅(xk+1​−rk​)2$

- ww = `tracking_weight`
    
- xk+1xk+1​ = 预测状态
    
- rkrk​ = 参考

### 对控制量的梯度

∂J∂uj=∑k=0N−12w⋅(xk+1−rk)⋅∂xk+1∂uj∂uj​∂J​=k=0∑N−1​2w⋅(xk+1​−rk​)⋅∂uj​∂xk+1​​

**关键**：ujuj​ 不只影响 xj+1xj+1​，还**间接影响所有后续状态** xj+2,xj+3,…xj+2​,xj+3​,…（通过状态递推）。

**这就是为什么需要"敏感度"**。
一阶模型：

xk+1=a⋅xk+b⋅ukxk+1​=a⋅xk​+b⋅uk​

其中 a=servo_polea=servo_pole，b=input_gainb=input_gain。

**ujuj​ 对 xk+1xk+1​ 的直接影响**：

∂xj+1∂uj=b∂uj​∂xj+1​​=b

**ujuj​ 对 xj+2xj+2​ 的影响**（通过 xj+1xj+1​）：

∂xj+2∂uj=∂xj+2∂xj+1⋅∂xj+1∂uj=a⋅b∂uj​∂xj+2​​=∂xj+1​∂xj+2​​⋅∂uj​∂xj+1​​=a⋅b

**ujuj​ 对 xj+3xj+3​ 的影响**：

∂xj+3∂uj=a2⋅b∂uj​∂xj+3​​=a2⋅b

**一般规律**：

∂xk+1∂uj=b⋅ak−j=input_gain⋅servo_polek−j∂uj​∂xk+1​​=b⋅ak−j=input_gain⋅servo_polek−j

**这正是代码里的 `sensitivity`**：

sensitivity=input_gain⋅servo_poledistancesensitivity=input_gain⋅servo_poledistance

其中 `distance = prediction - control`。
