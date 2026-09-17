# AlphaProof Matchmaker

## 1. 先说结论

先把两个最容易混淆的问题说清楚。

### Matchmaker 在哪

**Matchmaker 不只是 TTRL 里才有。**

AlphaProof 的 **Main RL 本身就有 Matchmaker**，而且它是 Main RL 三个核心组件之一：

$$
\boxed{
\text{Matchmaker}
+
\text{Distributed Actors}
+
\text{Learner}
}
$$

Main RL 训练完成后，遇到特别困难的测试题，AlphaProof 会进行 **TTRL（Test-Time Reinforcement Learning）**。TTRL 并没有换掉这套架构，而是重新运行一个更聚焦的 RL loop：

$$
\boxed{
\text{Main RL generalist}
\rightarrow
\text{TTRL specialist}
}
$$

TTRL 中依然存在：

- Matchmaker；
- distributed Actors；
- tree search；
- Lean；
- replay / Learner；
- policy / value 更新。

区别主要是**训练题库变了**：

- Main RL：约 8000 万个通用 formal problems；
- TTRL：当前 target $T$ 加上围绕它生成的 variants $V_T$。

所以准确说法是：

> **Matchmaker 是 AlphaProof RL 基础设施的一部分，Main RL 和 TTRL 都使用。**

另外，你这里写的“TD3RL”应该是 **TTRL**；AlphaProof 原文叫 **Test-Time Reinforcement Learning**。

---

## 2. 系统位置

Matchmaker 可以理解为 AlphaProof 的**题目级中央调度器**。

整体结构是：

    Problem Pool
         ↓
      Matchmaker
         ↓
       Actor
         ↓
    Tree Search
         ↓
        Lean
         ↓
    proof / disproof / failure
         ↓
      Matchmaker
       ↘
        Replay / Learner

其中 Matchmaker 主要决定三件事：

1. **下一次搜索哪道题；**
2. **这次尝试 prove 还是 disprove；**
3. **这次搜索给多少 simulation budget。**

因此它不只是一个：

> “哪些题 interesting、哪些题优先做”

的 scheduler。

它同时还是：

> **compute allocator**

也就是算力分配器。

可以概括成：

$$
\boxed{
\text{Which problem}
+
\text{How often}
+
\text{How much compute}
}
$$

Matchmaker 不负责：

- 生成 Lean tactic；
- 计算 proof-state value；
- 决定 MCTS 树内部下一步走哪里；
- 做 gradient update。

这些分别属于 Proof Network、Tree Search 和 Learner。

---

## 3. 两个阶段

### Main RL

Main RL 是 AlphaProof 的主要通用强化学习阶段。

训练 curriculum 主要来自：

> 约 100 万道自然语言数学问题  
> $\rightarrow$ auto-formalization  
> $\rightarrow$ 约 8000 万个 Lean formal problems。

此外还有约 3500 个人工形式化问题。

Main RL 训练约：

> **100 万 learner steps**

总训练计算约：

> **80,000 TPU-days**

在这里 Matchmaker 面对的是一个巨大而且难度高度不均匀的问题池。

它必须不断决定：

> 当前模型此时最值得在哪些问题上产生新的 RL experience？

---

### TTRL

对于 Main RL generalist 很难解决的目标 $T$，AlphaProof 会在测试阶段生成 target-specific variants：

$$
V_T=\{v_1,v_2,\ldots\}
$$

然后建立：

$$
C_T=\{T\}\cup V_T
$$

再运行一个 Focused RL loop。

这里仍然是：

    Matchmaker
         ↓
      Actors
         ↓
    Tree Search
         ↓
       Lean
         ↓
      Learner

只是 Matchmaker 不再面对 8000 万通用题，而是面对：

> **目标题 $T$ + 与它相关的 variants。**

所以：

> **TTRL 不是新增一个 Matchmaker，而是复用 Main RL 的 Matchmaker 机制，并使用 TTRL-specific hyperparameters。**

---

## 4. 调度依据

Matchmaker 会为每道 problem 保存最近若干次 attempts 的 outcomes。

例如：

    failure
    failure
    success
    failure
    success

论文说：

> problem 的 interestingness 根据最近 $N$ 次 attempts 决定。

这里的 $N$ 是一个 **recent-history window**。

也就是说 Matchmaker 更关心：

> **当前模型最近做这道题是什么表现**

而不是从训练第一天开始的所有历史结果。

这是合理的，因为 Proof Network 一直在更新：

$$
\theta_0
\rightarrow
\theta_1
\rightarrow
\theta_2
\rightarrow
\cdots
$$

