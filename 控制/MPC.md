---
tags:
  - 控制原理
  - 最优控制
---
## 1. MPC 到底在做什么

一句话概括：

> MPC 使用系统模型预测未来，在每个采样时刻求解一次“未来一段时间内怎样控制最好”的优化问题，但只执行算出的第一个控制量，然后重新测量、重新预测、重新优化。


例如控制小车停车：

- 只看当前误差，可能临近目标时才开始刹车；
    
- MPC 会预测“按当前速度继续走，未来会越过目标”，因此提前减速；
    
- 如果还规定电机电流不能超过 10 A、速度不能超过 2 m/s，MPC 可以把这些限制直接写入优化问题。
    

MPC 的三个核心词：

1. **模型**：描述控制输入怎样改变系统状态。
    
2. **预测**：根据当前状态计算未来若干步的状态。
    
3. **优化**：从许多候选控制序列中选择代价最小且满足约束的一组。
    

---

## 2. 先理解反馈控制

设系统希望达到参考值 $r$，传感器测得输出 $y$，误差为

$$  
e=r-y  
$$

控制器根据误差产生控制量 $u$。但是控制器真正面对的不只是当前误差，还包括：

- 系统的惯性；
    
- 执行器饱和；
    
- 状态和输入的安全范围；
    
- 多个目标之间的权衡；
    
- 扰动和建模误差。
    

PID 主要根据当前和过去的误差控制。LQR 根据当前状态计算最优线性反馈。MPC 在此基础上再向未来预测，并显式处理约束。


| 控制器 | 是否用模型 | 是否预测未来 | 是否原生处理约束 | 在线计算量 |
| --- | ----- | ------ | -------- | ----- |
| PID | 否     | 否      | 否        | 很小    |
| LQR | 是     | 否      | 否        | 很小    |
| MPC | 是     | 是      | 是        | 较大    |

## 3. MPC 使用的系统模型

### 3.1 状态、输入和输出

离散线性状态空间模型为

$$  
x_{k+1}=Ax_k+Bu_k  
$$

$$  
y_k=Cx_k+Du_k  
$$

符号含义：

- $k$：第 $k$ 个采样时刻；
    
- $x_k\in\mathbb{R}^n$：状态，如位置、速度、关节角和角速度；
    
- $u_k\in\mathbb{R}^m$：控制输入，如电压、电流、力矩或目标速度；
    
- $y_k\in\mathbb{R}^p$：可测或关注的输出；
    
- $A$：状态在一个采样周期内如何自行变化；
    
- $B$：控制输入如何影响状态；
    
- $C$：从状态中得到输出；
    
- $D$：输入对输出的直接影响，很多机械系统中 $D=0$。
    

状态不是“所有传感器数据”，而是足以描述系统未来演化的一组变量。




### .2 一个最简单的位置—速度模型

若状态为位置 $p$ 和速度 $v$，输入为加速度 $a$：

$$  
x_k=\begin{bmatrix}p_k\v_k\end{bmatrix},\qquad u_k=a_k  
$$

采样周期为 $T_s$，使用匀加速模型：
$$
\begin{bmatrix}1&T_s\0&1\end{bmatrix}  
\begin{bmatrix}p_k\v_k\end{bmatrix}  
+  
\begin{bmatrix}\frac12T_s^2\T_s\end{bmatrix}u_k  
$$

因此

  $$
A=\begin{bmatrix}1&T_s\0&1\end{bmatrix},\qquad  
B=\begin{bmatrix}\frac12T_s^2\T_s\end{bmatrix}  
$$


### 3.3 连续模型怎样离散化

若连续模型是

$$  
\dot{x}=A_cx+B_cu  
$$

最简单的前向欧拉离散化为

$$  
x_{k+1}\approx x_k+T_s(A_cx_k+B_cu_k)  
$$

所以

$$  
A_d=I+T_sA_c,\qquad B_d=T_sB_c  
$$

更准确的零阶保持离散化为

$$  
A_d=e^{A_cT_s}  
$$

