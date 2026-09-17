# 09｜Matchmaker：AlphaProof 的公开设计与 nanoproof 实现

依据：[[用户记录/Matchmaker-1]]。

**核心结论：Matchmaker 是题目层的在线调度器，依据搜索历史选择下一道题并分配搜索预算。官方伪代码与 nanoproof 都采用“逐题规则权重 → 全题库加权随机采样”；没有题目对的比较，也没有学出来的排序模型。** nanoproof 具体实现了预算计算、异常处理和恢复机制，但只做 prove，不能视为 DeepMind 生产代码。

## 1. 它在系统中的位置

```text
题库 → Matchmaker → Actor → proof network 引导树搜索 + Lean
          ↑                       │
          └──── outcome / 历史 ────┘
                                  └→ 成功证明经验 → Replay → Learner
                                                               ↓
                                                        更新 proof network
```

Matchmaker 决定**搜哪道题、以什么目标搜、分配多少搜索**；Actor 内的搜索决定探索哪些 tactic 和证明状态；Lean 检查证明；Learner 更新参数。问题调度与树内的 PUCT 选择属于两个层级。

**Q：simulation budget 是什么？** 一次 Actor 尝试最多允许执行多少轮树搜索 simulation；一轮通常包含选择路径、扩展和回传。它不是 token 数、tactic 数或固定墙钟时间，找到证明时可以提前结束。

**Q：Matchmaker 是 1B/2B 模型、encoder–decoder 或 MLP 吗？** 公开材料没有为它定义神经网络架构或参数量；官方附件是保存统计、执行条件分支和随机采样的普通程序。论文中的 **3B encoder–decoder 是 proof network**，不要移到 Matchmaker 上。

## 2. AlphaProof：正文、附件与伪代码分别公开了什么

