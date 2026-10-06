---
tags:
  - AI
---
# 从语言模型到 agent：感知—行动循环

## 本节回答的问题

agent 的严格定义是什么？它与“会推理的语言模型”差在哪里，奖励又从哪里来？

## 定义与四个要素

agent 是任何可以被视为“通过传感器感知环境、并通过执行器作用于环境”的东西。环境是一个仓库、一个网站、一个应用或一条工作流；状态与观测是消息、文件内容、网页、截图以及工具返回的结果；动作是回复用户、编辑文件、在 shell 里执行命令、调用网页背后的 API。


还有一个常被忽略的要素：**奖励**。要谈“这个 agent 做得好不好”，就需要一个数值化的成功判据。课程给出的三种来源按自动化程度递减：
- 代码任务的测试用例通过得 1、否则得 0；
- 无法程序化判定的任务用 LLM-as-a-judge（第二次作业就要实现它）；
- 最贴近真实目标的是用户反馈，这一部分会在后续“人与 agent 交互”的客座讲座里展开。
> **agent 的四元组**
>
> ***环境 $\mathcal{E}$、观测 $o_t$、动作 $a_t$、奖励 $r$*** 缺一不可。工程上最容易被忽略的是 $r$：如果没人能说清“怎样算成功”，后面的评测、强化学习与可靠性分析都无从谈起；这也是课程把“设计评测”列为四项学习目标之一的原因。
## 基线的语言模型：next-token prediction 与思维链
在讨论 agent 之前，幻灯片先回顾了两个非 agent 的事实。
**第一是 next-token prediction**：每个位置模型输出下一个 token 上的概率分布，采样（或按策略选一个）之后拼回前缀，再预测下一个。
$$p_\theta(x_t \mid x_1, \dots, x_{t-1}) = p_\theta(x_t \mid x_{<t})$$
式中 $x_{<t}$ 为已生成的前缀，$\theta$ 为模型参数；整个序列的生成即在这条分布上逐步前推。幻灯片给的例子是提示 2 × (3 + 4) 对应的答案 10——模型必须把这些中间 token 一起生成出来。
**第二是 chain of thought（思维链）**：在给出最终答案之前，先生成一段中间推理 token。这些 token 不是任务要求的输出，而是模型的草稿纸（scratchpad），后续生成可以以它们为条件。

> **草稿纸不是行动**
>
> 思维链改变的是"模型在作答时能看到什么"，它不会改变外部世界；一个动作（编辑文件、发邮件、跑命令）会改变环境的状态。把两者混为一谈，就会误以为"模型说了要做"等于"事情已经做了"——这正是很多 agent bug 的来源，也是下一节工具接口要解决的问题。

### 本章小结

agent = 语言模型（在循环里）+ 环境 + 观测 + 动作 + 奖励。定义本身与 1990 年代的 AI 教科书一致，新的部分在于"感知"与"行动"的接口由自然语言和工具调用实现。下一节就讲这个接口。



###  为什么使用json来作为给ai 的接口
#### 1. 结构化与机器可读性（最核心原因）

自然语言（如“帮我查一下北京明天的天气”）是模糊的，而代码执行需要精确的参数。

- **JSON 提供了严格的键值对结构**：`{"location": "北京", "date": "明天"}`。
    
- 程序（Agent 的执行器）可以直接解析这个 JSON，提取出 `location` 和 `date` 变量，直接传给后端的天气 API。如果不包装成 JSON，程序就需要用正则表达式或另一个 NLP 模型去解析自然语言，不仅容易出错，而且极其不稳定。
    

#### 2. 大模型训练与“对齐”的产物

现在的 LLM 在预训练和后期的指令微调（SFT）阶段，被灌输了海量的代码、API 文档和结构化数据。

- 模型发现，**JSON 是描述“函数签名”和“参数”最紧凑、最无歧义的方式**。
    
- 当你在 Prompt 中给模型一个 JSON Schema（比如 `{"name": "get_weather", "parameters": {"city": "string"}}`），模型能极好地理解“我现在需要输出一个符合这个格式的 JSON”。
    
