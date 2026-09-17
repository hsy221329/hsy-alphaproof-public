
# 09｜Matchmaker 与 nanoproof

本文重点讨论 **nanoproof 如何复现 AlphaProof 的 Matchmaker**。AlphaProof 本身的 Matchmaker 原理前文已经介绍过，这里只保留理解 nanoproof 所必需的部分。

先说结论：

> **nanoproof 的 Matchmaker 就是一套普通 Python 调度程序，不是神经网络。**
>
> 它不给题目读语义、也不训练一个“选题模型”，而是给每道题维护一份搜索历史，然后用固定规则决定：
>
> 1. 这道题下一次被抽中的概率有多大；
> 2. 如果抽中了，这一次给多少 tree-search simulations。

因此它复现的是 AlphaProof 的核心思想：

$$
\text{题目历史}
\rightarrow
\text{状态分类}
\rightarrow
\text{固定权重}
\rightarrow
\text{加权随机选题}
\rightarrow
\text{搜索预算}
$$

但 nanoproof 不是 DeepMind 原代码，而且在历史窗口、prove/disprove、异常处理等地方都有自己的简化和改动。

---

## 1. 原版规则

理解 nanoproof 前，先把 AlphaProof 的三个量分开。

### `trust_count`

它回答的是：

> **这道题是不是已经尝试得足够多，可以开始比较相信对它的判断？**

官方参数：

| 阶段 | `trust_count` |
|---|---:|
| Main RL | **8** |
| TTRL | **5** |

因此 Main RL 中，一道题如果只试了 1、2、3 次，即使全失败，也不会太早判定它“不会”。

而且：

> **超过 `trust_count` 并不意味着自动降权。**

超过阈值以后还要看结果：

- 一直失败：降低 priority；
- 有成功也有失败：仍然 highly interesting；
- 连续稳定成功：降低 priority。

所以 `trust_count` 更像：

> **“至少先观察多少次，再相信这个难度判断。”**

---

### 最近 $N$ 次

AlphaProof 原文还有另一个量：

> **last $N$ attempts**

它与 `trust_count` 不是同一个参数。

原文用最近 $N$ 次判断：

- 最近是否 success / failure 混合；
- 最近有多少 failures；
- 下一次 simulation budget 应增加多少。

但是：

> **公开论文没有给出 $N$ 的具体数值。**

因此不能写成：

$$
N=8
$$

也不能因为 TTRL 的 `trust_count=5` 就写：

$$
N=5
$$

AlphaProof 中应该理解成：

$$
\boxed{
\text{trust\_count}\neq N
}
$$

---

### 稳定成功

AlphaProof 还有：

> `trust_count_proved`

Main RL 中为：

> **12**

它回答：

> 连续成功多少次以后，可以认为这道题已经比较稳定地掌握？

因此 AlphaProof 原版至少有三种不同尺度：

| 参数 | 作用 |
|---|---|
| `trust_count` | 尝试多少次以后才开始相信难度判断 |
| $N$ | 观察最近多少次的 success / failure |
| `trust_count_proved` | 连续成功多少次算基本掌握 |

这三个概念不能混成一个“窗口长度”。

---

## 2. 失败处理

如果一道题长期失败，AlphaProof 并不是简单地：

> “失败越多 → 越优先做。”

它会同时产生两个方向不同的效果：

$$
\text{持续失败}
\Rightarrow
\begin{cases}
\text{选题 priority 下降}\\
\text{单次 search budget 上升}
\end{cases}
$$

也就是：

> **少抽一点，但抽中了就搜得更深。**

这解释了为什么 interestingness 和 simulation budget 必须分开理解。

nanoproof 基本保留了这个思想，但具体历史规则并不完全一样。

---

# 3. nanoproof

## 3.1 它是什么

nanoproof 的 Matchmaker 本质就是一个 **Python scheduler**。

它内部没有：

- Transformer；
- Encoder-Decoder；
- MLP；
- learned ranking model；
- gradient update。

它做的事情非常朴素：

> **为每道 theorem 保存一份历史记录，然后根据这份记录计算抽样权重和搜索预算。**

因此可以把它想象成给每道题建立一本账：

    题 A：
    没试过

    题 B：
    成功
    失败
    成功

    题 C：
    失败
    失败
    失败
    失败

    题 D：
    连续成功很多次

Matchmaker 每次 Actor 空闲时，就重新查看这些记录，然后决定下一道题。

---

## 3.2 是否是分类器

从算法本质上，你完全可以把 nanoproof 的 Matchmaker 理解成：

> **一个手写规则分类器 + 一张固定权重表。**

例如它并不会学习一个函数：