$$  
B_d=\int_0^{T_s}e^{A_c\tau}B_c,d\tau  
$$

工程中可以用 `scipy.signal.cont2discrete` 或 MATLAB `c2d` 计算。

---

## 4. MPC 怎样预测未来

已知当前状态 $x_k$，预测一步：

$$  
x_{k+1|k}=Ax_k+Bu_{k|k}  
$$

其中 $x_{k+1|k}$ 表示“站在时刻 $k$ 对 $k+1$ 时刻的预测”。

预测两步：

$$  
\begin{aligned}  
x_{k+2|k}  
&=Ax_{k+1|k}+Bu_{k+1|k}\  
&=A^2x_k+ABu_{k|k}+Bu_{k+1|k}  
\end{aligned}  
$$

预测三步：

$$  
x_{k+3|k}=A^3x_k+A^2Bu_{k|k}+ABu_{k+1|k}+Bu_{k+2|k}  
$$

将未来 $N$ 步状态和输入堆叠：

$$  
X=  
\begin{bmatrix}  
x_{k+1|k}\x_{k+2|k}\\vdots\x_{k+N|k}  
\end{bmatrix},\qquad  
U=  
\begin{bmatrix}  
u_{k|k}\u_{k+1|k}\\vdots\u_{k+N-1|k}  
\end{bmatrix}  
$$

可写成紧凑形式：

$$  
X=\mathcal A x_k+\mathcal B U  
$$

其中

$$  
\mathcal A=  
\begin{bmatrix}  
A\A^2\\vdots\A^N  
\end{bmatrix}  
$$

$$  
\mathcal B=  
\begin{bmatrix}  
B&0&\cdots&0\  
AB&B&\cdots&0\  
\vdots&\vdots&\ddots&\vdots\  
A^{N-1}B&A^{N-2}B&\cdots&B  
\end{bmatrix}  
$$

这两个矩阵把“未来状态”完全表示成“当前状态 + 未来控制序列”。这一步是线性 MPC 的核心。

---


## 5. 预测时域和控制时域

### 5.1 预测时域 $N_p$

$N_p$ 表示向未来预测多少个采样点。若 $T_s=0.02\text{ s}$，$N_p=20$，则预测未来

$$  
T_{preview}=N_pT_s=0.4\text{ s}  
$$

预测时间太短，控制器看不见较远的后果；太长则变量增多、计算变慢，而且远期预测受模型误差影响更大。

### 5.2 控制时域 $N_c$

$N_c$ 表示允许独立优化多少步控制量，通常 $N_c\le N_p$。超过 $N_c$ 后可令输入保持不变：

$$  
u_{k+N_c|k}=u_{k+N_c-1|k}  
$$

缩短控制时域可以减少优化变量，但自由度也会降低。初学时可直接令 $N_c=N_p$。



## 6. 目标函数：怎样定义“最好”

### 6.1 跟踪误差和控制代价

最常用的二次型目标函数为

$$  
J=\sum_{i=0}^{N-1}  
\left(  
|x_{k+i|k}-x^{ref}_{k+i}|_Q^2+  
|u_{k+i|k}-u^{ref}_{k+i}|_R^2  
\right)  
+|x_{k+N|k}-x^{ref}_{k+N}|_P^2  
$$

其中

$$  
|z|_Q^2=z^TQz  
$$

- $Q\succeq0$：状态误差权重；
    
- $R\succ0$：控制输入权重；
    
- $P\succeq0$：终端状态权重；
    
- $x^{ref}$：参考状态；
    
- $u^{ref}$：参考输入，调节到原点时通常取 0。
    

直观上：

- 增大某个 $Q_{ii}$：更重视第 $i$ 个状态的跟踪精度；
    
- 增大 $R$：减少控制动作，响应通常更温和、更慢；
    
- 增大 $P$：更在意预测终点是否接近目标。
    

权重的绝对数值不是唯一的，状态量的尺度和各权重的相对比例更重要。



### 6.2 输入变化率代价

真实执行器不适合频繁突变，常加入

$$  
\Delta u_k=u_k-u_{k-1}  
$$