- 甚至 OpenAI 等厂商专门推出了 **JSON Mode** 或 **Structured Outputs**，强制模型输出的 Token 必须符合 JSON 语法，从底层保证了接口的可靠性。
    

#### 3. 解决“草稿纸不是行动”的边界问题

结合你上一张图的观点，JSON 是区分**“思考”**和**“行动”**的物理边界：

- **思维链（CoT）**：模型可以用自然语言自由地“草稿纸”推理（“我觉得应该先查天气，再决定穿什么”）。
    
- **工具调用（Action）**：当模型决定行动时，它必须输出一段特定的 JSON（例如 `{"tool": "get_weather", "args": {...}}`）。




### 一、 核心观点：打破“JSON 是唯一接口”的错觉

图片中的红色警告框指出了核心：**“JSON 是接口”是错觉。**

- **底层逻辑**：不管模型输出的是 JSON、Python 代码还是 Bash 命令，本质上都是**模型生成了特定分布的 Token 序列**。
    
- **Harness 的角色**：外部的 Harness（能力框架）负责接收这些 Token，把它们翻译成对真实世界的操作（执行系统调用、修改文件等）。**翻译层才是真正的接口**，JSON 只是翻译层约定的一种“方言”。
    

### 二、 为什么“直接写代码”往往更有效？

图中提到：“模型在预训练中见过海量代码，用代码表达动作比生成结构化 JSON 更符合它的分布。” 这涉及到大模型训练的本质：

1. **数据分布优势**：像 GPT-4、Claude 3 等模型，在预训练阶段阅读了 GitHub 上数以亿计的代码。它们极其熟悉 Python 的语法、Bash 命令的格式。
    
2. **无需“翻译”**：让模型输出 `print("Hello")` 比让它输出 `{"action": "print", "args": {"text": "Hello"}}` 更接近它原本的训练数据。
    
3. **灵活性与表达力**：有些操作很难用 JSON 清晰表达（比如复杂的循环、条件判断、多步依赖）。直接用代码可以实现更复杂的逻辑，比如：



### 三、 为什么 JSON 依然存在且流行？

尽管直接写代码有优势，但 JSON 并没有被淘汰，原因在于**权衡（Trade-off）**：

|维度|JSON 接口|直接写代码|
|---|---|---|
|**模型适应性**|需要专门的微调（如 Function Calling）|天然契合预训练分布|
|**Harness 解析难度**|极低（直接解析键值对）|极高（需要解析 AST 语法树）|
|**安全隔离**|容易做白名单校验（只允许调用指定函数）|极难（代码可能包含 `rm -rf /`）|
|**执行环境**|不需要代码执行沙箱|必须使用强隔离的沙箱（如 Docker）|
|**适用场景**|标准化 API 调用、简单参数传递|复杂逻辑、文件操作、系统运维|


# ReAct: 把工具接成循环

## 本节回答的问题

一个 agent 的一步由哪些部分组成？循环什么时候结束？最小的实现有多小？

## 一步的结构与循环的四要素

这门课对"agent 循环"的经典参照是 ReAct (Yao et al., ICLR 2023)：推理（reasoning）与行动（acting）交替。幻灯片把上下文拆成四块：

- **Task**（任务定义，例如"在 GitHub 上解决这个 issue"）
- **History**（过去的动作与观测）
- **Context**（工具清单与系统提示）
- **当前状态**

模型在每个时间步基于全部上下文产出一段思维链，再产出一个或多个工具调用。

## 一步的流程更新式

一步的流程可以写成一个更新式：

$$
\begin{aligned}
c_t &\sim \pi_\theta(\cdot \mid \text{Task}, \text{Tools}, h_t), \\
a_t &\sim \pi_\theta(\cdot \mid \text{Task}, \text{Tools}, h_t, c_t), \\
o_t &= \text{Exec}(a_t), \\
h_{t+1} &= h_t \oplus (c_t, a_t, o_t)
\end{aligned}
$$