某道题可能经历：

    训练早期
    一直失败
        ↓
    训练中期
    有时成功
        ↓
    训练后期
    稳定成功

很早以前 $\theta_0$ 的失败，并不能很好代表当前 $\theta_t$ 的能力。

需要注意：

> “使用最近 $N$ 次 attempts”是论文明确写出的机制；

但“这是为了处理模型不断变化的 non-stationarity”属于对设计动机的合理解释，论文没有单独证明这一点。

---

## 5. Trust Count

这是之前最容易混淆的数字。

答案是：

> **Main RL 的 `trust_count = 8`；TTRL 的 `trust_count = 5`。**

不是一个统一数字。

AlphaProof Supplementary Table 7 给的是：

| 阶段 | `trust_count` |
|---|---:|
| Main RL | **8** |
| TTRL | **5** |

所以：

$$
\boxed{
\text{Main RL}: 8
\qquad
\text{TTRL}: 5
}
$$

`trust_count` 的作用是：

> **在对一道题下比较强的判断之前，至少先积累一定数量的尝试证据。**

例如 Main RL 中：

    attempt 1: failure
    attempt 2: failure

不能马上得出：

> “这题模型不会。”

因为单次 tree search 本身有随机性，而且搜索预算有限。

因此 attempts 少于 8 时，这道题仍然属于 highly interesting 的情况之一。

TTRL 中这个阈值更小：

> `trust_count = 5`

因为 TTRL 是更加集中的 target-specific RL。

---

## 6. 优先级

论文明确列出了 highly interesting 的三种主要情况。

### 从未尝试

    history = empty

系统完全不知道当前模型对这道题的能力，所以需要 exploration。

---

### 尝试太少

如果：

$$
\text{attempts}<\text{trust\_count}
$$

仍然保持较高 priority。

也就是：

- Main RL：少于 8 次；
- TTRL：少于 5 次。

---

### 成败混合

如果已经进行了足够多 attempts，但最近历史中：

    success
    failure
    success
    failure

说明模型：

> 已经有一定能力，但还不能稳定解决。

这正是最值得继续训练的一类题。

可以把它理解为当前模型的：

> **learning frontier**

---

相反，两种情况会降低 priority。

第一种是：

    failure
    failure
    failure
    ...

经过足够 attempts 仍持续无法解决。

这说明当前模型可能暂时够不到。

第二种是：

    success
    success
    success
    ...

已经连续稳定解决很多次。

说明这道题已经基本掌握，再继续投入大量训练计算的收益有限。

因此 Matchmaker 形成了一种动态课程：

    太难
     ↓
    少做

    正在学会
     ↓
    多做

    已经掌握
     ↓
    少做

---

## 7. 搜索预算

Matchmaker 的另一个核心功能就是：

> **决定一次 attempt 的 simulation budget。**

这一点是 AlphaProof 原文明说的，不只是对它的推测。

一次 attempt 可以理解为：

    problem
    +
    prove / disprove
    +
    simulation budget
    ↓
    Actor

其中 simulation budget 指：

> **这一次 tree search 最多允许使用多少个 search simulations。**

它不是：

- token 数；
- tactic 数；
- 独立 proof attempts 数。

一次 simulation 大致是：

    从 root 开始
        ↓
    搜索树 selection
        ↓
    到达待扩展 state
        ↓
    Proof Network 给 policy / value
        ↓
    Lean 执行 tactic
        ↓
    产生新 state
        ↓
    回传搜索统计

然后再开始下一次 simulation。

---

Main RL 的初始搜索预算是：

> **250 simulations**

如果这道题最近 repeatedly fails，下一次 attempt 的 budget 会按乘法增加。

官方参数为：

- initial simulations：**250**
- failure multiplier：**1.17**
- maximum simulations：**16,000**

可以概念化理解成：

$$
B(F)
=
\min
\left(
16000,
250\times1.17^F
\right)
$$

其中 $F$ 表示最近窗口中的 failure 数量。

论文没有公开具体整数 rounding 的实现，因此这个公式主要用于表达机制。

---

TTRL 使用的是另一套更激进的参数：

- initial simulations：**125**
- failure multiplier：**2**

于是大致是：

    125
     ↓
    250
     ↓
    500
     ↓
    1000
     ↓
    2000
     ↓
    ...

也就是：

$$
B(F)\approx125\times2^F
$$

Supplementary Table 7 没有单独给出 TTRL 的 simulation cap，因此不应该自行补一个最大值。

---

## 8. 两种控制

这里一定要区分：

> **interestingness / priority**

和：

> **simulation budget**