$$
w=f_\phi(\text{theorem},\text{history})
$$

也不会把 theorem 输入另一个神经网络，让模型预测：

> “这题 interestingness = 0.734。”

它实际上更接近：

    查看这道题的历史
          ↓
    满足哪一组规则？
          ↓
    分到哪个状态
          ↓
    查这个状态对应的固定 weight

例如简化后可以理解成：

| 当前状态 | nanoproof 默认权重 |
|---|---:|
| 未充分探索 | **1** |
| 多次尝试但从未成功 | **0.1** |
| 有成功但尚未稳定 | **1** |
| 已稳定成功 | **0.001** |
| 特殊异常题 | **0** |

因此权重确实非常接近一种：

> **table lookup / 查表值。**

不是每道题产生一个连续、精细、学习出来的 interestingness score。

例如：

    A：只试过两次，全失败
       ↓
    “尚未充分探索”
       ↓
    weight = 1

    B：已经试过很多次，从未成功
       ↓
    “currently unsolved”
       ↓
    weight = 0.1

    C：有成功也有失败
       ↓
    “仍然值得训练”
       ↓
    weight = 1

    D：最近连续稳定成功
       ↓
    “mastered”
       ↓
    weight = 0.001

接下来才使用这些 weights 做：

> **weighted random sampling。**

所以它不是严格按照：

    A > B > C > D
    → 永远选 A

而是：

$$
P(P_i)
=
\frac{w_i}{\sum_jw_j}
$$

权重越大，越容易抽中。

---

## 3.3 是否符合原版

这套设计**非常符合 AlphaProof 公开材料中 Matchmaker 的本意**。

AlphaProof 原文描述的也不是一个 learned Matchmaker model，而是一套根据 problem attempt history 判断 interestingness 的规则：

- 没尝试过：highly interesting；
- attempts 少于 `trust_count`：highly interesting；
- 已经尝试充分，但最近成功失败混合：highly interesting；
- 持续未解决：降低 priority；
- 连续 `trust_count_proved` 次成功：降低 priority；
- 成功 disprove：不再尝试。

Supplementary Table 7 又给这些状态配了类似：

> **1、0.1、0.001、0**

这样的离散 priority weights。

因此从**公开算法层面**，AlphaProof Matchmaker 完全可以概括成：

$$
\boxed{
\text{规则分类}
+
\text{固定权重}
+
\text{加权随机采样}
+
\text{自适应搜索预算}
}
$$

也就是说，它和 nanoproof 的总体思想确实非常接近。

可以想象成：

    theorem history
         ↓
    一组 if / else 规则
         ↓
    interesting
    undecided
    mastered
    disproved
         ↓
    1
    0.1
    0.001
    0
         ↓
    weighted random sampling

然后再单独根据近期 failures 决定：

> 这一次搜索给多少 simulations。

所以如果用非常通俗的话说：

> **AlphaProof 的 Matchmaker 并不是另一个聪明的大模型，而更像一个根据“最近考试成绩”给题目分组，再按照预先设置好的权重决定下一次练什么题的调度程序。**

---

### 需要保留的边界

不过这里不能进一步说：

> **“DeepMind 生产版 Matchmaker 就一定只是一个简单 Python `if-else` 脚本。”**

原因是：

> AlphaProof 的完整生产代码没有公开。

我们能确认的是：

1. 论文公开的算法描述是 rule-based；
2. Supplementary 给出了固定超参数和 priority weights；
3. 官方提供的高层伪代码也是 Python 形式的普通规则程序；
4. 没有公开一个需要训练的 Matchmaker neural network。

所以最稳妥的说法是：

> **从 AlphaProof 公开的算法层面看，它本质上就是一个 history-based rule scheduler，可以近似理解成“规则分类器 + 查表权重”；但生产系统在 8000 万题和大量 Actors 上如何高效维护、缓存、采样这些状态，是没有公开的工程细节。**

另外还要注意：

> AlphaProof 并不是简单地“固定看最近 8 次”。

因为：

- `trust_count=8` 是 Main RL 的探索可信阈值；
- `trust_count=5` 是 TTRL 的对应阈值；
- 最近窗口 $N$ 是另一个参数，而且公开值未知；
- `trust_count_proved` 又是另一个稳定掌握阈值。

所以“规则分类器”这个理解是对的，但具体输入规则不是单一一个固定窗口。

---

# 4. 初始化

一次全新训练开始时，nanoproof 大致做三件事：

1. 加载训练题；
2. 给每道题建立空的历史记录；
3. 加载一套固定 Matchmaker 参数。

默认主要参数是：

