# Replay Buffer

Replay Buffer 是 AlphaProof **Main RL 中保存成功搜索经验的经验池**，位于 Actor 搜索与 Learner 更新之间：

$$
\text{Policy / Value}
\rightarrow
\text{Search}
\rightarrow
\text{Lean}
\rightarrow
\text{Replay}
\rightarrow
\text{Learner}
\rightarrow
\text{新 Policy / Value}
$$

它的作用不是简单“保存历史数据”，而是把搜索发现的有效证明转化为可重复学习的数据，使搜索能力逐渐被神经网络内化。

> **MCTS 发现经验，Lean 验证经验，Replay Buffer 保存经验，Learner 吸收经验。**

## 数据生成

### Actor 搜索

Main RL 使用约 **8000 万个形式化问题**作为主要课程，另外加入约 **3500 个**人工形式化问题。

当前 Proof Network 输出：

$$
\pi_\theta(a\mid s),\qquad V_\theta(s)
$$

其中：

- $\pi_\theta(a\mid s)$：Policy，判断当前 Lean state 下哪些 tactic 值得尝试；
- $V_\theta(s)$：Value，估计当前状态距离完成证明的代价。

Matchmaker 负责选题并分配搜索预算，Actor 使用当前 Policy / Value 指导树搜索，直到：

$$
\text{proof},\qquad
\text{disproof},\qquad
\text{budget exhausted}
$$

其中，搜索过程中找到的 **Lean 验证成功的 proof / disproof** 会送给 Learner；单纯失败或耗尽预算的 attempt 会被过滤，**不参与网络更新**。

### 成功轨迹

若搜索找到：

$$
s_0\xrightarrow{a_0}s_1
\xrightarrow{a_1}s_2
\rightarrow\cdots\rightarrow s_T
$$

就得到一组成功经验：

$$
(s_0,a_0),\;
(s_1,a_1),\;
\dots,\;
(s_{T-1},a_{T-1})
$$

这些经验来自模型自己的搜索，但已经由 Lean 验证有效，并最终进入 Replay Buffer。

因此不要把 Replay Buffer 理解为“保存整个 MCTS 搜索树”。更准确地说：

> **MCTS 负责探索大量候选路径，Replay Buffer 保存其中最终被证明有效的经验。**

## 回报目标

AlphaProof 希望证明尽量短，因此每执行一个 tactic：

$$
r_t=-1
$$

从状态 $s_t$ 到证明结束的 return 为：

$$
G_t=\sum_{k=t}^{T-1}r_k
$$

若还需要 $n$ 步完成证明，则：

$$
G_t=-n
$$

例如还需要 5 步：

$$
G_t=-5
$$

因此 Value 本质上学习：

> **从当前 proof state 出发，还需要多大代价才能完成证明。**

当一个状态产生多个必须全部解决的 subgoal 时，AlphaProof 使用：

$$
G(s)=\min_i G(s_i)
$$

由于 return 为负数，这等价于关注**最长、最困难的证明分支**。

## 模型更新

Replay Buffer 本身不更新参数，真正训练网络的是中央 **Learner**。

### 数据混合

Learner 持续从两个数据源构造 batch：

$$
\boxed{
10\%\ \text{Mathlib SFT}
+
90\%\ \text{Replay Buffer}
}
$$

例如 batch size 为 1000，可直观理解为：

$$
100\text{ 条 Mathlib 样本}
+
900\text{ 条 Replay 样本}
$$

这里非常重要：

> **10% / 90% 是 batch 的数据采样比例，不是 SFT loss 与 RL loss 的权重。**

Mathlib 数据提供稳定的人类证明分布，Replay 数据则让模型学习自己通过搜索新发现的证明策略。

### Policy

对于成功经验：

$$
(s_t,a_t)
$$

Policy 学习预测成功 tactic：

$$
\mathcal L_{\text{policy}}
=
-\log\pi_\theta(a_t\mid s_t)
$$

若 tactic 是多个 token：

$$
\mathcal L_{\text{policy}}
=
-\sum_j
\log
\pi_\theta(y_j\mid s_t,y_{<j})
$$