它们不是同一个量。

### Priority

回答的是：

> **这道题相比其他题，有多值得再次被选中？**

也就是控制：

> **多久做一次。**

---

### Budget

回答的是：

> **既然这次已经决定做它，这一次要搜索多深？**

也就是控制：

> **一次做多少。**

---

所以一题不断失败时，可能同时发生：

    repeated failures
          │
     ┌────┴─────┐
     ▼          ▼
 priority ↓   budget ↑

意思是：

> 这道题可能变得更少被抽到；  
> 但下一次真的抽到时，会给它更深的搜索。

这是 Matchmaker 很重要的设计。

所以不能简单理解成：

> “越难的题一直给越多资源。”

准确地说是：

> **它同时调整尝试频率和单次搜索深度。**

---

## 9. 调度流程

于是一次完整的 Matchmaker loop 可以写成：

    1. Actor 空闲

    2. Matchmaker 查看所有 problem statistics

    3. 根据 recent history 判断：
       - unseen
       - attempts too few
       - mixed success
       - repeatedly unsolved
       - stably proved
       - disproved

    4. 给问题确定 scheduling priority

    5. 选择下一道 problem

    6. 分配 prove / disprove objective

    7. 根据 recent failures
       决定 simulation budget

    8. 把：
       problem
       objective
       simulation budget
       发送给 Actor

    9. Actor 使用当前 Proof Network
       运行 tree search

    10. Lean 返回：
        proof
        disproof
        或 budget exhausted

    11. outcome 返回 Matchmaker

    12. Matchmaker 更新该题 history

    13. 成功 proof / disproof trajectory
        进入 Replay / Learner

    14. Learner 更新 Proof Network

    15. 下一轮重新调度

因此 AlphaProof 同时存在两个反馈环：

    problem outcome
        ↓
    Matchmaker
        ↓
    更新调度策略

和：

    successful trajectory
        ↓
    Replay / Learner
        ↓
    更新 Proof Network

两者不能混为一谈。

---

## 10. 成败处理

### Proof 成功

Proof 一次成功后：

> **不会立刻永久删除。**

因为一次成功不代表稳定掌握。

Main RL 中还有：

> `trust_count_proved = 12`

也就是需要观察连续稳定成功。

原因是 tree search 本身具有随机性：

- tactic sampling 不同；
- branch exploration 不同；
- simulation budget 有限；
- Proof Network 还在继续更新。

所以可能出现：

    success
    failure
    success

作者 Julian Schrittwieser 还解释过：

> 已经证明成功的问题继续尝试，有时还能找到更好或更短的 proof。

---

### Disproof 成功

Disproof 则不同。

Main RL 的：

> **disprove rate = 50%**

也就是说 Matchmaker 会随机要求 Actor：

    prove P

或者：

    disprove P

原因是约 8000 万题中大量来自 auto-formalization，并不能保证 statement 为真。

如果 Lean 已经成功验证：

$$
\neg P
$$

那么这个 statement 已经确定为 false。

因此：

> **successfully disproved statement 不再重试。**

Main RL 中对应 priority weight：

> **0**

所以：

    proof success once
    → 以后仍可能再做

    disproof success
    → 不再做

---

### Search 失败

如果 Actor 没找到 proof / disproof：

> failure 不会作为成功 RL trajectory 送给 Learner。

但 failure 对 Matchmaker 仍然很重要，因为它会影响：

1. 后续 priority；
2. 后续 simulation budget。

所以：

    failed attempt
        │
        ├─ 不直接训练 Proof Network
        │
        └─ 更新 Matchmaker statistics

---

## 11. 参数对比

把最关键参数集中起来看最清楚。

| 参数 | Main RL | TTRL |
|---|---:|---:|
| Matchmaker | **有** | **有** |
| Curriculum | 通用题库 | Target + variants |
| `trust_count` | **8** | **5** |
| `trust_count_proved` | **12** | 表中未单列 |
| Initial simulations | **250** | **125** |
| Failure multiplier | **1.17** | **2** |
| Simulation cap | **16,000** | 表中未单列 |
| Interesting weight | **1.0** | 表中未单列 |
| Undecided weight | **0.1** | 表中未单列 |
| Fully proved weight | **0.001** | 表中未单列 |
| Disproved weight | **0.0** | 表中未单列 |
| Disprove rate | **50%** | 表中未单列 |

因此最容易记错的两个数字应该直接记成：

$$
\boxed{
\text{trust\_count: Main RL}=8
}
$$

$$
\boxed{
\text{trust\_count: TTRL}=5
}
$$

而搜索预算则是：

