---
tags:
  - AI
---
## 1. 从搜索到知识推理

上一讲解决的是：**怎样在状态空间中找到目标？**  
这一讲解决的是：**怎样表示已知事实，并判断某个结论是否必然成立？**

知识型智能体（Knowledge-Based Agent）内部保存对世界的表示，并在其上进行推理：

```
flowchart LR
    A[已有事实与规则] --> B[Knowledge Base]
    B --> C[Inference]
    C --> D[新结论]
```

- **Sentence（句子）**：知识表示语言中关于世界的断言。
    
- **Knowledge Base，KB（知识库）**：智能体已知为真的句子集合。
    
- **Inference（推理）**：从旧句子导出新句子的过程。

## 2. 命题逻辑

命题（proposition）是一个只能取真或假的陈述。

- `P`：今天下雨。
    
- `Q`：Harry 在室内。
    

`P`、`Q` 称为命题符号或原子命题。命题逻辑用逻辑连接词组合原子命题。

## 3. 五种逻辑连接词

### 3.1 否定 NOT：`¬P`

| P   | ¬P  |
| --- | --- |
| F   | T   |
| T   | F   |


### 3.2 合取 AND：`P ∧ Q`

只有二者都为真时为真。

|P|Q|P ∧ Q|
|---|---|---|
|F|F|F|
|F|T|F|
|T|F|F|
|T|T|T|

### 3.3 析取 OR：`P ∨ Q`

课程中的 `∨` 是**包含或**：至少一个为真就为真，二者同时为真也为真。

|P|Q|P ∨ Q|
|---|---|---|
|F|F|F|
|F|T|T|
|T|F|T|
|T|T|T|

异或 XOR（`P ⊕ Q`）则要求恰好一个为真，不要和 `∨` 混淆。

### 3.4 蕴含 Implication：`P → Q`

读作“如果 P，那么 Q”。

- `P`：前件（antecedent）；
    
- `Q`：后件（consequent）。
    

|P|Q|P → Q|
|---|---|---|
|F|F|T|
|F|T|T|
|T|F|F|
|T|T|T|

它只在“前件真、后件假”时为假。

为什么 `P=False` 时总为真？因为 `P → Q` 只承诺了 **P 发生时 Q 必须发生**；P 根本没有发生时，这条规则没有被违反，称为“空真”或“平凡为真”。

重要等价式：

[  
$P \rightarrow Q \equiv \neg P \lor Q$  
$]$

## 4. Model、KB 与 Entailment

### 4.1 Model（模型）

模型是对所有命题符号的一组真值赋值。

例如：

[  
M={P=True,Q=False}  
]

若有 `n` 个命题符号，共有：

[  
2^n  
]

个可能模型。

### 4.2 Knowledge Base（知识库）

KB 是被智能体接受为真的所有句子的合取。若 KB 中有 `P`、`¬Q` 和 `(P ∧ ¬Q) → R`，整体可理解为：

$[$  
$KB=P\land \neg Q\land((P\land\neg Q)\rightarrow R)$  
$]$

### 4.3 Entailment（蕴涵关系）

$[$  
$KB \models \alpha$  
$]$

表示：在**每一个使 KB 为真的模型中**，`α` 也都为真。换句话说，只要接受 KB，就不可能否认 `α`。


|   |   |   |
|---|---|---|
|`P → Q`|对象语言中的逻辑连接词|一个可以求真值的公式|
|`KB ⊨ α`|元语言中的语义关系|KB 的所有模型也是 α 的模型|
|`KB ⊢ α`|证明关系|可用某套推理规则从 KB 推出 α|

课程重点使用 `⊨`。理想推理系统希望既可靠又完备：能证明的都确实被蕴涵，确实被蕴涵的也都能证明。

## 5. Model Checking（模型检查）

目标：判断 `KB ⊨ α` 是否成立。

算法：

1. 枚举全部 `2^n` 个模型；
    
2. 只关注使 KB 为真的模型；
    
3. 检查这些模型中 `α` 是否全部为真；
    
4. 若全部为真，则 `KB ⊨ α`。
    

### 5.1 例子

已知：

$[$  
$P,\quad \neg Q,\quad (P\land\neg Q)\rightarrow R$  
$]$

查询：是否 `KB ⊨ R`？

只有满足 `P=True`、`Q=False` 且整个蕴含式为真的模型符合 KB。在这种模型里，`R` 必须为真，所以结论成立。


### 5.2 递归代码的核心