| 参数 | nanoproof |
|---|---:|
| `trust_count` | **4** |
| `trust_count_proved` | **6** |
| normal weight | **1** |
| unsolved weight | **0.1** |
| mastered weight | **0.001** |
| 初始 simulations | **64** |
| failure multiplier | **1.5** |
| simulation cap | **1024** |

注意：

> **nanoproof 的 `trust_count` 是 4，不是 AlphaProof Main RL 的 8，也不是 TTRL 的 5。**

这是社区复现自己选择的默认参数。

---

## 4.1 初始选题

刚开始时：

> 所有题都没有历史。

因此每道题的初始权重都是：

$$
w=1
$$

例如有四道题：

| 题目 | 权重 |
|---|---:|
| A | 1 |
| B | 1 |
| C | 1 |
| D | 1 |

那么第一次选择实际上就是：

> **等概率随机抽题。**

即：

$$
P(A)=P(B)=P(C)=P(D)=25\%
$$

所以训练刚开始时：

> **因为系统对所有题都不了解，所以先均匀随机探索。**

之后随着历史产生，各题权重才逐渐分化。

它不是先排序然后永远做最高分题，而是：

> **所有题参与加权随机抽样。**

因此低权重题只是更少出现，不一定完全消失。

---

# 5. Trust Count

nanoproof 默认：

$$
\text{trust\_count}=4
$$

含义和 AlphaProof 的基本思想一致：

> 在有效 attempts 还不到 4 次以前，不要过早判断这道题“太难”。

因此：

    0 次尝试
    → weight = 1

    1 次失败
    → weight = 1

    2 次失败
    → weight = 1

    3 次失败
    → weight = 1

即使前三次全部失败：

> **仍然保持正常高优先级。**

因为证据还不够。

---

## 5.1 超过阈值

这里非常重要：

> **尝试次数达到 4 并不意味着自动降低优先级。**

达到 `trust_count` 后，nanoproof 才开始根据历史进一步分类。

### 从未成功

例如：

    U U U U

其中 `U` 表示此次搜索没有证明成功。

此时：

> 已经有足够尝试，而且全历史从未找到 proof。

于是：

$$
w:1\rightarrow0.1
$$

也就是：

> **当前看起来比较难，少抽一些。**

---

### 曾经成功

例如：

    S U S U

其中 `S` 表示成功 proof。

虽然已经尝试 4 次，但它并不是“一直做不出来”。

所以：

$$
w=1
$$

仍然保持正常高优先级。

这种题很像 AlphaProof 所说的：

> **有时成功、有时失败的 learning-frontier problem。**

因此：

$$
\boxed{
\text{attempts}>4
\not\Rightarrow
\text{priority下降}
}
$$

真正决定是否降权的是：

> **历史属于哪一种状态。**

---

# 6. 三种历史

这是 nanoproof 和 AlphaProof 最值得区分的地方。

nanoproof **并没有一个统一的“最近 $N$ 次窗口”控制所有判断**。

它实际上用了不同范围。

### 探索阈值

默认：

$$
\text{trust\_count}=4
$$

用途是：

> 判断是否已经积累足够 attempts。

少于 4 次：

$$
w=1
$$

先继续探索。

---

### 掌握窗口

默认：

$$
\text{trust\_count\_proved}=6
$$

如果最近 **6 次有效 attempts 全部成功**：

    S S S S S S

那么认为：

> 这道题已经比较稳定地掌握。

于是：

$$
w=0.001
$$

所以 nanoproof 判断“最近一直成功、已经学会”时，看的是：

> **最近 6 次，而不是最近 4 次。**

---

### 全部历史

nanoproof 还有一些判断会查看**全部历史**。

最重要的是：

> **这道题历史上到底有没有成功 proof 过？**

例如：

    S U U U U U U U U U

虽然最近已经连续失败很多次，但因为历史上成功过：

> 它不会进入“从来没解决”的 `0.1` 分支。

如果又没有最近连续 6 次成功，那么通常：

$$
w=1
$$

也就是说：

> **nanoproof 中“最近一直失败”并不一定导致 sampling priority 下降。**

这取决于它以前有没有成功过。

---

# 7. 权重规则

忽略少数工程特判后，nanoproof 的核心规则可以整理成：

| 状态 | 默认权重 |
|---|---:|
| 没尝试过 | **1** |
| 有效尝试少于 4 次 | **1** |
| 至少 4 次且全历史从未成功 | **0.1** |
| 最近 6 次全部成功 | **0.001** |
| 有成功也有失败、尚未稳定 | **1** |

所以可以记成：

    未充分探索
        ↓
       1

    一直不会
        ↓
      0.1

    有时会、有时不会
        ↓
       1

    已经稳定会
        ↓
     0.001