$$
\boxed{
\text{Main RL}:250,\times1.17,\text{cap}=16000
}
$$

$$
\boxed{
\text{TTRL}:125,\times2
}
$$

---

## 12. TTRL 停止

TTRL 还有一个 Main RL 没有的 target-level 停止条件。

设真正目标为：

$$
T
$$

variants 为：

$$
V_T
$$

Focused RL 的 curriculum 是：

$$
C_T=\{T\}\cup V_T
$$

一旦 $T$ 获得 Lean-verified proof：

> Matchmaker 就停止给 $T$ 和它的 variants $V_T$ 分配新的 attempts。

即：

$$
T\text{ solved}
\Rightarrow
\text{stop scheduling }\{T\}\cup V_T
$$

原因是 TTRL 的目标不是把所有 variants 做完，而是：

> **借助 variants 训练 specialist，最终突破真正目标 $T$。**

---

## 13. 组件边界

Matchmaker 和其他组件的职责可以这样区分：

| 组件 | 负责什么 |
|---|---|
| Matchmaker | 选题、prove/disprove、simulation budget |
| Actor | 执行一次完整 attempt |
| Tree Search | 在当前 proof tree 中探索哪里 |
| Policy | 提议值得尝试的 tactic |
| Value | 判断当前 proof state 的前景 |
| Lean | 验证 tactic 和最终 proof |
| Learner | 用成功 experience 更新模型 |

所以尤其不要混淆：

### Matchmaker vs Value

Value 判断：

> **当前 proof state 有多有希望？**

Matchmaker 判断：

> **这道 theorem 现在值不值得继续投入训练资源？**

一个是：

> proof-state level

一个是：

> problem level。

---

## 14. 作用边界

从机器学习视角，Matchmaker 很像一种：

> **online adaptive curriculum**

传统 curriculum 可能预先规定：

    easy
     ↓
    medium
     ↓
    hard

AlphaProof 则根据当前模型的真实表现动态判断：

    从未尝试
    → exploration

    尝试太少
    → 继续收集证据

    success / failure 混合
    → learning frontier
    → 高优先级

    长期失败
    → 当前可能太难

    连续稳定成功
    → 当前已经太容易

而随着模型更新，同一道题会不断在这些状态之间移动。

它也和 Multi-Armed Bandit 的资源分配思想有相似之处，但论文没有说 Matchmaker 使用 UCB、Thompson Sampling 等标准 Bandit 算法。

因此更准确的描述是：

> **基于 recent attempt history、threshold 和 priority weight 的在线调度 heuristic。**

另外，论文没有提供：

> “去掉 Matchmaker 后性能下降多少”

这样的独立 ablation。

所以不能把 AlphaProof 最终性能提升中的某个百分比单独归因于 Matchmaker。

---

## 15. 最后总结

Matchmaker 的完整作用可以浓缩成：

    Problem Pool
         ↓
    recent history
         ↓
    interestingness
         ↓
    决定哪些题优先
         ↓
    选择 problem
         ↓
    prove / disprove
         ↓
    根据 failures
    决定 simulation budget
         ↓
       Actor
         ↓
    Tree Search
         ↓
       Lean
         ↓
    outcome 返回
         ↓
    更新 history
         ↓
      下一轮

最关键的三点是：

> **第一，Matchmaker 不只是 TTRL 才有，Main RL 本身就有，TTRL 复用了同一套 Matchmaker 架构。**

> **第二，`trust_count` 不是统一数字：Main RL 是 8，TTRL 是 5。**

> **第三，Matchmaker 不只决定“哪些题先做、哪些题后做”，还明确负责给每次 Actor attempt 分配 adaptive simulation budget。**

因此最准确的一句话定义是：

> **AlphaProof 的 Matchmaker 是 Main RL 与 TTRL 共用的 problem-level 在线课程调度器和算力分配器：它根据每道题最近的成功/失败历史动态决定题目 priority，同时根据近期 failures 调整该题下一次 tree search 的 simulation budget，从而同时控制“做哪题、多久做一次、一次做多深”。**

## 16. 来源

主要依据：

1. Hubert et al., *Olympiad-level formal mathematical reasoning with reinforcement learning*：
   - Main RL；
   - Matchmaker System；
   - Actor Experience Generation；
   - Focused RL / TTRL。

2. AlphaProof Supplementary Information：
   - Supplementary Table 6；
   - Supplementary Table 7。

3. Julian Schrittwieser 的 AlphaProof 技术说明：
   - 小预算起步；
   - successive attempts 增加 compute；
   - 成功题仍可能继续搜索更好、更短的 proof。