```
def check_all(knowledge, query, symbols, model):
    if not symbols:
        if knowledge.evaluate(model):
            return query.evaluate(model)
        return True

    remaining = symbols.copy()
    p = remaining.pop()

    model_true = model.copy()
    model_true[p] = True

    model_false = model.copy()
    model_false[p] = False

    return (
        check_all(knowledge, query, remaining, model_true)
        and
        check_all(knowledge, query, remaining, model_false)
    )
```

理解这段代码的三个关键点：

1. 每取出一个 symbol，就把当前分支拆成 True 和 False 两个分支；
    
2. symbols 为空时得到一个完整模型；
    
3. 若该模型不满足 KB，返回 `True`，意思不是 query 为真，而是这个模型与已知世界不相容，不构成反例。
    

模型检查简单且可靠，但时间复杂度随命题数指数增长：

[  
O(2^n)  
]

## 6. Knowledge Engineering（知识工程）

知识工程是把自然语言中的事实与规则准确编码成逻辑公式的过程。真正困难的地方常不是推理算法，而是：

- 应该设计哪些命题符号？
    
- “至少一个”和“恰好一个”如何表示？
    
- 一条线索究竟排除了一个组合，还是排除了每个元素？
    

### 6.1 “至少一个”不等于“恰好一个”

三名嫌疑人中至少一人作案：

$[$  
$A\lor B\lor C$  
$]$

若要求恰好一人，还必须加入“两两不能同时为真”：

$[$  
$(A\lor B\lor C)$  
$\land(\neg A\lor\neg B)$  
$\land(\neg A\lor\neg C)$  
$\land(\neg B\lor\neg C)$  
$]$

### 6.2 Clue 例子中的易错点

若“Scarlet 在 library 用 wrench”这个完整猜测是错的，正确表示是：

$[$  
$\neg(Scarlet\land Library\land Wrench)$  
$]$

由德摩根律：

$[$  
$\neg Scarlet\lor\neg Library\lor\neg Wrench$  
$]$

它只说明三者中至少一个是错的，不能直接推出三者都错。


## 7. 常用推理规则与等价变换

### 7.1 Modus Ponens（肯定前件）

$[$  
$\frac{P\rightarrow Q,\quad P}{Q}$  
$]$

知道“下雨则在室内”以及“下雨”，可推出“在室内”。

### 7.2 And Elimination（合取消去）

$[$  
$\frac{P\land Q}{P}$  
$\qquad$  
$\frac{P\land Q}{Q}$  
$]$

### $7.3 Double Negation（双重否定）$

$[$  
$\neg\neg P\equiv P$  
$]$

### 7.4 Implication Elimination（消去蕴含）

$[$  
$P\rightarrow Q\equiv\neg P\lor Q$  
$]$

注意错误逆推：从 `P → Q` 和 `Q` 不能推出 `P`，这叫**肯定后件谬误**。

合法的逆否推理是：

$[$  
$P\rightarrow Q\equiv\neg Q\rightarrow\neg P$  
$]$

### 7.5 Biconditional Elimination（消去双条件）

$[$  
$P\leftrightarrow Q$  
$\equiv(P\rightarrow Q)\land(Q\rightarrow P)$  
$]$

### 7.6 De Morgan’s Laws（德摩根律）

$[$  
$\neg(P\land Q)\equiv\neg P\lor\neg Q$  
$]$

$[$  
$\neg(P\lor Q)\equiv\neg P\land\neg Q$  
$]$

### 7.7 Distributive Laws（分配律）

$[$  
$P\lor(Q\land R)\equiv(P\lor Q)\land(P\lor R)$  
$]$

$[$  
$P\land(Q\lor R)\equiv(P\land Q)\lor(P\land R)$  
$]$


## 8. 推理也可以看成搜索

|搜索问题组成|逻辑推理中的对应项|
|---|---|
|Initial state|初始知识库|
|Actions|可应用的推理规则|
|Transition model|应用规则后增加了句子的知识库|
|Goal test|待证明结论是否已出现|
|Path cost|证明使用的步骤数|

因此，证明一个命题本质上也可以看成在“可能的证明步骤空间”中寻找一条路径。

## 9. Resolution（归结）

### 9.1 Literal 与 Clause

- **Literal（文字）**：原子命题或其否定，例如 `P`、`¬P`。
    
- **Complementary literals（互补文字）**：`P` 与 `¬P`。
    
- **Clause（子句）**：若干文字的析取，例如 `P ∨ ¬Q ∨ R`。
    

### 9.2 归结规则

$[$  
$\frac{P\lor Q,\quad \neg P\lor R}{Q\lor R}$  
$]$