正文描述按近期尝试的 interestingness 调度：未知、尝试不足、成功失败混合的题优先；长期未解或稳定证明的题降权；已证伪题停止重试。预算从小值开始，按近期失败次数乘法增加并设上限；TTRL 解出目标后停止给目标及关联 variants 分配新尝试。[论文 Methods：Main RL / Focused RL](https://www.nature.com/articles/s41586-025-09833-y)

**Q：interestingness 只有大致高低，还是有明确算法？** 正文给状态规则，Supplementary Table 7 给数值，官方伪代码给可读的权重与采样逻辑。因此它比“只有概念介绍”具体，但三份材料并非逐行一致的生产规格。[官方 Supplementary Data 1（ZIP）](https://media.springernature.com/original/springer-static/esm/art%3A10.1038%2Fs41586-025-09833-y/MediaObjects/41586_2025_9833_MOESM2_ESM.zip)

### 2.1 官方伪代码中的逐题权重

附件中的文件实际名为 `2025-06-13867A-s2/pseudocode.py`，论文 Code availability 称其为 `alphaproof_pseudocode.py`。以下按下载文件中的 `Matchmaker.Stats.weight()` 解释。

**AlphaProof 的 Matchmaker 初始化的是“配置 + 题库及其历史”，不是一组待训练的模型权重。** 官方训练入口创建 `Matchmaker(config)`；构造函数保存配置，并用注释说明从数据库加载题目与统计。不过实际加载代码被省略，`theorem_stats` 只是一个空字典占位，所以附件没有完整交代新题入库、首次统计建立或续训恢复过程。对没有尝试历史的新题，权重函数返回 1；对已有历史的题，则按历史判断。Main RL 与 TTRL 使用不同的起始题池及部分调度参数，但不是先训练一个 Matchmaker 再迁移其神经网络参数。

每道题记录全部 `(disprove, success)` 尝试；前者表示此次是否要求证伪，后者表示此次目标是否完成。按顺序判断：

| 条件 | 权重 |
|---|---:|
| 从未尝试 | 1 |
| 历史中任何一次成功证伪 | 0 |
| 总尝试数小于 `mm_trust_count` | 1 |
| 尝试已足够，但历史中既没有 proof 也没有 disproof | `mm_undecided_weight` |
| 否则，取最后 `mm_fully_decided_trust_count` 次；它们全是成功 prove | `mm_proved_weight` |
| 其他 | 1 |

`Stats.update()` 追加 `(game.disprove, game.root.is_optimal)`，再按规则重算权重；没有权重网络训练、梯度更新或“加一个分数”的递推。

这里有两个阅读边界：

- “历史上有没有成功”使用**全历史**，并非所有判断都只看近期窗口。即使近期全失败，只要曾证明成功，也不会进入“历史从未解决”的降权分支。
- 伪代码对最近切片执行 `all(...)`，没有再要求切片长度达到 12。例如已有 8 次成功 prove，满足默认探索阈值 8 后，长度为 8 的切片也可直接降权。它与正文“连续达到阈值才视为掌握”的表述存在细节差异；不能把简化伪代码当成无遗漏的生产实现。

### 2.2 是对所有题打分，还是相对排序？

官方 `get_start_position()` 遍历 `theorem_stats` 中所有题，计算每道题的权重，再调用 `random.choices(..., weights, k=1)`。设当前加载题库为 $\mathcal P$，则：

$$
\Pr(P_i\text{ 被选中})=\frac{w_i}{\sum_{P_j\in\mathcal P}w_j}.
$$

因此可以说**所有题都参与权重计算**，但“打分”只是按历史分档赋权，不是读取题目文本预测数学价值。没有 pairwise ranking，没有排序后取 top-1；低权重题仍能抽中，只有权重 0 被排除。权重先独立产生，最终概率通过分母依赖整个题库。

抽中后，以 `mm_disprove_rate` 随机决定 prove/disprove，再确定预算并创建 `Game`。默认 `mm_disprove_rate=0.5`；证伪尝试失败不等于证明原题为真，prove 失败也不等于原题为假。

### 2.3 公开参数与预算边界

| 参数 | AlphaProof Main RL | AlphaProof TTRL | nanoproof 默认 |
|---|---:|---:|---:|
| Disprove rate | 50% | 同 Main RL | 不提供，只 prove |
| `trust_count` | 8 | 5 | 4 |
| `trust_count_proved` | 12 | 同 Main RL | 6 |
| interesting / undecided / fully proved 权重 | 1 / 0.1 / 0.001 | 同 Main RL | 1 / 0.1 / 0.001 |
| disproved 权重 | 0 | 同 Main RL | 无此状态 |
| 起始 simulations | 250 | 125 | 64 |
| 每次失败的倍率 | 1.17 | 2 | 1.5 |
| simulations cap | 16000 | 同 Main RL | 1024 |

AlphaProof 数值来自 [Supplementary Table 7](https://media.springernature.com/original/springer-static/esm/art%3A10.1038%2Fs41586-025-09833-y/MediaObjects/41586_2025_9833_MOESM1_ESM.pdf)；该表 TTRL 列的破折号沿用 Main RL 设置。nanoproof 数值来自 `MatchmakerConfig.defaults()`。

**重要区别：官方伪代码的 `compute_num_simulations()` 只是固定 `return 1000`。** 它没有实现正文描述的自适应预算，也没有暴露正文所说窗口 $N$ 的完整预算配置。不能把伪代码中的 1000 当成实际训练参数，更不能直接认定 AlphaProof 的窗口 $N$ 等于 `trust_count`。

**Q：为什么看最近窗口，不看全部历史？** 模型持续更新，早期失败会逐渐过时；近期表现更适合估计当前能力与计算需求。这是设计动机的解释，不是所有实现分支的事实：官方伪代码和 nanoproof 都保留全历史，并在“是否曾成功”判断中使用它。

## 3. nanoproof 的 Matchmaker：沿实际调用链读代码

nanoproof 的实现思路可以先理解为：**给每道题放一本尝试记录，按记录决定它多久再被抽到，以及抽到后值得花多少时间搜索。** 开始时还不了解题目，先给大家相同的机会；尝试几次仍没证明过的题降低抽中机会，已经稳定证明的题也少做，把更多尝试留给尚未充分探索、或已经有成功但还不稳定的题。选题不是按固定顺序轮流做，而是让高权重题更容易抽中，同时保留低权重题再次被尝试的机会。

选中题目后，它再做第二个决定：最近失败越多，这次允许搜索的轮数越多。这样可以同时做到“困难题少抽一些，但抽中时搜得更充分”，而不是每次都用很小的预算重复同一种失败。Actor 返回结果后，只需把这次结果记到账上；下一次抽题时按同样规则重新判断，整个课程就会跟随实际表现变化。异常另行处理，一步就能证明的题也会提前降权，下面再展开这些细节。

这套设计不需要额外训练一个选题模型，也不直接阅读题意估计数学价值。**变化的是记录、抽中概率与搜索预算；规则和阈值通常由启动配置确定。** 其直接效果是重新分配计算资源；是否提高最终证明率，还需与均匀抽题做同算力实验，不能把设计动机当成已证实的性能增益。

下面把这个过程对应到实际代码。主要源码为 [experience_collection.py](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/experience_collection.py)，Actor 调用见 [prover.py](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/prover.py)。规则以**函数体**为准；个别注释比实际实现更笼统。

### 3.1 从初始化到运行：准备题库、记录与固定规则

一次全新运行先从默认值或命令行取得调度配置：尝试多少次才作判断、各类题的抽样权重、起始搜索预算、失败倍率和预算上限。这些设置保存在 `MatchmakerConfig` 中，本次实现不会根据梯度自动优化它们。随后创建 `Matchmaker(datasets, lean_version, config, seed)`，加载指定数据集的训练题，按 Lean 版本过滤不能正常建立初始状态的题目，并为每道可用题建立一份空记录。

每份记录由 `TheoremStats` 保存，内容是历次“结果 + 成功证明的步数”。结果分为已证明、此次未证明和运行异常；“此次未证明”不表示命题为假。全新运行的记录都是空的，所以默认各题权重为 1，起始预算都是 64。题目直接合并成一个池，**不是先等概率抽数据集再抽题**；因此题目更多的数据集最初占据更多抽样概率。实现还建立题目索引以定位记录，用随机种子控制抽样，并用线程锁协调并发读写。

续训时也先建立上述对象，再通过 `reconstruct_from_run_dir()` 读取已有运行日志，恢复每道题的尝试历史。因此，续训不需要把所有题重新当成未知题，也不是加载某个“训练好的 Matchmaker 权重文件”。具体恢复边界见 3.5。[初始化调用：rl.py](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/rl.py)

**固定调度程序不等于固定调度结果。** 运行中每次收到搜索结果，题目历史都会追加，权重与预算随后按固定规则重算；proof network 的参数更新也会通过改变后续搜索表现，间接改变调度。两者没有直接的“Learner 更新 Matchmaker 参数”连接。

起始题库也会影响结果：换一批题，会改变参与抽样的题目及总权重，即使某道保留题的自身权重不变，它的抽中概率也可能改变。nanoproof 当前代码在构造时加载题库，没有实现运行中自动增删题目的接口；恢复时还检查数据集配置一致。AlphaProof 正文则明确区分 Main RL 通用题池与 TTRL 目标相关题池，但生产系统如何在运行中扩充题库、同步旧统计，官方伪代码没有展开。因而应区分**初始化时选择或加载题库**与**运行时持续更新历史**，不能笼统地说 Matchmaker“完全不更新”或“会随数据重新训练”。

### 3.2 `TheoremStats.weight()`：真实判定顺序

| 顺序 | 实际条件 | 默认返回 | 目的 / 后果 |
|---:|---|---:|---|
| 1 | 原始历史最后两次均为 `error` | 0 | 排除反复异常的题 |
| 2 | 历史任何一次 `proven`，且 `proof_size == 1` | 0.001 | 一步证明题立即降权 |
| 3 | 过滤 `error` 后无有效尝试，或有效尝试少于 4 | 1 | 先探索，积累证据 |
| 4 | 有效尝试至少 4 次，但**全历史从未 `proven`** | 0.1 | 当前未解题降权 |
| 5 | 最近 6 次有效尝试存在且全是 `proven` | 0.001 | 稳定成功题降权 |
| 6 | 其他 | 1 | 继续重点尝试 |

三个容易忽略的设计：

1. **异常和搜索失败分开。** `error` 不进入有效尝试窗口，不加失败预算；两次连续原始异常将权重归零。正常调度不会再抽到它；若已有在途任务后来返回非异常结果，末尾条件可能改变，所以代码并非不可逆的删除标记。
2. **一次短证明可以跳过掌握阈值。** 即使后续失败，只要全历史中出现过一步 proof，仍返回低权重；但连续异常的归零分支优先级更高。
3. **“近期失败多”不必降低采样权重。** 例如曾找到多步 proof，后来连续失败 10 次，仍为权重 1；因为未解分支检查“历史从未证明”，而不是“近期没有证明”。代码注释中“trust window 内无 proof”的说法不能代替函数体。

与官方伪代码相比，nanoproof 明确检查最近成功切片长度至少为 `trust_count_proved`，避免不足 6 次就因 `all()` 被当成稳定掌握。

### 3.3 `num_simulations()`：公式与实际可达预算

过滤 `error`，取最近 `trust_count` 次有效结果，令其中 `unproven` 的数量为 $F_i$：

$$
B_i=\min\left(B_{\max},\left\lfloor B_0 m^{F_i}\right\rfloor\right).
$$

对默认配置，$B_0=64,m=1.5,B_{\max}=1024$，且 $0\le F_i\le4$：

| 最近 4 次有效结果中的失败数 | 0 | 1 | 2 | 3 | 4 |
|---|---:|---:|---:|---:|---:|
| 分配 simulations | 64 | 96 | 144 | 216 | 324 |

**所以默认最高可达预算是 324，1024 只是配置上限，当前窗口下不会触发。** 持续失败 100 次也不会突破 324。默认基数与倍率不变时，窗口至少允许 7 个失败，才可能触及 1024 的 cap。

这不是“每次失败就永久把上次预算乘 1.5”，而是**每次从近期失败数重新算**。新成功进入窗口、旧失败退出后，预算会下降；未完成的在途尝试也不计入此次计算。

### 3.4 `next_assignment()` 与 `send_result()`

```text
next_assignment()：在锁内
    对全部已加载题目调用 stats.weight(config)
    确认总权重大于 0
    random.choices(所有题目索引, weights=weights, k=1)
    仅对抽中的题计算 num_simulations(config)
    返回 (theorem, num_simulations)

Actor：
    尝试证明 → 搜索、Lean 检查、生成经验
    _report_collect() 判定 outcome，计算 proof_size
    holder.record_attempt(...) 保存尝试与可用训练经验
    matchmaker.send_result(...) 在锁内追加该题历史
```

这里没有 objective 字段或随机 disprove 分支。`prover.py` 将显式异常归为 `error`；根节点已解决归为 `proven`；其余归为 `unproven`。失败尝试可以改变后续调度，但不会凭空产生一条成功 proof trajectory。

线程锁保证抽题读取与历史写入的一致性，**不保证每题只有一个 Actor 在搜**：它没有预订题目或扣除在途权重，多名 Actor 可以同时抽中同一题。历史按结果到达顺序追加，未必是启动顺序。

### 3.5 恢复与工程成本

`reconstruct_from_run_dir()` 检查当前与先前的 datasets 配置一致，按 step 顺序读取 `step_*/theorems.jsonl`，重放 outcome 与 proof_size；当前 whitelist 不再包含的题会跳过。这能恢复已有日志对应的统计，避免重启把所有题重新当成未知。

不过恢复**统计状态**不等于恢复完全相同的未来采样序列：该函数没有恢复 RNG 状态、在途任务或完整线程时序。

每次抽题都遍历全题库；每次 `weight()` 又扫描该题完整历史。粗略复杂度为 $O(M+\sum_i H_i)$，其中 $M$ 是题数、$H_i$ 是历史长度；扫描发生在锁内。它适合小规模可读实现，不能据此推断 AlphaProof 对约 8000 万题也逐次使用同样的全扫描工程方式。大规模实现可缓存权重、增量维护状态并使用分档或加权采样数据结构，但这些是改进建议，不是已公开的生产细节。

## 4. 设计效果：用同一组小例子看清楚

以下 $S$ 表示成功的**多步** proof，$U$ 表示 unproven，$E$ 表示 error；使用 nanoproof 默认配置。

| 题目状态 | 权重 | 单次预算 | 解释 |
|---|---:|---:|---|
| A：未尝试 | 1 | 64 | 便宜探索 |
| B：`S,U,S,U` | 1 | 144 | 已有成功但未稳定，继续重点投入 |
| C：连续 6 次 `S` | 0.001 | 64 | 已稳定，极少重试 |
| D：`U,U,U,U` | 0.1 | 324 | 少抽，但抽中时多搜 |
| E：一次一步 proof | 0.001 | 64 | 特殊短证明规则直接降权 |
| F：末尾 `E,E` | 0 | 不再新分配 | 异常排除 |

只看 A–D，总权重为 2.101，对应采样概率约 **47.60%、47.60%、0.048%、4.76%**。D 的权重是 A 的十分之一，但每次预算约是 A 的 5.06 倍；只比较 A 与 D，按“概率 × 分配预算”估算，D 的预算份额约是 A 的 0.506 倍。实际耗时还受提前成功、节点扩展成本与服务延迟影响。

这说明 **interestingness 控制重试频率，budget 控制抽中后的搜索强度**。困难题可以“少做、每次多搜”；两者不是同一个分数。

## 5. AlphaProof 与 nanoproof 到底有什么区别

| 方面 | AlphaProof 公开部分 | 本次核对的 nanoproof |
|---|---|---|
| 题目选择 | 官方伪代码逐题规则赋权、全题库加权随机采样 | 同类机制，明确平铺所有已加载题目 |
| 调度目标 | prove / disprove 随机分配 | prove-only |
| 确定性退出 | 成功 disproof 权重 0；正文说明 TTRL 目标完成后停止其课程 | 无 disproof 或目标—variant 联动退出；有连续异常归零 |
| 历史使用 | 正文强调近期；伪代码部分判断查全历史 | 存全历史；曾成功与一步 proof 查全历史，掌握与预算查窗口 |
| 稳定掌握 | 正文是连续成功阈值；伪代码切片缺少长度检查 | 显式检查至少 6 次有效连续成功 |
| 短证明特判 | 官方 Matchmaker 伪代码未见一步 proof 降权 | 一步 proof 立即降到 0.001 |
| 自适应预算 | 正文与表 7 给原则和参数；伪代码固定返回 1000 | 真正实现近期失败计数、乘法增长、整数截断与 cap |
| 持久化 / 并发 | 伪代码仅概述从数据库加载统计 | JSONL 重建历史、线程锁、Actor 调用链 |

因此区别不是“一个打分、另一个排序”，而是**同一种历史启发式采样框架，目标范围、状态规则、预算实现与工程完整度不同**。nanoproof 也没有复现这里所述的完整 TTRL 目标及 variants 调度。

## 6. 哪些效果有证据，哪些只是机制推断

从代码与上例可以直接验证：未解题降权、稳定题低频重试、近期失败增加单次预算、异常不增加预算。这些是算法行为。把它解释为减少容易题重复、为尚未稳定的题持续生成成功经验，是合理的设计动机；**权重 1 也包含“曾成功、现在长期失败”的题，因此不能直接等同于学习收益最优的题。**

项目固定版本 README 报告整体系统最好结果为 **MiniF2F-Valid 52.7%，512 simulations**。[nanoproof README](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/README.md) 这是整个证明系统的结果，而且是验证集；512 是评测搜索预算，不是上面 Matchmaker 的默认训练预算。它不能量化 Matchmaker 单独带来的增益，也不能与不同题目版本、计算预算的 AlphaProof 数字直接比较。

核对的公开材料没有提供 Matchmaker 相对 uniform sampling 的独立性能消融。要证明调度带来净收益，应固定模型、题库、搜索与训练设置，按**相同总计算成本**比较 uniform 与 matchmaker 的有效经验产量和独立留出集 solve rate；只比较相同尝试次数，会因 adaptive budget 导致算力不等而失真。

组会可以这样表述：**AlphaProof 公开了可解释的课程调度启发式，nanoproof 把它变成了可运行、可追踪的工程组件。两者都靠历史分档赋权后随机抽题，而不是用一个额外大模型给全部题目排序；公开证据支持机制说明，尚不足以归因一个独立的性能提升百分比。**