### 符号说明

- $h_t$：为第 $t$ 步之前的历史（动作与观测）
- $c_t$：为思维链
- $a_t$：为工具调用
- $\text{Exec}$：是在环境中执行调用的 harness
- $o_t$：是返回的观测
- $\oplus$：表示把三者追加进历史

> **循环终止条件**：循环持续到某个调用显式结束任务为止。





## 最小实现：mini-swe-agent

幻灯片展示了一个"极简但有效"的实现 mini-swe-agent：它在 SWE-bench 这类编码 agent 基准上有很好的表现，而代码只有几段。核心就是四个方法：

- **`run`**：初始化消息并进入无限循环
- **`step`**：先查询再执行动作
- **`query`**：把完整历史发给模型并追加回复
- **`execute_actions`**：执行消息中的动作，把观测追加为消息并返回

### 代码实现

```python
def run(self, task: str, **kwargs) -> dict:
    self.messages = []
    self.add_messages(
        self.model.format_message(role="system", content=self._render_template(self.config.system_template)),
        self.model.format_message(role="user", content=self._render_template(self.config.instance_template)),
    )
    while True:
        self.step()

def step(self) -> list[dict]:
    return self.execute_actions(self.query())

def query(self) -> dict:
    message = self.model.query(self.messages)
    self.add_messages(message)
    return message

def execute_actions(self, message: dict) -> list[dict]:
    """Execute actions in message, add observation messages, return them."""
    
  ```     
### 1. `run` 方法：初始化与启动循环

这是整个 Agent 的入口点。

python

def run(self, task: str, **kwargs) -> dict:
    self.messages = []
    # ... 拼接系统提示词和用户任务 ...
    while True:
        self.step()

- **做了什么**：
    
    1. **清空历史**：`self.messages = []` 对应公式中的初始历史 h0h0​。
        
    2. **构建上下文 (Context)**：通过 `add_messages` 将系统提示词（`system_template`，通常包含工具清单和规则）和用户的具体任务（`instance_template`，即 Task）加入到消息列表中。
        
    3. **进入无限循环**：`while True` 对应公式中“循环持续到某个调用显式结束任务为止”。它不断调用 `self.step()`，直到 `step` 内部抛出异常或触发终止条件（代码中虽未写出终止逻辑，但实际运行时会在执行到“结束任务”的动作时跳出循环）。
       
       
## 两条建设路线：训练模型还是工程化 harness

### 本节回答的问题

面对一条能力短板，应该去训练模型，还是去改 harness？讲者给出的排序依据是什么？

### **选择的成本结构**

两条路的差别不只是效果，还有**反馈周期**：改 harness 是小时级、可回滚、每个团队都能做；训练模型是周级、需要数据与环境、只有少数团队能做。讲者的"先 harness 后训练"不是偏好问题，而是由反馈周期决定的最优路径。








# 两侧手段对照


### 准确工具调用

幻灯片把一次工具调用拆成四步：选择工具（Select tool）→ 组装参数（Arguments）→ 校验（Validate）→ 执行（Execute）。两侧手段正好对应两个薄弱点：harness 侧用**文法约束解码**（grammar-constrained decoding）保证输出一定符合工具规格：

### **compaction 的代价**

开场那条清空收件箱的事故，根源正是压缩：总结历史时把"不要动手"这条约束丢了。压缩不是无损操作——**压缩策略必须显式保留约束类信息**（用户禁止事项、权限边界、验收标准），否则越长的任务越容易在某个压缩点失守。

### 可定制性

讲者认为这是当前 harness engineering 最好的用武之地：**agent memory**（让 agent 随着交互持续学习你的偏好）、**skills**（把提示与脚本按需拉进上下文）、**自定义工具**（为特定任务造专用接口）。训练侧的对应手段是从用户反馈学习（点赞/点踩等偏好信号）。他坦言这一侧在训练上仍是"新兴领域"，做深入工作的人还不多