它体现了 AlphaProof 的基本思想：

> **两头少做，中间多做。**

即：

- 当前完全不会的题少做；
- 已经完全学会的题也少做；
- 处在学习边界的题重点做。

---

# 8. 最近失败

最近一直失败，到底是提高优先级还是降低优先级？

必须把：

- sampling priority；
- simulation budget

分开回答。

### 从未成功

例如：

    U U U U

已经达到 `trust_count=4`，而且历史从未成功。

那么：

$$
\text{sampling weight}=0.1
$$

即：

> **选题优先级降低。**

但与此同时，它最近失败很多，所以：

> **单次 simulation budget 增大。**

因此：

$$
\boxed{
\text{少抽，但抽中后多搜}
}
$$

---

### 以前成功

例如：

    S U U U U U U

它以前至少成功过一次。

那么 nanoproof 通常不会把它归入“从未解决”的低权重组：

$$
w=1
$$

所以：

> **sampling priority 仍然可以保持正常。**

但是因为最近 failures 很多：

> simulation budget 仍会上升。

因此 nanoproof 的真实规则不是：

> “最近失败越多 → 权重越低。”

而是：

> **recent failures 主要直接控制搜索预算；sampling weight 还取决于整段历史有没有成功过。**

---

# 9. 搜索预算

nanoproof 也复现了 Matchmaker 的另一半功能：

> **adaptive simulation budget**

默认参数：

$$
B_0=64,\qquad m=1.5,\qquad B_{\max}=1024
$$

这里的窗口非常明确：

> **只看最近 `trust_count=4` 次有效 attempts。**

设最近 4 次中失败数量为 $F$，则：

$$
B=
\min
\left(
1024,
\left\lfloor64\times1.5^F\right\rfloor
\right)
$$

所以：

| 最近 4 次失败数 | simulations |
|---:|---:|
| 0 | **64** |
| 1 | **96** |
| 2 | **144** |
| 3 | **216** |
| 4 | **324** |

例如：

    S S S S
    → 64

    S U S U
    → 144

    U U U U
    → 324

因此：

> **越近期频繁失败，这一次就允许搜得更深。**

虽然配置中写着：

> simulation cap = 1024

但默认 `trust_count=4`，所以 $F$ 最大只有 4。

因此默认配置真正能达到的最大值是：

$$
64\times1.5^4=324
$$

所以：

> **默认设置下 1024 这个 cap 实际不会触发。**

---

# 10. 两个窗口

预算窗口和掌握窗口不要混淆。

nanoproof 默认：

$$
\boxed{
\text{预算窗口}=4
}
$$

因为 budget 看最近 `trust_count=4` 次。

而：

$$
\boxed{
\text{掌握窗口}=6
}
$$

因为 mastered 判断看最近 `trust_count_proved=6` 次。

比如：

    S S S S

此时：

- 最近 4 次没有失败；
- budget 回到 64；
- 但还没达到连续 6 次成功；

因此：

$$
w=1
$$

还没有被认为完全掌握。

直到：

    S S S S S S

才变为：

$$
w=0.001
$$

---

# 11. 完整流程

现在把 nanoproof 从头到尾串起来。

### 初始化

所有题：

    A: []
    B: []
    C: []
    D: []

全部：

$$
w=1
$$

所以开始时基本就是均匀随机探索。

---

### 加权抽题

每次 Actor 空闲：

> Matchmaker 根据每道题历史得到 weight，然后按这些 weights 随机抽题。

不是固定排序。

---

### 决定预算

题目抽中后，看最近 4 次有效 attempts。

根据 failures：

    64
    96
    144
    216
    324

确定这次搜索预算。

---

### Actor 搜索

Actor 使用当前 proof network：

> Tree Search + Lean

尝试证明这道题。

nanoproof 当前实现是：

> **prove-only**

不像 AlphaProof Main RL 那样还随机分配 prove / disprove。

---

### 写回结果

结果记录为：

- proven；
- unproven；
- error。

其中 `unproven` 只表示：

> **这一次搜索没有找到 proof。**

不意味着 theorem 是假的。

---

### 重新调度

假设一题连续失败：

    []

    weight = 1
    budget = 64

第一次失败：

    [U]

    weight = 1
    budget = 96

第二次失败：

    [U,U]

    weight = 1
    budget = 144

第三次失败：

    [U,U,U]

    weight = 1
    budget = 216

第四次失败：

    [U,U,U,U]

    weight = 0.1
    budget = 324

所以它会从：

> **经常抽、便宜搜**

逐渐变成：

> **少抽、但一次搜得更深。**

