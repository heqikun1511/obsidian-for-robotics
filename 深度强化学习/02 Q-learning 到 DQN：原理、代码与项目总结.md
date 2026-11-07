# 从 Q-learning 到 DQN：原理、代码与项目总结

> [!info] 阅读范围
> 本笔记衔接 Hugging Face Deep RL Course 的 [Unit 2：Q-learning](https://huggingface.co/learn/deep-rl-course/en/unit2/introduction) 和 [Unit 3：从 Q-learning 到 DQN](https://huggingface.co/learn/deep-rl-course/en/unit3/from-q-to-dqn)。重点是上一单元的表格型 Q-learning 原理与完整代码，以及本单元 Space Invaders 项目的实现流程和复盘。本文代码为依据课程原理整理的独立示例；文中的项目分数要求来自课程，不代表已获得实验成绩。

[[QLearning]] · [[深度强化学习]]

## 目录

1. [从交互到动作价值](#1-从交互到动作价值)
2. [表格型 Q-learning 的原理](#2-表格型-q-learning-的原理)
3. [手算一次 Q 值更新](#3-手算一次-q-值更新)
4. [Unit 2：完整程序代码](#4-unit-2完整程序代码)
5. [代码逐段解释与实验设计](#5-代码逐段解释与实验设计)
6. [从 Q 表到 DQN](#6-从-q-表到-dqn)
7. [DQN 的训练算法](#7-dqn-的训练算法)
8. [Unit 3：Space Invaders 项目](#8-unit-3space-invaders-项目)
9. [两个项目的总结与常见问题](#9-两个项目的总结与常见问题)
10. [参考资料](#10-参考资料)

## 1. 从交互到动作价值

### 1.1 状态、动作、奖励和回合

在离散时间步 $t$，环境给出状态 $s_t$；智能体按策略选择动作 $a_t$；环境返回奖励 $r_{t+1}$、下一状态 $s_{t+1}$ 以及回合是否结束。一次交互记为 $(s_t,a_t,r_{t+1},s_{t+1})$。训练的目标是让智能体取得尽可能大的未来累计奖励，而不是只贪图当前这一步的奖励。

从时刻 $t$ 开始的折扣回报定义为：

$$G_t=\sum_{k=0}^{\infty}\gamma^k r_{t+k+1},\qquad 0\leq\gamma<1.$$

$\gamma$ 是折扣因子。越接近 0，越重视眼前奖励；越接近 1，越重视较远的后果。有限回合在终止后不再有未来奖励。

**动作价值函数** $Q^{\pi}(s,a)$ 表示：在状态 $s$ 先做动作 $a$，之后按策略 $\pi$ 行动时，预期得到多少折扣回报。最优动作价值 $Q^*(s,a)$ 假设之后始终采取最优策略。如果已经知道最优 Q 值，则策略可写为：

$$\pi^*(s)=\arg\max_a Q^*(s,a).$$

奖励是某一步实际收到的反馈，Q 值是关于长期回报的估计。Taxi 中即使每移动一步都会扣分，先移动到乘客位置、接人再送达，仍可能比反复尝试非法上下车更有价值。

### 1.2 本课程的两个小环境

| 环境 | 状态、动作与目标 | 学习重点 |
| --- | --- | --- |
| FrozenLake-v1，4×4 | 16 个格子状态，4 个移动动作；从起点到终点，避免落入冰洞 | 稀疏奖励、探索；`is_slippery=False` 时转移较容易理解 |
| Taxi-v3 | 500 个离散状态、6 个动作；接乘客并送达指定地点 | 状态编码、动作惩罚与长期规划 |

FrozenLake 的动作奖励规则、滑动模式与地图配置见 [Gymnasium 官方文档](https://gymnasium.farama.org/environments/toy_text/frozen_lake/)；Taxi 的状态、动作与奖励规则见 [Taxi 官方文档](https://gymnasium.farama.org/environments/toy_text/taxi/)。课程 [Unit 2 实践](https://huggingface.co/learn/deep-rl-course/en/unit2/hands-on) 使用这两个环境从零实现 Q-learning。

## 2. 表格型 Q-learning 的原理

### 2.1 Q 表是什么

当状态和动作都是数量可控的离散编号时，可以创建二维数组：

```text
Q[状态编号, 动作编号] = 对该状态执行该动作后的预计长期回报
```

行数等于状态数，列数等于动作数。例如 FrozenLake 4×4 的 Q 表形状是 `(16, 4)`；Taxi-v3 的表形状是 `(500, 6)`。开始时将表置零，经过训练后，各单元逐渐反映经验。Q 表是**价值函数的一种表示**；Q-learning 是**更新它的算法**，不能把两者混为一谈。[课程原理](https://huggingface.co/learn/deep-rl-course/en/unit2/q-learning)

### 2.2 贝尔曼最优方程

最优动作价值满足：

$$Q^*(s,a)=\mathbb E\bigl[r+\gamma\max_{a'}Q^*(s',a')\mid s,a\bigr].$$

这个等式的直觉是：一个动作的价值由**这一步的奖励**和**下一状态能取得的最优未来价值**共同决定。训练时未知真实 $Q^*$，便用当前估计值构造一步时序差分目标（TD target）：

$$y_t=r_{t+1}+\gamma(1-d_t)\max_{a'}Q(s_{t+1},a').$$

$d_t=1$ 表示转移进入真正的终止状态，此时没有后续价值；否则 $d_t=0$。然后只更新本次访问的那个状态—动作对：

$$Q(s_t,a_t)\leftarrow Q(s_t,a_t)+\alpha\bigl[y_t-Q(s_t,a_t)\bigr].$$

其中 $\alpha\in(0,1]$ 是学习率，$y_t-Q(s_t,a_t)$ 是 TD 误差。其他未访问的 Q 表单元在这一步不变。课程伪代码也采用“执行一步、观察新状态、更新 Q 值”的顺序。[课程算法](https://huggingface.co/learn/deep-rl-course/en/unit2/q-learning)

### 2.3 为什么称作时序差分学习

蒙特卡洛方法通常要等到回合结束，得到完整回报后再更新；Q-learning 每得到一步奖励和下一状态，就能通过已有的 $Q(s',a')$ 估计后续价值，立即更新。因此它是**一步 TD 学习**。这里用已有估计值构造新目标，称为**自举**（bootstrapping）。好处是无需等回合结束；代价是目标本身含有估计误差。

### 2.4 探索与利用：$\epsilon$-greedy

仅选当前 Q 值最大的动作可能错过更好的路线；始终随机选动作又无法利用已学知识。$\epsilon$-greedy 的规则是：

$$a_t=\begin{cases}
\text{随机动作}, & \text{概率 }\epsilon,\\
\arg\max_a Q(s_t,a), & \text{概率 }1-\epsilon.
\end{cases}$$

可以令探索率随回合编号 $i$ 衰减：

$$\epsilon_i=\epsilon_{\min}+(\epsilon_{\max}-\epsilon_{\min})e^{-ki}.$$

初期多探索；后期多利用，但保留少量探索。Q 表刚初始化时，一行中可能所有动作的值都相同；**并列最大值随机打破平局**，能避免总是选编号最小的动作。评估时应关闭随机探索，使用贪心策略，以测量已学到的决策能力。

### 2.5 为什么 Q-learning 是离策略算法

在环境中行动时使用 $\epsilon$-greedy 策略；TD 目标却使用下一状态的 $\max_{a'}Q(s',a')$，假设下一步按贪心策略行动。**生成经验的行为策略**与**更新所估计的目标策略**不同，因此称为离策略。对照之下，SARSA 的目标含有按当前策略实际选择的下一动作 $a'$，属于同策略方法。见 [课程对两类策略的说明](https://huggingface.co/learn/deep-rl-course/en/unit2/q-learning)。

### 2.6 `terminated` 与 `truncated` 的区别

Gymnasium 的 `env.step()` 返回 `terminated` 和 `truncated`。前者表示环境任务本身终止，例如到达目标或失败；后者通常表示达到时间上限。两者都意味着**当前回合停止交互**；对于有时间限制的 TD 目标，通常只在真正终止时把后续价值设为零，而在时间截断时允许从最后观测继续自举。[Gymnasium 环境 API](https://gymnasium.farama.org/api/env/)

这个细节在课程的入门伪代码中可能被简化；下面的实现显式区分两者。在手动设置的 `max_steps` 到达时，也只是结束本次训练回合，不会错误地把最后一步当成环境真正终止。

## 3. 手算一次 Q 值更新

设当前 $Q(s,a)=2$，这一步奖励 $r=1$，下一状态所有动作中最大的 Q 值是 $4$，取 $\gamma=0.9$、$\alpha=0.1$，且没有终止：

$$y=1+0.9\times4=4.6,$$

$$\delta=y-Q(s,a)=4.6-2=2.6,$$

$$Q_{\mathrm{new}}(s,a)=2+0.1\times2.6=2.26.$$

如果这一步已真正终止，则 $y=r=1$，新值为 $2+0.1(1-2)=1.9$。可见**终止标记改变目标，不只是决定是否跳出循环**。

## 4. Unit 2：完整程序代码

### 4.1 安装与运行

示例只依赖 Gymnasium 和 NumPy，适合直接观察 Q 表、TD 更新和评估流程：

```bash
python -m pip install "gymnasium[toy-text]" numpy
python q_learning_demo.py
```

将下面代码单独保存为 `q_learning_demo.py`。本笔记只存放代码文本，不会自动生成该 `.py` 文件。运行时会先训练确定性 FrozenLake，再训练 Taxi（优先课程所用的 `Taxi-v3`，未注册时改用 `Taxi-v4`），并打印平均回报、标准差、`mean - std`、成功率和 Q 表形状。超参数是教学示例，并非课程认证保证值。

```python
from dataclasses import dataclass

import gymnasium as gym
import numpy as np


@dataclass(frozen=True)
class Config:
    episodes: int
    max_steps: int
    learning_rate: float
    gamma: float
    min_epsilon: float
    decay_rate: float
    seed: int = 42


def greedy_action(q_row: np.ndarray, rng: np.random.Generator) -> int:
    """在最大 Q 值并列时随机选择，以免固定偏向动作 0。"""
    candidates = np.flatnonzero(np.isclose(q_row, q_row.max()))
    return int(rng.choice(candidates))


def train_q_learning(env: gym.Env, cfg: Config) -> np.ndarray:
    if not isinstance(env.observation_space, gym.spaces.Discrete):
        raise TypeError("该示例要求离散状态空间")
    if not isinstance(env.action_space, gym.spaces.Discrete):
        raise TypeError("该示例要求离散动作空间")

    q_table = np.zeros(
        (env.observation_space.n, env.action_space.n), dtype=np.float64
    )
    rng = np.random.default_rng(cfg.seed)
    env.action_space.seed(cfg.seed)

    for episode in range(cfg.episodes):
        epsilon = cfg.min_epsilon + (1.0 - cfg.min_epsilon) * np.exp(
            -cfg.decay_rate * episode
        )
        state, _ = env.reset(seed=cfg.seed + episode)

        for _ in range(cfg.max_steps):
            if rng.random() < epsilon:
                action = int(env.action_space.sample())
            else:
                action = greedy_action(q_table[state], rng)

            next_state, reward, terminated, truncated, _ = env.step(action)

            # 真正终止：未来奖励为 0；时间截断：仍允许从下一状态自举。
            next_best = 0.0 if terminated else float(q_table[next_state].max())
            td_target = reward + cfg.gamma * next_best
            td_error = td_target - q_table[state, action]
            q_table[state, action] += cfg.learning_rate * td_error

            state = next_state
            if terminated or truncated:
                break

    return q_table


def evaluate(
    env: gym.Env,
    q_table: np.ndarray,
    episodes: int,
    max_steps: int,
    seed: int,
) -> tuple[float, float, float]:
    """使用无探索的贪心策略，返回平均回报、标准差、成功率。"""
    rng = np.random.default_rng(seed)
    returns: list[float] = []
    successes = 0

    for episode in range(episodes):
        state, _ = env.reset(seed=seed + episode)
        episode_return = 0.0

        for _ in range(max_steps):
            action = greedy_action(q_table[state], rng)
            state, reward, terminated, truncated, _ = env.step(action)
            episode_return += float(reward)
            # 两个课程环境的成功终止都给正奖励；落入冰洞不算成功。
            if terminated and reward > 0:
                successes += 1
            if terminated or truncated:
                break

        returns.append(episode_return)

    scores = np.asarray(returns, dtype=np.float64)
    return float(scores.mean()), float(scores.std()), successes / episodes


def run_experiment(env_id: str, cfg: Config, **env_kwargs) -> None:
    training_env = gym.make(env_id, **env_kwargs)
    evaluation_env = gym.make(env_id, **env_kwargs)
    try:
        q_table = train_q_learning(training_env, cfg)
        mean, std, success_rate = evaluate(
            evaluation_env,
            q_table,
            episodes=100,
            max_steps=cfg.max_steps,
            seed=cfg.seed + 100_000,
        )
        print(f"\n{env_id} {env_kwargs}")
        print(f"Q 表形状: {q_table.shape}")
        print(f"平均回报: {mean:.3f}, 标准差: {std:.3f}")
        print(f"mean - std: {mean - std:.3f}")
        print(f"成功率: {success_rate:.1%}")
        print("状态 0 的 Q 值:", q_table[0])
    finally:
        training_env.close()
        evaluation_env.close()


if __name__ == "__main__":
    run_experiment(
        "FrozenLake-v1",
        Config(
            episodes=20_000,
            max_steps=100,
            learning_rate=0.8,
            gamma=0.95,
            min_epsilon=0.05,
            decay_rate=0.0005,
        ),
        map_name="4x4",
        is_slippery=False,
    )
    # 课程使用 Taxi-v3；若当前 Gymnasium 未注册它，则用新版 Taxi-v4。
    taxi_env_id = "Taxi-v3" if "Taxi-v3" in gym.envs.registry else "Taxi-v4"
    run_experiment(
        taxi_env_id,
        Config(
            episodes=40_000,
            max_steps=200,
            learning_rate=0.1,
            gamma=0.99,
            min_epsilon=0.05,
            decay_rate=0.0002,
        ),
    )
```

> [!note] 成功率与环境版本
> `success_rate` 统计“真正终止且终止这一步获得正奖励”的比例，适用于本文的默认 FrozenLake 和 Taxi 奖励规则；FrozenLake 落入冰洞虽然也会 `terminated`，但不算成功。课程使用 `Taxi-v3`，当前 Gymnasium [Taxi 文档](https://gymnasium.farama.org/environments/toy_text/taxi/)展示 `Taxi-v4`，因此代码在 `Taxi-v3` 未注册时退回到 `Taxi-v4`。不同版本的实验结果应分别记录，不能直接当成课程 `Taxi-v3` 的分数。

## 5. 代码逐段解释与实验设计

### 5.1 初始化 Q 表

`env.observation_space.n` 和 `env.action_space.n` 分别给出离散状态数与动作数。`np.zeros((n_states, n_actions))` 建表。代码先检查是否为 `Discrete`，是因为图像观测和连续动作无法直接以整数索引访问这张表。

### 5.2 选动作

`rng.random() < epsilon` 时调用 `env.action_space.sample()` 探索；否则调用 `greedy_action()`。`np.isclose` 寻找并列最大值，再随机挑一个动作。评估函数始终调用 `greedy_action()`，因此没有 $\epsilon$ 随机动作。

### 5.3 更新 Q 值

代码中的关键四行，与公式逐项对应：

```python
next_best = 0.0 if terminated else float(q_table[next_state].max())
td_target = reward + cfg.gamma * next_best
td_error = td_target - q_table[state, action]
q_table[state, action] += cfg.learning_rate * td_error
```

`next_best` 是 $\max_{a'}Q(s',a')$，`td_target` 是目标 $y$，`td_error` 是 $y-Q(s,a)$。每一步只修改当前 `(state, action)`。注意必须先用**本次转移的旧状态**更新，再把 `state` 赋为 `next_state`。

### 5.4 评估指标与调参

- **平均回报**：在多个独立回合中，累计奖励的平均值；更接近任务目标。
- **标准差**：不同回合成绩的波动；高均值伴随很大波动，策略可能不稳定。
- **成功率**：在本例中为成功到达终点或送达乘客的比例；与平均回报解释角度不同。
- **`mean - std`**：课程排行榜采用的保守分数形式；应在该排行榜要求的环境与评估流程下比较，不能把任意本地 100 回合结果当作官方成绩。

若 FrozenLake 长时间学不到路径，可检查 `is_slippery`、训练回合数和探索衰减。奖励稀疏时，过快降低 $\epsilon$ 会使终点附近的奖励难以传播到起点。若 Taxi 有大量负回报，先检查非法上下车惩罚、训练是否充分，再检查 `learning_rate`、`gamma` 和探索率；调参最好每次只改一项并保留随机种子。课程 Unit 2 实践还包含滑动版 FrozenLake 及模型上传步骤。[课程实践](https://huggingface.co/learn/deep-rl-course/en/unit2/hands-on)

## 6. 从 Q 表到 DQN

### 6.1 Q 表的规模问题

Q 表需要为每个状态—动作对保存一个数。课程举例：Atari 原始单帧为 $210\times160\times3$ 的 RGB 图像，每个通道值可取 0 到 255；理论可能观测数为 $256^{210\times160\times3}=256^{100800}$。即使状态只是一张图，也无法枚举每幅画面并建一行 Q 表。[课程：从 Q-learning 到 DQN](https://huggingface.co/learn/deep-rl-course/en/unit3/from-q-to-dqn)

DQN 使用带参数 $\theta$ 的神经网络 $Q_\theta(s,a)$ 来近似价值函数。输入一个状态，输出每个离散动作对应的 Q 值；选动作时依然可以使用 $\epsilon$-greedy。它通过共享网络参数，让相似的视觉状态共享部分特征，避免逐状态存储。

| 方面 | 表格型 Q-learning | DQN |
| --- | --- | --- |
| 状态形式 | 小型离散编号 | 图像等高维观测 |
| Q 值表示 | `Q[state, action]` | 网络 $Q_\theta(s,a)$ |
| 一次更新 | 修改一个表格单元 | 对批量样本反向传播，修改网络参数 |
| 选动作 | $\epsilon$-greedy | $\epsilon$-greedy |
| 训练稳定性 | 主要看探索与学习率 | 还需要经验回放、目标网络等机制 |
| 局限 | 状态空间增大时表急剧膨胀 | 训练需要更多样本与计算；基础 DQN 用离散动作 |

### 6.2 图像预处理和四帧堆叠

课程 DQN 网络将 Atari 输入变为 $84\times84$ 灰度图，并堆叠连续四帧。灰度和缩放降低输入维度；连续帧让模型辨别物体移动趋势。例如单张画面看到球的位置，却无法判断它正向左还是向右。随后卷积层提取空间特征，全连接层输出各动作 Q 值。[课程：DQN 网络](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-network)

> [!important] 四帧堆叠的作用
> 它提供有限的时间信息，并不等于完整记录历史，也不保证任意环境下都满足马尔可夫性质。课程中用于 Atari 输入，是一种实用的状态构造办法。

## 7. DQN 的训练算法

### 7.1 从表格更新到损失函数

表格型算法能直接改 `Q[s,a]`；神经网络的一个参数可能影响很多状态的预测，因此 DQN 将 TD 误差写成损失，并用梯度下降更新：

$$y_i=r_i+\gamma(1-d_i)\max_{a'}Q_{\theta^-}(s'_i,a'),$$

$$L(\theta)=\frac{1}{B}\sum_{i=1}^{B}\ell\bigl(Q_\theta(s_i,a_i),y_i\bigr).$$

$B$ 是批量大小；$\theta$ 是在线网络参数，$\theta^-$ 是目标网络参数。$\ell$ 可以是平方误差或 Huber 损失。只对经验中**实际采取的动作 $a_i$** 的 Q 输出计算损失；目标 $y_i$ 在反向传播中视为常数。[课程：DQN 算法](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-algorithm)

核心计算可写成下列**示意片段**，它不是完整的 Atari 训练脚本：

```python
# online 和 target 是 PyTorch 网络；batch 是经验回放抽取的张量。
q_sa = online(batch.states).gather(1, batch.actions[:, None]).squeeze(1)
with torch.no_grad():
    next_q = target(batch.next_states).max(dim=1).values
    y = batch.rewards + gamma * (~batch.terminated).float() * next_q
loss = torch.nn.functional.smooth_l1_loss(q_sa, y)
optimizer.zero_grad()
loss.backward()
optimizer.step()
# 定期调用：target.load_state_dict(online.state_dict())
```

`batch.terminated` 必须是布尔张量；真正运行时还需要处理设备、图像预处理、经验采集和随机种子。因此项目部分采用课程推荐的 RL-Baselines3-Zoo，而不把上面片段假装成完整训练代码。

### 7.2 经验回放（experience replay）

每次环境交互得到一条经验 `(state, action, reward, next_state, terminated)`，先加入有容量上限的缓冲区。缓冲区积累足够经验后，训练时随机抽取一小批样本。这样同一经验可被多次利用，训练样本也不总是按时间顺序高度相关。应注意：经验回放**缓解**相关性，并不证明样本完全独立。[课程：经验回放](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-algorithm)

### 7.3 目标网络（fixed Q-target）

如果同一个网络既预测当前 $Q_\theta(s,a)$，又生成目标 $r+\gamma\max Q_\theta(s',a')$，每次更新都使“答案”一起移动，容易震荡。DQN 保留目标网络 $Q_{\theta^-}$，隔一定步数才把在线网络权重复制过去。在两次复制之间，TD 目标相对稳定。[课程：目标网络](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-algorithm)

### 7.4 Double DQN 与本项目的区别

`max` 作用于有噪声的估计时，可能倾向于选中被高估的动作。Double DQN 将“下一动作的选择”和“该动作的价值评估”分开：

$$a^*=\arg\max_{a'}Q_\theta(s',a'),\qquad
 y^{\mathrm{Double}}=r+\gamma(1-d)Q_{\theta^-}(s',a^*).$$

课程理论讲到了这种改进；**Unit 3 的 RL-Baselines3-Zoo 项目使用基础 DQN**，Stable-Baselines3 官方也注明其 DQN 实现没有 Double-DQN、Dueling-DQN 或优先经验回放扩展。[课程算法](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-algorithm) · [SB3 DQN 文档](https://stable-baselines3.readthedocs.io/en/master/modules/dqn.html)

## 8. Unit 3：Space Invaders 项目

### 8.1 项目目标与交付

课程实践在 `SpaceInvadersNoFrameskip-v4` 中训练 DQN，通过 RL-Baselines3-Zoo 完成配置、训练、评估、模型卡和 Hub 上传。课程实践页列出的认证指标是排行榜的 `mean_reward - std_reward >= 200`。这只是课程项目要求，本文没有执行 Atari 训练，也没有已获分数。[Unit 3 实践](https://huggingface.co/learn/deep-rl-course/en/unit3/hands-on)

预期交付包括：训练配置 `dqn.yml`、`logs/` 中的模型与日志、评估结果，以及上传 Hub 后的模型仓库。发布需要自己的 Hugging Face 账号与写权限。

### 8.2 项目配置：`dqn.yml`

课程给出的主要配置如下：

```yaml
SpaceInvadersNoFrameskip-v4:
  env_wrapper:
    - stable_baselines3.common.atari_wrappers.AtariWrapper
  frame_stack: 4
  policy: 'CnnPolicy'
  n_timesteps: !!float 1e7
  buffer_size: 100000
  learning_rate: !!float 1e-4
  batch_size: 32
  learning_starts: 100000
  target_update_interval: 1000
  train_freq: 4
  gradient_steps: 1
  exploration_fraction: 0.1
  exploration_final_eps: 0.01
  optimize_memory_usage: False
```

| 参数 | 在项目中的意义 |
| --- | --- |
| `env_wrapper` | Atari 输入预处理；与帧堆叠共同构成网络输入 |
| `frame_stack: 4` | 组合连续四帧以提供运动信息 |
| `policy: CnnPolicy` | 使用适合图像输入的卷积网络策略 |
| `n_timesteps` | 环境交互总步数；课程配置为一千万步 |
| `buffer_size` | 经验回放容量，示例为十万条 |
| `learning_rate` | 优化器的学习率 |
| `batch_size` | 每次从回放缓冲区抽取的样本数 |
| `learning_starts` | 缓冲区先积累多少经验，再开始梯度更新 |
| `target_update_interval` | 目标网络更新间隔 |
| `train_freq` 与 `gradient_steps` | 训练触发频率和每次触发的梯度更新次数 |
| `exploration_fraction`、`exploration_final_eps` | 探索率衰减的阶段和最低值 |

课程实践页解释了这些参数及训练步数；更精确的参数语义应查 [Stable-Baselines3 DQN 参数文档](https://stable-baselines3.readthedocs.io/en/master/modules/dqn.html#parameters)。对于计算资源有限的练习，可以先把步数缩短以检查流程，再进行正式训练；短跑结果不能据此推断认证成绩。[课程项目配置](https://huggingface.co/learn/deep-rl-course/en/unit3/hands-on)

### 8.3 安装和环境准备

课程 Notebook 使用 Colab GPU，安装 RL-Baselines3-Zoo、Gymnasium Atari 依赖和 ROM，并为无显示器环境配置虚拟显示。课程页面中的安装命令可能依赖其当时的版本组合；实际执行时优先遵循课程 Notebook 与 [RL-Baselines3-Zoo 项目说明](https://github.com/DLR-RM/rl-baselines3-zoo)。可用以下命令先检查关键组件：

```bash
python -m pip install git+https://github.com/DLR-RM/rl-baselines3-zoo
python -m pip install "gymnasium[atari]"
python -m rl_zoo3.train --help
```

Atari ROM 获取、游戏环境命名和附加依赖应以当前 [Gymnasium Atari 文档](https://gymnasium.farama.org/environments/atari/) 为准。若提示环境未注册，先检查 ROM 与安装版本，再检查 `SpaceInvadersNoFrameskip-v4` 这个课程使用的环境 ID 是否仍可用；不要只修改 YAML 键而忘记同时修改训练命令中的环境名。

### 8.4 训练、评估、上传命令

在含有 `dqn.yml` 的目录执行课程命令：

```bash
python -m rl_zoo3.train --algo dqn \
  --env SpaceInvadersNoFrameskip-v4 -f logs/ -c dqn.yml

python -m rl_zoo3.enjoy --algo dqn \
  --env SpaceInvadersNoFrameskip-v4 \
  --no-render --n-timesteps 5000 --folder logs/

python -m rl_zoo3.push_to_hub --algo dqn \
  --env SpaceInvadersNoFrameskip-v4 \
  --repo-name dqn-SpaceInvadersNoFrameskip-v4 \
  -orga YOUR_HF_USERNAME -f logs/
```

第一条训练并将模型保存于 `logs/`；第二条载入模型，运行 5000 环境步，观察或检查策略；第三条在完成 Hub 登录后上传模型。`YOUR_HF_USERNAME` 需要换成自己的用户名。课程实践说明 `push_to_hub` 还会生成模型卡与回放材料。**5000 步展示和排行榜多回合评分是不同环节**，不能把展示步数或某次回合得分写作排行榜成绩。[课程命令](https://huggingface.co/learn/deep-rl-course/en/unit3/hands-on)

### 8.5 如何判断项目是否完成

1. 训练命令结束，`logs/` 中确有模型文件和可检查的训练记录。
2. 评估脚本能载入对应环境的模型并完成运行。
3. 发布后 Hub 模型仓库包含模型、配置或模型卡等材料。
4. 排行榜出现该模型的评估结果，并按课程规定的 `mean_reward - std_reward` 判断是否达标。
5. 记录实际使用的依赖版本、训练步数、配置与随机种子，避免今后复现实验时无法解释差异。

## 9. 两个项目的总结与常见问题

### 9.1 从 FrozenLake/Taxi 到 Space Invaders 的知识线

```text
状态和动作 → 奖励与未来回报 → Q(s,a)
     ↓
小型离散状态：Q 表 + ε-greedy + TD 单元更新
     ↓
高维图像状态：图像预处理 + 卷积网络 Qθ(s,a)
     ↓
批量 TD 损失 + 经验回放 + 目标网络
     ↓
Space Invaders 训练 → 评估 → 上传模型 → 检查排行榜
```

第 2 单元的核心不是“把动作编号填进表里”，而是理解价值估计如何从环境反馈中更新。第 3 单元保留动作价值与 $\epsilon$-greedy 决策方式，只是表示和学习 Q 值的方式发生了变化。图像输入又带来样本相关、目标漂移和计算成本，因此需要专门的训练机制。

### 9.2 容易混淆的点

- **Q-learning 与 Q 表不是同义词**：前者是训练算法，后者是小型离散问题的一种数据结构。
- **即时奖励不是 Q 值**：Q 值考虑后续折扣奖励。
- **`terminated` 与 `truncated` 不相同**：两者都会停止回合，但 TD 目标对未来价值的处理可以不同。
- **Q-learning 的目标使用最大下一动作价值**：它与当前行为策略是否随机探索无关，这正是离策略的关键。
- **DQN 不是简单地把表换成网络**：网络更新会同时影响许多状态，训练还需要回放缓冲区和目标网络稳定目标。
- **Double DQN 是扩展算法**：课程理论介绍了它，Space Invaders 的基础 DQN 项目没有启用。
- **演示成功不等于课程认证**：必须根据实际评估结果与课程规定指标判断。

### 9.3 最终速记

$$\boxed{\text{表格型 Q-learning：}\quad Q(s,a)\leftarrow Q(s,a)+\alpha\bigl[r+\gamma\max_{a'}Q(s',a')-Q(s,a)\bigr]}$$

真正终止时去掉后续价值项；时间截断时根据任务定义决定是否自举。

$$\boxed{\text{DQN：}\quad Q_\theta(s,a)\xrightarrow{\text{与 TD 目标比较}}L(\theta)\xrightarrow{\text{梯度下降}}\theta_{\mathrm{new}}}$$

基础 DQN 在此之外使用经验回放、目标网络，并对 Atari 图像做缩放、灰度化和多帧堆叠。

## 10. 参考资料

- [Hugging Face Deep RL Course：Unit 2 介绍](https://huggingface.co/learn/deep-rl-course/en/unit2/introduction)
- [Unit 2：Q-learning 原理](https://huggingface.co/learn/deep-rl-course/en/unit2/q-learning)
- [Unit 2：FrozenLake 与 Taxi 实践](https://huggingface.co/learn/deep-rl-course/en/unit2/hands-on)
- [Unit 3：从 Q-learning 到 DQN](https://huggingface.co/learn/deep-rl-course/en/unit3/from-q-to-dqn)
- [Unit 3：DQN 网络](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-network)
- [Unit 3：DQN 算法](https://huggingface.co/learn/deep-rl-course/en/unit3/deep-q-algorithm)
- [Unit 3：Space Invaders 实践](https://huggingface.co/learn/deep-rl-course/en/unit3/hands-on)
- [Gymnasium：FrozenLake](https://gymnasium.farama.org/environments/toy_text/frozen_lake/) · [Taxi](https://gymnasium.farama.org/environments/toy_text/taxi/) · [Env API](https://gymnasium.farama.org/api/env/)
- [Stable-Baselines3：DQN](https://stable-baselines3.readthedocs.io/en/master/modules/dqn.html)