以及

$$  
J_{\Delta u}=\sum_{i=0}^{N-1}|\Delta u_{k+i|k}|_S^2  
$$

增大 $S$ 会让控制命令更平滑，但也会减慢响应。

### 6.3 输出跟踪形式

若只关心输出而不是所有状态，可以写成

$$  
J=\sum_{i=1}^{N}|y_{k+i|k}-r_{k+i}|_{Q_y}^2  
+\sum_{i=0}^{N-1}|u_{k+i|k}|_R^2  
$$

由于 $y=Cx+Du$，仍可转换为关于 $U$ 的优化问题。



## 7. 从目标函数得到二次规划 QP

把参考轨迹堆叠成 $X_{ref}$，并定义块对角矩阵

$$  
\bar Q=\operatorname{diag}(Q,Q,\ldots,Q,P)  
$$

$$  
\bar R=\operatorname{diag}(R,R,\ldots,R)  
$$

代价函数可写成

$$  
J=(X-X_{ref})^T\bar Q(X-X_{ref})+U^T\bar R U  
$$

代入

$$  
X=\mathcal A x_k+\mathcal B U  
$$

得到

$$  
J=(\mathcal A x_k+\mathcal B U-X_{ref})^T  
\bar Q(\mathcal A x_k+\mathcal B U-X_{ref})+U^T\bar R U  
$$

展开并去掉与 $U$ 无关的常数项：

$$  
J=U^T(\mathcal B^T\bar Q\mathcal B+\bar R)U  
+2(\mathcal A x_k-X_{ref})^T\bar Q\mathcal B U+\text{常数}  
$$

标准 QP 通常写成

$$  
\min_U\quad \frac12U^THU+f^TU  
$$

因此可取

$$  
H=2(\mathcal B^T\bar Q\mathcal B+\bar R)  
$$

$$  
f=2\mathcal B^T\bar Q(\mathcal A x_k-X_{ref})  
$$

不同求解器对前面的 $\frac12$ 定义可能不同，使用前必须检查接口文档。

只要模型线性、代价为凸二次型、约束为线性不等式，这就是凸二次规划；在可行时通常能够稳定地求得全局最优解。****

## 8. 约束：MPC 最有价值的部分

### 8.1 输入约束

执行器幅值限制：

$$  
u_{min}\le u_{k+i|k}\le u_{max}  
$$

输入变化率限制：

$$  
\Delta u_{min}\le u_{k+i|k}-u_{k+i-1|k}\le\Delta u_{max}  
$$

### 8.2 状态和输出约束

$$  
x_{min}\le x_{k+i|k}\le x_{max}  
$$

$$  
y_{min}\le y_{k+i|k}\le y_{max}  
$$

对机器人可以分别表示：

- 关节角限位；
    
- 最大关节速度；
    
- 最大力矩或电流；
    
- 机身姿态范围；
    
- 与障碍物的最小距离。
    

将 $X=\mathcal A x_k+\mathcal B U$ 代入状态约束，就能整理为

$$  
GU\le h+Ex_k  
$$

于是完整优化问题为

$$  
\begin{aligned}  
\min_U\quad &\frac12U^THU+f^TU\  
\text{s.t.}\quad &GU\le h+Ex_k  
\end{aligned}  
$$



### 8.3 硬约束和软约束

**硬约束**必须满足，例如机械极限、电流绝对上限。违反硬约束就意味着优化问题不可接受。

**软约束**允许短时违反，但要引入松弛变量 $\epsilon\ge0$：

$$  
y\le y_{max}+\epsilon  
$$

并在代价中加入

$$  
\rho_1\epsilon+\rho_2\epsilon^2  
$$

软约束可降低“无可行解”的风险。安全极限一般仍设为硬约束，舒适性、普通跟踪边界可以设为软约束。


## 9. 滚动时域控制

在时刻 $k$，优化器算出

$$  
U^_=\begin{bmatrix}  
u^{k|k}&u^*_{k+1|k}&\cdots&u^*_{k+N-1|k}  
\end{bmatrix}^T  
$$