两个子句中 `P` 与 `¬P` 不可能同时为真，因此消去这对互补文字，合并剩余文字。

特殊情况：

$[$  
$\frac{P,\quad \neg P}{\square}$  
$]$

`□` 是空子句，永假，表示产生矛盾。


## CNF（合取范式）

CNF 是“若干子句的合取”：

$[$  
$(A\lor B\lor C)\land(D\lor\neg E)\land(F\lor G)$  
$]$

### 10.1 转换步骤

1. 消去双条件：  
    $[$  
    $\alpha\leftrightarrow\beta$  
    $\Rightarrow(\alpha\rightarrow\beta)\land(\beta\rightarrow\alpha)$  
    $]$
    
2. 消去蕴含：  
    $[$  
    $\alpha\rightarrow\beta\Rightarrow\neg\alpha\lor\beta$  
    $]$
    
3. 用德摩根律把否定推进到原子命题前；
    
4. 消去双重否定；
    
5. 用分配律将 `∨` 分配到 `∧` 上

### 10.2 转换例子

将 `(P ∨ Q) → R` 转成 CNF：

$[$  
$(P\lor Q)\rightarrow R$  
$]$

$[$  
$\equiv\neg(P\lor Q)\lor R$  
$]$

$[$  
$\equiv(\neg P\land\neg Q)\lor R$  
$]$

$[$  
$\equiv(\neg P\lor R)\land(\neg Q\lor R)$  
$]$

最后一行是 CNF。


## 11. 归结反证法

要证明：

$[$  
$KB\models\alpha$  
$]$

等价于证明：

$[$  
$KB\land\neg\alpha$  
$]$

不可满足。

算法流程：

1. 将 `KB ∧ ¬α` 转为 CNF；
    
2. 不断选择含互补文字的两个子句进行归结；
    
3. 把新子句加入集合；
    
4. 若得到空子句 `□`，产生矛盾，所以 `KB ⊨ α`；
    
5. 若再也无法生成新子句且没有空子句，则不能由 KB 蕴涵 α。
    

### 11.1 例子

判断：

$[$  
$(A\lor B)\land(\neg B\lor C)\land\neg C\models A$  
$]$

加入待证结论的否定 `¬A`：

1. `¬C` 与 `¬B ∨ C` 归结得到 `¬B`；
    
2. `¬B` 与 `A ∨ B` 归结得到 `A`；
    
3. `A` 与假设 `¬A` 归结得到空子句 `□`；
    
4. 因此原 KB 必然蕴涵 `A`。
## 12. First-Order Logic（一阶逻辑）

命题逻辑需要为每个具体事实创建独立符号。在对象和关系较多时，符号数量会爆炸。一阶逻辑引入：

- **Constant（常量）**：具体对象，如 `Minerva`、`Gryffindor`；
    
- **Predicate（谓词）**：对象的性质或对象间关系；
    
- **Variable（变量）**：`x`、`y`；
    
- **Quantifier（量词）**：`∀`、`∃`。
    

例子：

```
Person(Minerva)
House(Gryffindor)
BelongsTo(Minerva, Gryffindor)
```

### 12.1 全称量词 `∀`

$[$  
$\forall x; Person(x)\rightarrow Mortal(x)$  
$]$

表示“对所有 x，如果 x 是人，则 x 会死亡”。

注意括号和作用域：通常应写成

$[$  
$\forall x;(Person(x)\rightarrow Mortal(x))$  
$]$

### 12.2 存在量词 `∃`

$[$  
$\exists x;(House(x)\land BelongsTo(Minerva,x))$  
$]$

表示“至少存在一座 Minerva 所属的学院”。

### 12.3 两种量词顺序不能互换

$[$  
$\forall x\exists y;Loves(x,y)$  
$]$

表示“每个人都爱某个人”，不同的人可以爱不同对象。

$[$  
$\exists y\forall x;Loves(x,y)$  
$]$

表示“存在同一个人，所有人都爱他/她”。两者含义完全不同。

## 13. 核心概念对比

|概念|回答的问题|
|---|---|
|Syntax|这个公式怎样合法书写？|
|Semantics|这个公式在一个模型中是真是假？|
|Model|世界中每个命题的真值如何？|
|KB|智能体接受哪些句子为真？|
|Entailment|KB 为真时，结论是否必然为真？|
|Model Checking|枚举模型检验蕴涵|
|Inference Rule|从某些句子合法推出新句子|
|Resolution|通过消去互补文字生成新子句|
|CNF|适合归结算法处理的标准形式|
|First-Order Logic|用对象、谓词和量词紧凑表达知识|