---

# 12. 成功案例

假设另一题变成：

    U U S U

因为历史上已经成功过：

$$
w=1
$$

最近 4 次里有 3 次失败：

$$
B=216
$$

于是：

> **它仍然经常被抽，而且得到较大的搜索预算。**

这种题正处于：

> 有一定能力，但还不稳定

的区域。

如果后来变成：

    S S S S S S

则：

$$
w=0.001
$$

而且：

$$
B=64
$$

系统认为：

> 基本已经掌握，没有必要继续大量投入训练计算。

---

# 13. 四题例子

| 题目 | 历史 | 权重 | Budget |
|---|---|---:|---:|
| A | 未尝试 | **1** | **64** |
| B | `S,U,S,U` | **1** | **144** |
| C | `S,S,S,S,S,S` | **0.001** | **64** |
| D | `U,U,U,U` | **0.1** | **324** |

于是：

**A：未知**

> 高频率、低成本探索。

**B：学习边界**

> 高频率、中等预算，是最值得继续训练的一类。

**C：已经掌握**

> 极少重试。

**D：目前太难**

> 低频率，但抽中了就给较大搜索预算。

所以 nanoproof 的思想可以概括成：

> **未知先探索；完全不会的少做但深搜；有时会的重点做；稳定会的基本退休。**

---

# 14. 两个特判

nanoproof 还有两个自己的工程特判。

### 一步证明

如果历史上出现过：

> 一步完成的 proof，

直接：

$$
w=0.001
$$

也就是说它认为这种题显然已经太容易，不必等待连续 6 次成功。

---

### 连续异常

如果运行连续出现异常：

$$
w=0
$$

防止 Actor 不断浪费资源。

这里：

- `unproven` = 正常运行但没找到证明；
- `error` = 程序、环境或 theorem 处理发生异常。

二者不是一个概念。

---

# 15. 两者对比

| 机制 | AlphaProof | nanoproof |
|---|---|---|
| Matchmaker 类型 | rule-based scheduler | Python rule-based scheduler |
| Learned model | 没有公开 | 没有 |
| 权重形式 | 离散 priority weights | 离散固定 weights |
| 未探索题 | high priority | **1** |
| `trust_count` | Main **8**；TTRL **5** | **4** |
| 最近窗口 | $N$，数值未公开 | 没有统一 $N$ |
| 稳定成功 | `trust_count_proved` | 最近 **6** 次 |
| 持续未解 | priority 降低 | 从未成功且至少 4 次 → **0.1** |
| mixed success | highly interesting | 通常 **1** |
| 预算窗口 | 最近 $N$ 次 failures | 最近 **4** 次 |
| 初始预算 | Main 250；TTRL 125 | **64** |
| Failure multiplier | Main 1.17；TTRL 2 | **1.5** |
| Prove / disprove | 有 | **prove-only** |
| Disproof 后退出 | 有 | 无 |
| TTRL variants | 有 | 未完整复现 |

所以两者在核心思想上非常接近：

$$
\boxed{
\text{history}
\rightarrow
\text{规则分类}
\rightarrow
\text{固定权重}
\rightarrow
\text{随机调度}
}
$$

再加上另一条：

$$
\boxed{
\text{recent failures}
\rightarrow
\text{adaptive search budget}
}
$$

但 nanoproof 把 AlphaProof 中未完全公开的部分做成了自己明确的工程选择，因此不能认为两者逐行等价。

---

# 16. 最后总结

理解 nanoproof Matchmaker，最简单的方法就是：

> **它不是模型，而是一套会记账的 Python 调度规则。**

一开始所有题都没有信息：

> **等权随机探索。**

默认先给每道题至少约 4 次机会。

之后根据历史把题分成几类：

    未探索充分
        ↓
    weight = 1

    从来做不出来
        ↓
    weight = 0.1

    有时做得出来
        ↓
    weight = 1

    连续稳定做出来
        ↓
    weight = 0.001

然后再根据近期 failures 单独决定：

> **本次搜索应该给多少 simulations。**

因此最准确的抽象是：

$$
\boxed{
\text{Matchmaker}
=
\text{规则分类器}
+
\text{权重表}
+
\text{加权随机采样}
+
\text{自适应预算}
}
$$

这个抽象不仅很适合 nanoproof，也基本符合 AlphaProof 公开材料所描述的 Matchmaker 本意。

不过对 AlphaProof 本身必须保留一句边界：

> **我们知道公开算法是这种 rule-based scheduler，但 DeepMind 没公开完整生产代码，因此不能进一步断言其 8000 万题生产系统在工程上就只是一个简单 Python `if-else` 脚本。**
