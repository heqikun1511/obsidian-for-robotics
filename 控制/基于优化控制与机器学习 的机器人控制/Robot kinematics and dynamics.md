




# ROBOT MANIPULATOR

## 结构组成
- Set of **rigid bodies** connected by **joints**
  - **rigid bodies**（刚体） = **links**（连杆）
  - **joints**（关节）：
    - **revolute**（旋转关节）
    - **prismatic**（棱柱/移动关节）

## 驱动方式
- Typically joints are actuated by systems of
  **motor + transmission** (e.g. gear box)

## 图示与坐标定义
- **Base**（基座）
- **End-effector**（末端执行器）
- $q_i$：$i$-th joint coordinate（第 $i$ 个关节坐标）

## 配置向量
$$
q = \begin{bmatrix}
q_1 \\
q_2 \\
\vdots \\
q_n
\end{bmatrix}
\leftarrow \text{configuration（构型）}
$$


这里的机器人需要低速大扭矩，所以需要变速器


[[Homogenous]]