但控制器只执行第一项：

$$  
u_k=u^*_{k|k}  
$$

到 $k+1$ 时刻重新测量或估计 $x_{k+1}$，再求一次新的最优序列。

这样做的原因是：

- 模型不可能完全准确；
    
- 系统会受到外部扰动；
    
- 新的测量可以修正之前的预测；
    
- 参考轨迹和约束可能发生变化。
    

因此 MPC 虽然使用开环预测，却通过每一步重算形成闭环控制。

## 10. 一个完整的控制周期

每个采样周期依次执行：

1. 读取传感器。
    
2. 用观测器估计完整状态 $\hat x_k$。
    
3. 更新未来参考轨迹和约束。
    
4. 根据 $\hat x_k$ 构造 QP 的 $f$ 和约束右端项。
    
5. 调用求解器得到 $U^*$。
    
6. 检查求解状态、计算时间和结果范围。
    
7. 只发送第一个控制量 $u^*_{k|k}$。
    
8. 保存本次解，用作下一次求解的热启动。
    
9. 等待下一个周期，重复以上过程。


伪代码：

```
initialize model, weights, constraints, solver
u_last = safe_input

every Ts seconds:
    measurements = read_sensors()
    x_hat = state_estimator(measurements, u_last)
    reference = build_reference_trajectory()
    update_qp(x_hat, reference, limits)
    status, U = solve_qp(warm_start=True)

    if status is optimal and solve_time < deadline:
        u_cmd = first_element(U)
    else:
        u_cmd = fallback_controller(x_hat)

    u_cmd = final_safety_clamp(u_cmd)
    send_to_actuator(u_cmd)
    u_last = u_cmd
```         


## 11. 为什么需要状态估计器

MPC 通常需要 $x_k$，但传感器不一定能测到全部状态。例如只测到关节角，角速度可能要估计。

常用方法：

- 差分加低通滤波：简单但噪声较大；
    
- Luenberger 观测器：适合线性确定性模型；
    
- 卡尔曼滤波器：显式考虑过程噪声和测量噪声；
    
- EKF/UKF：适合非线性模型。[[kalman与EKF]]
    

控制器使用的是 $\hat x_k$：

$$  
x_{k+1}=Ax_k+Bu_k+w_k  
$$

$$  
y_k=Cx_k+v_k  
$$

其中 $w_k$ 是过程噪声，$v_k$ 是测量噪声。

观测器不应被误解为 MPC 的一部分；它负责回答“系统现在在哪里”，MPC 负责回答“接下来怎样控制”。

## 12. 稳态误差与扰动模型

如果模型认为维持目标只需 $u=0$，但真实系统存在重力、摩擦或恒定负载，普通 MPC 可能出现稳态误差。

一种方法是增加积分状态：

$$  
z_{k+1}=z_k+(r_k-y_k)  
$$

将增广状态定义为

$$  
x_k^a=\begin{bmatrix}x_k\z_k\end{bmatrix}  
$$

另一种方法是加入常值扰动模型：

$$  
x_{k+1}=Ax_k+Bu_k+B_dd_k  
$$

$$  
d_{k+1}=d_k  
$$

通过观测器估计 $d_k$，再让 MPC 补偿它。这类方法称为无偏差 MPC（offset-free MPC）。




## MPC 的主要类型

### 15.1 线性 MPC

使用线性模型、二次目标和线性约束，一般转化成 QP。成熟、可解释、适合实时实现，是工程入门首选。

### 15.2 非线性 MPC（NMPC）

使用

$$  
x_{k+1}=f(x_k,u_k)  
$$

能够描述复杂非线性，但在线求解的是非线性规划，计算量更大，并且更依赖初值。

### 15.3 线性时变 MPC

每个周期在当前轨迹附近线性化：

$$  
\delta x_{k+1}=A_k\delta x_k+B_k\delta u_k  
$$

它位于固定线性 MPC 与 NMPC 之间，自动驾驶轨迹跟踪中很常见。

### 15.4 鲁棒 MPC

