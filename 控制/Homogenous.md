
## 1. 为什么需要齐次变换

刚体在三维空间中的位姿包含两部分：

- 位置：刚体原点在哪里；
    
- 姿态：刚体坐标轴朝向哪里。
    

单独使用旋转矩阵只能表示旋转：

$$  
p'=Rp  
$$

若还存在平移，则需要：

$$  
p'=Rp+t  
$$

右侧同时包含矩阵乘法和向量加法，不便于把多个坐标变换统一连乘。齐次坐标给三维点增加第 4 个分量：

$$  
\[ \tilde p= \begin{bmatrix} p_x\\ p_y\\ p_z\\ 1 \end{bmatrix} \]
$$

于是旋转和平移可以写成一次矩阵乘法：

$$  
[ \tilde p'= \underbrace{ \begin{bmatrix} R&t\\ 0_{1\times3}&1 \end{bmatrix}}_{T} \tilde p ] 
$$

这就是齐次变换矩阵。

## 3. 旋转矩阵

三维旋转矩阵：

$$  
R\in SO(3)  
$$

其中：

$$  
SO(3)={R\in\mathbb{R}^{3\times3}\mid R^TR=I,\det(R)=1}  
$$

两个关键性质：

$$  
R^{-1}=R^T  
$$

$$  
\det(R)=1  
$$

第一条表示旋转矩阵的逆等于转置；第二条排除了镜像反射。

### 3.1 旋转矩阵每一列的意义

设：

$$  
{}^{A}R_B=  
\begin{bmatrix}  
|&|&|\  
{}^{A}\hat x_B&{}^{A}\hat y_B&{}^{A}\hat z_B\  
|&|&|  
\end{bmatrix}  
$$


那么三列分别是坐标系 $B$ 的 $x,y,z$ 单位轴在坐标系 $A$ 中的表达。

因此，${}^{A}R_B$ 描述了坐标系 $B$ 相对于坐标系 $A$ 的姿态，同时可以把一个向量从 $B$ 坐标表达变换到 $A$ 坐标表达：

$$  
{}^{A}v={}^{A}R_B{}^{B}v  
$$

记忆方式：

$$  
{}^{A}R_{\cancel B},{}^{\cancel B}v={}^{A}v  
$$

中间的 $B$ 可以像单位一样“消去”。


### 3.2 常用基本旋转矩阵

绕 $x$ 轴旋转 $\theta$：

$$[ R_x(\theta)= \begin{bmatrix} 1&0&0\\ 0&\cos\theta&-\sin\theta\\ 0&\sin\theta&\cos\theta \end{bmatrix} ]
$$

绕 $y$ 轴旋转 $\theta$：

$$  
R_y(\theta)=  
\begin{bmatrix}  
\cos\theta&0&\sin\theta\  
0&1&0\  
-\sin\theta&0&\cos\theta  
\end{bmatrix}  
$$

绕 $z$ 轴旋转 $\theta$：

$$  
R_z(\theta)=  
\begin{bmatrix}  
\cos\theta&-\sin\theta&0\  
\sin\theta&\cos\theta&0\  
0&0&1  
\end{bmatrix}  
$$

正方向遵循右手定则。

## 4. 齐次变换矩阵的结构

从坐标系 $B$ 到坐标系 $A$ 的齐次变换写作：

$$  
{}^{A}T_B=  
\begin{bmatrix}  
{}^{A}R_B&{}^{A}p_{AB}\  
0_{1\times3}&1  
\end{bmatrix}  
$$

各部分含义：

- ${}^{A}R_B$：坐标系 $B$ 相对于 $A$ 的姿态；
    
- ${}^{A}p_{AB}$：从 $A$ 原点指向 $B$ 原点的向量，用 $A$ 坐标表达；
    
- 最后一行 $[0\ 0\ 0\ 1]$：使平移能够纳入矩阵乘法。
    

它属于特殊欧氏群：
从坐标系 $B$ 到坐标系 $A$ 的齐次变换写作：

$$  
{}^{A}T_B=  
\begin{bmatrix}  
{}^{A}R_B&{}^{A}p_{AB}\  
0_{1\times3}&1  
\end{bmatrix}  
$$

各部分含义：

- ${}^{A}R_B$：坐标系 $B$ 相对于 $A$ 的姿态；
    
- ${}^{A}p_{AB}$：从 $A$ 原点指向 $B$ 原点的向量，用 $A$ 坐标表达；
    
- 最后一行 $[0\ 0\ 0\ 1]$：使平移能够纳入矩阵乘法。
    

## 5. 齐次点与齐次方向向量

### 5.1 点

三维点的齐次形式：

$$  
\tilde p=  
\begin{bmatrix}  
p\1  
\end{bmatrix}  
$$

变换后：
$$
\begin{bmatrix}  
Rp+t\1  
\end{bmatrix}  
$$

点受到旋转和平移共同影响。


## 6. 二维齐次变换

二维刚体位姿属于 $SE(2)$：

$$\[ T= \begin{bmatrix} \cos\theta&-\sin\theta&t_x\\ \sin\theta&\cos\theta&t_y\\ 0&0&1 \end{bmatrix} \]$$

二维点：

\$$[ \tilde p= \begin{bmatrix} x\\ y\\ 1 \end{bmatrix} \]$$

变换结果：

$$\[ \begin{aligned} x'&=\cos\theta\,x-\sin\theta\,y+t_x\\ y'&=\sin\theta\,x+\cos\theta\,y+t_y \end{aligned} \]$$

移动机器人平面定位中常使用 $(x,y,\theta)$，其矩阵形式就是 $SE(2)$ 齐次变换。

## 8. 两个齐次变换相乘的展开形式

设：

\$$[ T_1= \begin{bmatrix} R_1&t_1\\ 0&1 \end{bmatrix},\qquad T_2= \begin{bmatrix} R_2&t_2\\ 0&1 \end{bmatrix} \]$$

则：

\$$[ T_1T_2= \begin{bmatrix} R_1R_2&R_1t_2+t_1\\ 0&1 \end{bmatrix} \]$$

注意平移部分不是简单的 $t_1+t_2$，而是：

$$\[ t=R_1t_2+t_1 \]$$

原因是 $t_2$ 原本在第二个局部坐标系中表达，必须先经过 $R_1$ 旋转到共同坐标系，才能与 $t_1$ 相加。


## 9. 齐次变换的逆

若：

$$\[ {}^{A}T_B= \begin{bmatrix} R&t\\ 0&1 \end{bmatrix} \]$$

那么反向变换：

$$\[ {}^{B}T_A=({}^{A}T_B)^{-1} \]$$

其解析形式为：

$$\[ T^{-1}= \begin{bmatrix} R^T&-R^Tt\\ 0&1 \end{bmatrix} \]$$

不要错误地写成：

$$\[ \begin{bmatrix} R^T&-t\\ 0&1 \end{bmatrix} \]$$

平移向量也必须转换到反向坐标系，因此是 $-R^Tt$。

验证：

$$ TT^{-1}= \begin{bmatrix} RR^T&-RR^Tt+t\\ 0&1 \end{bmatrix} =I_4 $$



## 10. 手算示例：把传感器点变到车体坐标系

假设激光雷达坐标系为 $L$，车体坐标系为 $B$。雷达原点在车体坐标系中位于：

\[$$ {}^{B}p_{BL}= \begin{bmatrix} 1\\0\\0.5 \end{bmatrix}$$ \]

雷达相对于车体绕 $z$ 轴旋转 $90^\circ$：

\[ $${}^{B}R_L= \begin{bmatrix} 0&-1&0\\ 1&0&0\\ 0&0&1 \end{bmatrix}$$ \]

因此：

\[ $${}^{B}T_L= \begin{bmatrix} 0&-1&0&1\\ 1&0&0&0\\ 0&0&1&0.5\\ 0&0&0&1 \end{bmatrix} $$\]

雷达测得点：

\[ $${}^{L}\tilde p= \begin{bmatrix} 2\\0\\0\\1 \end{bmatrix}$$ \]

转换到车体坐标系：

\[ $${}^{B}\tilde p={}^{B}T_L{}^{L}\tilde p= \begin{bmatrix}1\\2\\0.5\\1\end{bmatrix} $$\]

解释：雷达自身前方 2 m 的点，因为雷达相对车体旋转了 $90^\circ$，在车体坐标中出现在左侧 2 m；再加上传感器安装位置偏移。

## 11. 主动变换与被动变换

相同的旋转矩阵可能出现在两种语境中。

### 11.1 主动变换

坐标系固定，让几何对象真正发生旋转或移动：

\[ $$p'=Rp$$ \]

### 11.2 被动变换

几何对象不动，仅改变描述它的参考坐标系：

\[$$ {}^{A}p={}^{A}R_B{}^{B}p$$ \]

主动旋转一个向量 $R$，等价于保持向量不动、反向旋转坐标系。因此两种描述常相差一个逆或转置。

工程中看到公式方向不一致时，先检查作者讨论的是：

- 移动物体；还是
- 更换坐标表达。

不要只比较矩阵表面形式。


## 齐次变换与 ROS TF2

ROS TF2 维护的是一棵随时间变化的坐标系变换树，例如：

```
map
└── odom
    └── base_link
        ├── lidar_link
        └── camera_link
```

若希望把 `lidar_link` 中的点转换到 `base_link`，需要：

\[$$ {}^{base}T_{lidar}$$ \]


## 练习题与答案

### 练习 1：判断变换方向

已知：

\[$$ {}^{map}T_{base} $$\]

它能直接把哪个坐标系中的点转换到哪个坐标系？

**答案：**

\[$$ {}^{map}p={}^{map}T_{base}{}^{base}p$$ \]

把 `base` 中表达的点转换成 `map` 中的表达。

### 练习 2：组合变换

已知 ${}^{map}T_{base}$ 和 ${}^{base}T_{lidar}$，求雷达点到地图坐标的变换。

**答案：**

\[ $${}^{map}T_{lidar} ={}^{map}T_{base}{}^{base}T_{lidar} $$\]\[$$ {}^{map}p ={}^{map}T_{lidar}{}^{lidar}p$$ \]

### 练习 3：求逆

\[ $$T= \begin{bmatrix} 0&-1&0&1\\ 1&0&0&2\\ 0&0&1&0\\ 0&0&0&1 \end{bmatrix} $$\]

求 $T^{-1}$。

**答案：**

\[ $$R^T= \begin{bmatrix} 0&1&0\\ -1&0&0\\ 0&0&1 \end{bmatrix} $$\]\[ $$-R^Tt=\begin{bmatrix}-2\\1\\0\end{bmatrix}$$ \]

所以：

\[ T^{-1}= \begin{bmatrix} 0&1&0&-2\\ -1&0&0&1\\ 0&0&1&0\\ 0&0&0&1 \end{bmatrix} \]

### 练习 4：点与方向

同一个 $T$ 分别作用于 $1,0,0,1^T$ 和 $1,0,0,0^T$，区别是什么？

**答案：** 前者是点，会同时旋转和平移；后者是方向，只旋转，不受平移影响。

### 练习 5：检查坐标链

下面哪个乘法有明确的坐标变换意义？

\[ {}^{A}T_B{}^{C}T_D \]\[ {}^{A}T_B{}^{B}T_D \]

**答案：** 第二个。相邻的 $B$ 可以消去，结果为 ${}^{A}T_D$。第一个缺少把 $C$ 与 $B$ 联系起来的变换。

---

## 21. 最终记忆框架

### 核心矩阵

\[ {}^{A}T_B= \begin{bmatrix} {}^{A}R_B&{}^{A}p_{AB}\\ 0&1 \end{bmatrix} \]

### 坐标转换

\[ {}^{A}p={}^{A}T_B{}^{B}p \]

### 变换组合

\[ {}^{A}T_C={}^{A}T_B{}^{B}T_C \]

### 逆变换

\[ T^{-1}= \begin{bmatrix} R^T&-R^Tt\\ 0&1 \end{bmatrix} \]