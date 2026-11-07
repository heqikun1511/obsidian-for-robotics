[[强化学习与最优控制


第四讲把第三讲的 Euler–Lagrange 方程扩展到更实际的最优控制问题：终点和终止时间不一定固定，还要加入动力学约束。最终得到 PMP 的基本骨架。[stanfordasl.github.io](https://stanfordasl.github.io//aa203/sp2526/pdfs/lecture/lecture_4.pdf?utm_source=chatgpt.com)

## 1. 终端条件不固定时：为什么会多出边界条件

第三讲默认：

$\[ t_f\text{ 固定},\qquad x(t_0),x(t_f)\text{ 固定}. \]$

所以端点扰动为零：

$\[ \delta x(t_0)=\delta x(t_f)=0. \]$

第四讲放宽这一设定：

- 初始时间、初始状态仍固定；
- 终止时间 \(t_f\) 可固定或自由；
- 终端状态 \(x(t_f)\) 的某些分量可固定或自由。

无论端点是否自由，内部轨迹都仍要满足 Euler–Lagrange 方程：

$\[ \boxed{ g_x-\frac{d}{dt}g_{\dot{x}}=0 } \]$

差别在于：分部积分产生的端点项不再必然消失，因此它决定了额外的横截条件（transversality conditions）。
## 最重要的终端条件结论

考虑

$\[ J= h(x(t_f),t_f) + \int_{t_0}^{t_f}g(x,\dot{x},t)\,dt. \]$

记

$\[ p(t)=g_{\dot{x}}(x^\ast,\dot{x}^\ast,t). \]$

终端的一般条件可紧凑理解为：

$\[ \boxed{ (h_x-p(t_f))^\top\delta x_f + \left(H(t_f)+h_t\right)\delta t_f=0 } \]$

其中

$\[ H=g+p^\top\dot{x} \]$

或在最优控制形式中写成 $\(H=g+p^\top f\)。$


常见情况：

| 终端条件               | 必须满足的条件                                              |
| ------------------ | ---------------------------------------------------- |
| \(x_i(t_f)\) 固定    | $\(\delta x_i(t_f)=0\)，不额外产生该状态分量条件$                   |
| \(x_i(t_f)\) 自由    | $\(\boxed{p_i(t_f)=\frac{\partial h}{\partial x_i}}\)$ |
| \(t_f\) 固定         | $\(\delta t_f=0\)$                                     |
| \(t_f\) 自由         | $\(\boxed{H(t_f)+\frac{\partial h}{\partial t_f}=0}\)$ |
| 无终端代价 \(h=0\)，终态自由 | $\(\boxed{p(t_f)=0}\)$                                 |
| 无终端代价、终止时间自由       | $\(\boxed{H(t_f)=0}\)$                                 |

对你之后做赛车轨迹优化尤其重要：是否固定终点状态、是否允许自由终止时间，会直接改变待求边界条件。