显式考虑模型误差和有界扰动。Tube MPC 会让名义状态沿中心轨迹运动，同时使用局部反馈把真实状态限制在一条“管道”内。

### 15.5 随机 MPC

将不确定性写成概率分布，并使用机会约束，例如：

$$  
\Pr(x_k\le x_{max})\ge0.99  
$$

适用于允许小概率违约且具有可靠统计模型的场景。






## 最小 Python 示例

下面用 CVXPY 表示一个位置—速度系统。此代码强调结构，工程使用时还要增加状态估计、日志、超时和安全降级。

```
import cvxpy as cp
import numpy as np

Ts = 0.1
A = np.array([[1.0, Ts],
              [0.0, 1.0]])
B = np.array([[0.5 * Ts**2],
              [Ts]])

nx, nu = 2, 1
N = 20

Q = np.diag([20.0, 2.0])
R = np.diag([0.2])
P = Q.copy()

x = cp.Variable((nx, N + 1))
u = cp.Variable((nu, N))
x0 = cp.Parameter(nx)
x_ref = cp.Parameter((nx, N + 1))

cost = 0
constraints = [x[:, 0] == x0]

for i in range(N):
    cost += cp.quad_form(x[:, i] - x_ref[:, i], Q)
    cost += cp.quad_form(u[:, i], R)
    constraints += [x[:, i + 1] == A @ x[:, i] + B @ u[:, i]]
    constraints += [cp.abs(u[:, i]) <= 2.0]
    constraints += [cp.abs(x[1, i]) <= 1.5]

cost += cp.quad_form(x[:, N] - x_ref[:, N], P)

problem = cp.Problem(cp.Minimize(cost), constraints)

state = np.array([0.0, 0.0])
reference = np.zeros((nx, N + 1))
reference[0, :] = 1.0

x0.value = state
x_ref.value = reference
problem.solve(solver=cp.OSQP, warm_start=True)

if problem.status in ("optimal", "optimal_inaccurate"):
    u_now = float(u.value[0, 0])
else:
    u_now = 0.0

print("本周期控制量:", u_now)
```

注意：仿真循环中每次只能执行 `u.value[:, 0]`，然后更新真实/仿真状态并重新求解。

---


## 蛇形机器人的 MPC 建模

你的系统由树莓派负责较高层计算、STM32F407 负责实时通信和执行，控制八个串行总线舵机。最初不建议直接建立包含接触力的整机 NMPC，应分阶段实现。

### 20.1 第一阶段：关节轨迹跟踪 MPC

对第 $i$ 个关节，可使用简化状态：

$$  
x_i=\begin{bmatrix}\theta_i\\dot\theta_i\end{bmatrix}  
$$

八个关节堆叠为

$$  
x=\begin{bmatrix}  
\theta_1&\dot\theta_1&\theta_2&\dot\theta_2&\cdots&\theta_8&\dot\theta_8  
\end{bmatrix}^T  
$$

如果舵机接口只能接收目标角度，而不能直接接收电流或力矩，则外层 MPC 的输入可定义为：

- 目标角速度；或
    
- 下一周期目标角度增量 $\Delta\theta^{cmd}$。
    

此时 MPC 并不是直接控制舵机电机电流，而是生成满足速度、角度和变化率约束的平滑目标指令。

### 行波参考轨迹

蛇形机器人的基础参考角可写成

$$  
\theta_i^{ref}(t)=A\sin(\omega t+i\phi)+b  
$$

- $A$：摆动幅值；
    
- $\omega=2\pi f$：角频率；
    
- $\phi$：相邻关节相位差；
    
- $b$：偏置，可用于转弯。
    

MPC 不是一定要替代行波生成器。更清晰的分层方式是：

1. 步态生成器产生未来一段时间的 $\theta^{ref}$；
    
2. MPC 根据关节状态、舵机约束和耦合目标修正参考；
    
3. STM32 将第一步目标通过串行总线发送给舵机。





# 扩展：
[[投影梯度法]]】
[[【MPC反向传播]]】