所以优化形式看起来很像普通 SFT。真正不同的是**训练数据来源**：

$$
\text{SFT}:
\quad
\text{Human Proof}
\rightarrow
(s,a)
$$

而 Main RL 是：

$$
\text{Current Model}
\rightarrow
\text{Search}
\rightarrow
\text{Lean Verify}
\rightarrow
(s,a)
$$

因此 AlphaProof 的 RL 性质主要来自：

$$
\boxed{
\text{模型生成经验}
\rightarrow
\text{环境验证}
\rightarrow
\text{学习经验}
\rightarrow
\text{再次生成}
}
$$

而不是 PPO / GRPO 式的 policy-gradient 更新。

### Value

同一条成功轨迹还能提供：

$$
(s_t,G_t)
$$

Value Head 学习：

$$
V_\theta(s_t)\approx G_t
$$

AlphaProof 使用的是 **categorical value**：网络先预测 return 所属各区间的概率，

$$
p_\theta(z_i\mid s)
$$

再由分布得到 value：

$$
V_\theta(s)
=
\sum_i z_i\,p_\theta(z_i\mid s)
$$

因此一次成功轨迹可以同时产生：

$$
\boxed{(s_t,a_t,G_t)}
$$

其中：

- $a_t$ 用于训练 Policy；
- $G_t$ 用于训练 Value。

概念上一次联合更新可写为：

$$
\mathcal L
=
\mathcal L_{\text{policy}}
+
\lambda_V\mathcal L_{\text{value}}
$$

注意 $\lambda_V$ 属于 Policy / Value loss 的组合问题，和 **10% / 90% 数据采样比例**不是一回事。

## RL 闭环

Main RL 可以压缩为八步：

1. 当前网络输出 Policy 与 Value：
   $$(\pi_{\theta_k},V_{\theta_k})$$
2. Matchmaker 从训练课程中选题并分配预算；
3. Actor 使用当前网络指导 MCTS；
4. Lean 验证搜索得到的 proof / disproof；
5. 成功经验进入 Replay Buffer，失败 attempt 被过滤；
6. Learner 按 **90% Replay + 10% Mathlib** 采样；
7. 同时更新 Policy 与 Value：
   $$(\pi_{\theta_k},V_{\theta_k})
   \rightarrow
   (\pi_{\theta_{k+1}},V_{\theta_{k+1}})$$
8. 新网络重新指导下一轮搜索，产生新的经验。

因此整个系统形成：

$$
\boxed{
\text{Search}
\rightarrow
\text{Lean}
\rightarrow
\text{Replay}
\rightarrow
\text{Learn}
\rightarrow
\text{Search}
}
$$

![[ChatGPT-replay-buffer.png]]

## 关键规模

| 项目 | AlphaProof |
|---|---:|
| Mathlib SFT | 约 30 万个 state–tactic pair |
| 自然语言题库 | 约 100 万题 |
| 形式化课程 | 约 8000 万题 |
| 人工形式化补充 | 约 3500 题 |
| SFT 采样 | 10% |
| Replay 采样 | 90% |
| Main RL 算力 | 约 8 万 TPU-days |

其中 **8 万 TPU-days** 是整个 Main RL 的计算规模，并不是 Replay Buffer 的容量。论文也没有把 Replay Buffer 描述成一个固定容量为多少条样本的普通队列，因此不应自行假定具体 buffer size。

## 核心理解

Replay Buffer 真正解决的是：

$$
\text{搜索发现的能力}
\quad\Longrightarrow\quad
\text{可训练的数据}
$$

最终形成：

$$
\boxed{
\text{会搜索}
\rightarrow
\text{找到成功 proof}
\rightarrow
\text{存入 Replay}
\rightarrow
\text{学习 proof}
\rightarrow
\text{更会搜索}
}
$$

因此 Replay Buffer 是 AlphaProof **Main RL 自我增强闭环中的中间记忆层**：没有它，搜索得到的成功经验只能用于当前一次搜索；有了它，这些经验能够持续用于训练 Policy / Value，并进一步改善下一轮 MCTS。