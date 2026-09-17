# 省流
- 本文这些项目都只复现或覆盖了alphaproof的部分环节。
	- REAL-Prover是best-first，不是MCTS；可以复用逐步策略和检索。
	- REAP有价值驱动搜索代码；搜索可以参考，value训练需要另行实现或核实。
	- nanoproof有RL和matchmaker，没有TTRL；progressive sampling会清空旧子节点，有问题
	- StepProver有逐步策略、critic和专家迭代；critic学状态偏好，alphaproof的value学剩余证明深度。
	- Kimina是整篇证明生成，有测试时引理搜索和RL；不是alphaproof的逐步策略、状态value和MCTS闭环。
	- DeepSeek-Prover-V2也是整篇证明生成，没有alphaproof式状态value、逐步MCTS和目标TTRL。
		- 这两个适合做效果好的小prover。Kimina的训练重点是长推理、蒸馏、GRPO和错误修复；V2是教师分解、子目标验证、蒸馏和RL。
- 最直接的复用资源是公开数据和权重，此外还有检索、搜索和部分训练代码；数据见[[03_alphaproof_public_models#数据公开情况|数据公开情况]]。
- 关于基座和训练的复用，结论是：
	- 做小prover，基座能力最好的是Kimina 1.7B / 8B和V2-7B；复现alphaproof，逐步策略是StepProver / REAL 但其实能力都不咋样
	- 本文列出的整篇生成模型中，miniF2F-test、pass@32：Kimina 72B报84.0%，V2-671B报82.4%；小模型Kimina 8B报77.86%、1.7B报76.63%，V2-7B报75.6%。
		- 原版alphaproof在TTRL前，平均每题2 TPU分钟搜索为96.3%，12 TPU小时为97.7%。
	- 训练方式：alphaproof式搜索—训练闭环看HTPS / nano；小prover看Kimina公开的GRPO、错误修复代码和V2的教师蒸馏方法。
	- 具体测试集和预算见[[03_alphaproof_public_models#汇总比较：定位与测试|测试与跑分]]；matchmaker实现见[[03_alphaproof_public_models#match maker 是否还没有开源复现？|matchmaker专节]]。
# AlphaProof 相关公开模型、复现机制与复用判断

**先给出复用判断。** 若目标是 AlphaProof 机制复现，最相关的是逐步策略、价值目标、AND–OR 搜索、成功回放与任务调度，现成模型通常只能替代其中部分。若目标是效果好的独立小 prover，V2-7B 与 Kimina 小模型更适合作为整篇生成基线，REAL 与 ReProver 则提供库知识检索路线。后文保留各系统的机制细节，并集中列出成绩、数据下载入口和全量公开的证据边界。

本篇回答两个相互关联的问题：**已有的公开证明模型，可以替我们省去哪些工作？不同系统与 AlphaProof 相似在哪里，又不能等同在哪里？** 讨论对象包括逐步证明模型、整篇证明模型、搜索与训练框架，以及使用大模型规划证明的代理系统；不评估我们项目的当前进度。nanoproof 已有独立源码讲义，本篇只在 matchmaker 问题中补充直接相关的调度代码。

阅读顺序是：先统一评测口径，再了解逐步模型与 Reap，随后重点讨论 DeepSeek-Prover-V2、Kimina，最后比较 LEAP、Nexus 和早期工作。下文“论文报告”指作者给出的实验结果，“源码可见”指本次读取代码能够确认的行为；推导、举例和选型建议另作说明。**本文未重新训练这些模型，也未重新运行整套基准；代码存在与结果已独立复现不是同一层证据。** 公开范围核查截至 2026 年 9 月 17 日，成绩保留各原始资料的版本与预算，而不是混成一个当前排行榜。

## 对象与评测

**首先分清四层对象。** 模型权重决定给定输入时生成什么；搜索算法决定下一次在哪里尝试；Lean 交互框架负责执行 tactic、返回状态和检查证明；训练控制系统负责收集数据、计算损失、更新参数和发布检查点。tactic 是 Lean 的证明指令，可以是简单改写，也可以是内部进行复杂自动推理的一条命令。因此，“一条 tactic”不等于“一次简单数学推理”。

对应到本文：StepProver、REAL-Prover 有可下载的逐步证明模型；DeepSeek-Prover-V2、Kimina 主要提供带或不带自然语言推理的完整证明生成能力；Reap 是连接后端服务的 Lean 搜索框架；LEAP、Nexus 是组织推理、草图和工具的代理系统。不能把这些名字当作同一类基座直接排名。[StepProver 论文](https://arxiv.org/html/2410.15700v2) · [REAL-Prover 论文](https://arxiv.org/html/2505.20613v1) · [DeepSeek 官方说明](https://github.com/deepseek-ai/DeepSeek-Prover-V2) · [Reap](https://github.com/frenzymath/reap) · [LEAP 论文](https://arxiv.org/html/2606.03303v2) · [Nexus 论文](https://arxiv.org/html/2605.22763v2)

B 表示十亿参数，M 表示百万参数。SFT 是 supervised fine-tuning，即监督微调；RL 是 reinforcement learning，即强化学习；CoT 是 chain of thought，即生成出来的推理过程；MCTS 是 Monte Carlo tree search，即蒙特卡洛树搜索。TTT 指测试时训练，TTRL 指测试时强化学习。rollout 指模型与环境交互所得的一次尝试过程。**有多次尝试不等于有训练，有训练也不等于有针对测试目标的训练。**

**比较能力之前，必须知道测试集考什么。** miniF2F 常用划分包含 244 道验证题和 244 道测试题，来源包括 AMC、AIME、IMO 和 MATH 数据集，主体是初等数学。它既有相当直接的题，也有竞赛题；不能把整个集合称作“244 道 IMO 难题”。不同工作修订过错误形式化，同名数据集也未必逐题一致。[DeepSeek-Prover-V2，§3.1](https://arxiv.org/html/2504.21801v1#S3.SS1)

ProofNet 是大学纯数学教材题，涉及实分析、复分析、线性代数、抽象代数和拓扑。近期若干 Lean 4 实验用 185 道验证题、186 道测试题，但较早论文可能评估另一个可运行子集，或者合并验证集与测试集。**总体分数不能直接回答“代数强还是分析强”**：这需要按学科划分的逐题结果，而不是看到题库包含分析，就推断模型擅长分析。[ProofNet 原论文](https://arxiv.org/abs/2302.12433) · [DeepSeek-Prover-V2，§3.2](https://arxiv.org/html/2504.21801v1#S3.SS2)

PutnamBench 来自大学生 Putnam 数学竞赛，不是一般大学教材习题。题目往往要求非例行的构造、组合或分析技巧；数据集持续扩充，本文引用的 640、658 等分母属于不同历史版本。它也不能与“某系统做出了 2025 年 Putnam 的全部 12 题”互换：后者是一年的题集，前者是跨年份题集。[PutnamBench 论文](https://arxiv.org/abs/2407.11214) · [DeepSeek-Prover-V2，§3.2](https://arxiv.org/html/2504.21801v1#S3.SS2)

其余专门评测在相应章节介绍：FATE-M 侧重大学抽象代数；ProverBench 混合教材题与近期 AIME 题；CombiBench 侧重组合竞赛题；Lean-IMO-Bench 刻意挑选需要非例行思路的竞赛问题。不同分布测量的是不同能力，不应只比较成功率的大小。

**预算也必须同口径。** 对整篇生成模型，pass@32 通常表示最多生成 32 份完整候选证明时，至少找到一份正确证明的覆盖率或相应估计。对逐步搜索器，一次 pass 自身可能包含几百次状态扩展，每次扩展又生成几十条 tactic。因此，搜索器的 pass@32、整篇生成的 pass@32、MCTS 的 32 次 simulation 不是同一预算。simulation 是一次树内选择、扩展和回传过程，通常不会等价于一份完整证明。[StepProver，§2--3](https://arxiv.org/html/2410.15700v2) · [AlphaProof，Methods](https://www.nature.com/articles/s41586-025-09833-y.pdf)

公开程度也要逐层问：是否有权重、训练数据、搜索代码、训练代码、评测脚本、逐题证明和环境锁定信息？公开“训练题目”不等于公开“全部训练轨迹”；公开“解答文件”不等于公开“生成这些解答的代理程序”。

证据优先采用原论文、官方 GitHub 与相应模型或数据卡。Kimina 的 7 月 TTRL 细节主要来自作者技术说明，本文明确标作“作者报告”；4 月论文和 8 月 GitHub 训练配方只能支持各自阶段，不能替它补出未公开的实现。公开边界结论限于所列来源，不主张已经穷尽所有第三方复现项目。

## 原版参照

这里不重讲 AlphaProof 全流程，只建立后文比较所需的参照。原版 proof network 是约 3B 的编码器—解码器网络，同时承担策略和价值预测：策略根据证明状态生成 tactic；价值估计从该状态完成证明的回报。先做数学库证明数据的 SFT，再进入搜索产生数据、数据更新网络的主 RL，困难测试题可以进一步进入目标相关的 TTRL。[AlphaProof 原论文，Prover agent、Training](https://www.nature.com/articles/s41586-025-09833-y.pdf)

**原版 value 不是简单的“成功概率”。** 每走一条实际证明 tactic 有步数代价；多个子目标构成 AND 关系，需要全部完成，回报沿最困难的分支计算。对一棵已经找到的证明树，可用 $G(s)=-D(s)$ 理解其监督目标，其中 $D(s)$ 是从状态 $s$ 出发的**剩余最长证明分支长度**。这里是该成功证明结构给出的目标，不表示已经找到了所有可能证明中的全局最短证明。[AlphaProof 原论文，Tree search algorithm](https://www.nature.com/articles/s41586-025-09833-y.pdf)

主 RL 中，actor 是负责搜索的执行单元，learner 是负责更新参数的训练单元，matchmaker 是选择问题并分配预算的调度器。经验证的成功证明或成功证否提供训练样本；未成功的尝试进入调度统计，但不作为该机制的网络更新样本。learner 的批次混合 Mathlib SFT 数据与经验回放数据，策略使用交叉熵学习成功 tactic，价值学习证明回报。因此，**不能把“损失是交叉熵”当作“不是 AlphaProof 式 RL”的充分理由**；应看它是否处于搜索改进策略、策略再改进搜索的闭环中。[AlphaProof 原论文，Main RL](https://www.nature.com/articles/s41586-025-09833-y.pdf)

TTRL 则把起始题库换成当前难题及其相关变体，让从通用模型初始化的 specialist——针对这些目标继续适应的证明模型——在搜索与更新之间反复迭代。变体可以是简化、推广、类比或引理，但**不要求每一道变体都是原题证明必需的逻辑前提**。它们首先服务于参数学习，这一点与后文的证明依赖图很不同。[AlphaProof 原论文，Scaling with TTRL](https://www.nature.com/articles/s41586-025-09833-y.pdf)

## StepProver

### 定位与评价器架构

**它是什么。** 本文的 StepProver 特指 InternLM2.5-StepProver，而不是“逐步证明器”的统称。其公开组件包括 7B prover 和独立的 1.8B critic：前者生成下一条 tactic，后者给证明状态评分，帮助决定下一步展开哪个状态。模型面向 Lean 4，主要研究 critic 引导搜索与专家迭代，而不是整篇自然语言证明生成。[论文](https://arxiv.org/html/2410.15700v2) · [7B 模型](https://huggingface.co/internlm/internlm2_5-step-prover) · [1.8B critic](https://huggingface.co/internlm/internlm2_5-step-prover-critic)

**critic 就是 value 网络吗？** 从作用看，可以把这里的 critic 理解为一种状态价值评估器：输入状态 $s$，输出标量 $c_\phi(s)$，其中 $\phi$ 是它的参数。它不负责提出证明步骤，而负责评估搜索位置。但是，“critic”是功能名称，“value head”是网络结构名称，两者不要求采用同样的实现；一个 critic 可以是独立模型，也可以是共享基座上的一个头。[StepProver，§2.1](https://arxiv.org/html/2410.15700v2#S2.SS1)

更关键的是**分数具体学什么**。StepProver 把 critic 当作**偏好排序**模型训练：一类样本取自同一条成功路径，让更靠近完成状态的节点排在祖先节点前面；另一类比较兄弟分支，让通向已找到证明的状态排在未找到证明的兄弟状态前面。因此，它学习的是“这两个搜索位置，哪个更值得继续”，不是直接回归“还需要五步”，也不是已校准的“成功概率为 80%”。未成功的兄弟分支只是这次搜索没有闭合，不等于逻辑上不可证明。[StepProver，Path Pairs、Sibling Pairs](https://arxiv.org/html/2410.15700v2#S2.SS1)

一个用于理解的例子：状态 $s_1$ 得分 2，$s_2$ 得分 5，排序器会优先探索 $s_2$。把所有分数同时加上 100，排序不变；但若把这些分数直接交给一个按负剩余步数设计的 MCTS，数值含义就完全变了。**能正确排序，不代表能未经校准地替代 AlphaProof 的 value。** 这是由两种目标的数学含义推出的接口要求，不是说 StepProver 的 critic 无效。

### 训练方式与 AlphaProof 的异同

**当初怎么训练。** prover 以 InternLM-math-plus-7B 为基础；critic 从 1.8B 模型初始化。系统使用 Mathlib、Lean-Workbook、Lean-Github 等来源启动，再通过专家迭代补充数据。所谓 expert iteration，是“当前模型配合搜索找到经验证的证明 → 把成功轨迹加入训练集 → 训练改进模型 → 再搜索”。它还考虑候选命题的否定，以应对自动形式化产生的错误命题，并在后续轮次提高搜索预算、重新评估较有希望的未解题。[StepProver，§2.2、附录 A](https://arxiv.org/html/2410.15700v2)

这与 AlphaProof 确有结构上的接近：都有逐步动作、搜索改进、成功经验和评估器学习。区别是 StepProver 使用自身的优先搜索与偏好 critic，不能因它有“双模型加专家迭代”就说它实现了原版 AND–OR MCTS、负分支深度 value 或目标变体 TTRL。所核查材料描述的是训练阶段的迭代，没有给出与原版对应的完整目标专项 TTRL 闭环。

### 推理接口与搜索原则

**输入接口有什么特点。** 官方提示包含 `NAME`、`PROOF_BEFORE`、`STATE_BEFORE`、`TACTIC`，也就是定理名字、已写出的证明、当前状态和待生成步骤。它不只是“裸 goal → tactic”。同一 Lean 状态配上不同历史，可能得到不同动作分布；因此接入状态去重、缓存或树搜索时，需要决定历史是否属于模型输入和缓存键。[模型卡](https://huggingface.co/internlm/internlm2_5-step-prover)

**搜索问题的明确回答。** StepProver 没有实现原版 AlphaProof 的 AND–OR MCTS。critic-guided search 优先扩展 critic 评分高的状态；best-first search 则使用路径 tactic 的平均对数似然。每次扩展按设定宽度采样候选，执行后保留有效后继；论文的代表设置为宽度 32、最多 600 次扩展。固定采样宽度不代表每个节点必然产生 32 个不同、有效的子节点，也不代表 MCTS 的访问计数、探索项与回传机制。[论文 §3.1](https://arxiv.org/html/2410.15700v2#S3.SS1)

### 测试表现与领域边界

**能力证据和领域。** miniF2F-test 的代表成绩是 65.9%，对应混合 best-first search 与 critic-guided search 的配置；critic-guided 单独为 65.6%，不能把二者写成同一个实验。该最高配置的搜索预算为 $256\times32\times600$，分别涉及搜索尝试、展开宽度和每次最大状态扩展数。这里的 65.9% 不是整篇生成 pass@256。[StepProver，§3.1](https://arxiv.org/html/2410.15700v2#S3.SS1)

论文还报告 ProofNet 27.0%，但它是**验证集与测试集合并后的整体成绩**，不是后文 DeepSeek、REAL 所报的 ProofNet-test 成绩。作者分析显示 critic 能帮助找到较深证明，同时低预算下也可能漏掉普通搜索更容易找到的解。因此，合理判断是：它具有竞赛题和部分大学教材题能力，critic 提供与策略概率不同的搜索信号；没有足够的分学科证据把它认定为“分析专长模型”或“抽象代数专长模型”。[StepProver，ProofNet 与 Critic-Guided Search](https://arxiv.org/html/2410.15700v2#S3.SS1)

### 公开材料与复用判断

**能下载什么。** prover、critic 权重与调用示例公开；官方仓库的 2024-10-22 发布记录还明确链接了约 1.4 万份 Lean-Workbook 搜索证明，不能只笼统称为“数据来源公开”。[发布记录](https://github.com/InternLM/InternLM-Math#news) [Lean-Workbook](https://huggingface.co/datasets/internlm/Lean-Workbook)、[Lean-Github](https://huggingface.co/datasets/internlm/Lean-Github) 是公开数据入口。但数据来源公开，不自动说明每轮处理后的全部轨迹、全部 critic 偏好对和训练回执均能一一下载。本次没有核实到覆盖所有阶段的完整发布清单，不把它标成“全部训练材料完整公开”。对需要现成逐步策略与独立评价器的研究，它是可研究的起点；对要严格复刻 AlphaProof 数值语义的研究，需要另外处理价值目标与搜索接口。

## REAL-Prover

### 定位与检索接口

**它是什么，偏向什么任务。** REAL-Prover 是 Retrieval Augmented Lean Prover，即检索增强的 Lean 证明器，以 Qwen2.5-Math-7B 为基础。它不是仅靠模型记忆引理，而是在当前证明状态下，通过 LeanSearch-PS 检索 Mathlib 中可能有用的已有定理，再将这些前提交给模型生成 tactic。[论文](https://arxiv.org/html/2505.20613v1) · [代码](https://github.com/frenzymath/REAL-Prover) · [模型](https://huggingface.co/FrenzyMath/REAL-Prover)

**检索问题的明确回答。** LeanSearch-PS 是另一个检索模块，论文以 E5-mistral-7b-instruct 为基础训练稠密检索器，采用 LoRA；它不是 REAL-Prover 的同一个 7B 生成模型。原论文流程由程序在每一步取当前状态、检索 top-k 前提、拼接到 prover 输入，随后生成 tactic；并非让 prover 自主决定是否调用一个检索工具。[论文 §2.2.1、§3.1](https://arxiv.org/html/2505.20613v1#S3.SS1)

premise selection 指“选择证明可能需要的前提或引理”。这里的前提通常是库中已有定理，不是把待证明的新猜想直接当成真。LeanSearch-PS 把状态与库定理编码成向量，通过相似性找候选；模型再决定如何使用。Jixia-interactive 则负责执行 tactic 和返回状态。**检索模型不是价值模型**：检索回答“找哪条引理”，价值回答“这个搜索状态值得继续吗”。[REAL-Prover，§2.2](https://arxiv.org/html/2505.20613v1#S2.SS2)

一个说明其用途的例子：证明群同态的核是正规子群，数学上可能很熟悉，但 Lean 里仍需要找到正确的对象、类型假设与库定理。检索能减少“数学思路已有，却不知道 Lean 名字或接口”的困难。反过来，若难题的瓶颈是构造全新的不变量，仅检索现成引理并不能自动创造这个思路。这是任务结构上的分析，不是该例的模型实测。

### 训练数据与专家迭代

**当初怎么训练。** 论文使用数学库轨迹、人工标注的大学代数证明、Lean-Workbook，以及专家迭代生成的证明。自动形式化部分通过 HERALD-AF 从自然语言教材题得到 Lean 命题，并用回译和语言模型检查降低语义偏差；随后真正搜索、验证证明，再抽取 state–tactic 对继续 SFT。论文总训练集合计 210,420 对，训练上下文为 8192 tokens。[REAL-Prover，§2.1、§3.1](https://arxiv.org/html/2505.20613v1)

### 搜索方式与 AlphaProof 的异同

这里包含自举，但作者明确将该版本称作仅使用监督目标训练的系统。它的原始搜索是 best-first search：按路径上 tactic 对数概率的长度归一化分数，优先展开候选状态，不是原版 AlphaProof MCTS，也没有在该原论文中训练一个 AlphaProof 式 value。不要把 2025 年 REAL-Prover 的架构与后面持续更新的 Reap 架构混在一起。[REAL-Prover，§2.2、Limitations](https://arxiv.org/html/2505.20613v1)

### 测试表现与代数任务证据

**测试表现和难度。** 论文报告 miniF2F-test 54.1%、ProofNet-test 23.7%、FATE-M 56.7%，采用作者记作 $64\times64$ 的逐步搜索预算：64 次搜索尝试、每步生成 64 个候选。这个记法没有把整棵搜索中的总 token 数直接编码进去，不能据此与整篇生成 4096 次等价。[REAL-Prover，§4](https://arxiv.org/html/2505.20613v1#S4)

FATE-M 全称 Formal Algebra Theorem Evaluation--Medium，包含 141 道大学抽象代数题，来自 12 本教材，覆盖群、环、域等结构。论文引言用过 graduate-level，但专门介绍数据集的 §3.2 又明确称 undergraduate-level、simple to moderate；据此更稳妥地理解为**以大学抽象代数为主、简单到中等难度的专门测试**，不能凭论文宣传语升级成研究级代数难题集。例如“两个子群的并仍是子群时，其中一个包含另一个”，属于该集合的典型问题。[FATE-M 定义与示例](https://arxiv.org/html/2505.20613v1#S3.SS2)

**“偏代数”在这里有依据，但边界仍然重要。** 训练资料确实加入了大学代数，FATE-M 也直接测代数。检索相关消融中，不带检索训练和推理的配置在 FATE-M 为 44.7%，检索增强配置为 56.7%；ProofNet 则为 22.6% 与 23.7%。这是两套匹配训练方式的系统配置比较，不是固定同一模型、只打开一个开关的纯推理消融。它支持“检索有助于该类代数任务”，不支持“所有大学数学均领先”或“分析能力已经被充分验证”。[REAL-Prover，§4.4](https://arxiv.org/html/2505.20613v1#S4.SS4)

### 公开材料与复用判断

**公开范围。** 官方仓库提供模型入口、搜索与交互相关代码、FATE-M 数据，并链接约 5 万条公开 [state–tactic 数据](https://huggingface.co/datasets/FrenzyMath/state_tactic_pairs)。这个公开口径与论文的 210,420 对不相同，不能宣称全部最终训练集已经按同样处理方式公开。[官方 README](https://github.com/frenzymath/REAL-Prover/blob/main/README.md)

对研究比较而言，REAL-Prover 的价值是现成的逐步 Lean 能力与检索配套，尤其值得在库依赖较强的代数任务上测试。它原论文中的主要不足是缺少训练得到的搜索价值模块和显式长推理流程；但这些“不足”是相对于特定研究目标而言，不等于已证明给它加上任何 value 或 CoT 就会提高成绩。

## Reap：搜索框架与价值接口

### 搜索框架与价值接口

**先回答最容易混淆的问题：不能根据公开 Reap 仓库，把差距简单归结成“没有价值头”或者“价值头不好”。** Reap 的 Lean 搜索代码已经调用并使用价值服务；但后端究竟是哪种神经网络、用什么数据训练、价值是否校准，需要检查对应的服务实现和权重。Lean 客户端本身无法替这些问题作证。

本节核查的是官方仓库固定提交 [`c6980f3418bc33eed291577aaf7830ab5de561e8`](https://github.com/frenzymath/reap/tree/c6980f3418bc33eed291577aaf7830ab5de561e8)，不是我们项目的某个 fork，也不是把旧实验推测成当前实现。

**源码已经具备什么。** `Generator.lean` 的 `TacticGenerator` 包含策略、价值、前提检索三个客户端。`generateValueFromPrompt` 会请求价值服务，读取返回的 `score`，再把它取负交给搜索。`generatePolicyValue` 将策略和价值请求结合起来；策略还提供动作对数概率。因此，“Reap 完全没有 value 通路”不符合这份代码。[Generator.lean](https://github.com/frenzymath/reap/blob/c6980f3418bc33eed291577aaf7830ab5de561e8/Reap/Tactic/Generator.lean)

### 价值接口与数值语义

`TreeSearch.lean` 也不只是写了 MCTS 名字：它包含 AND/OR 节点、访问计数、价值累计、策略先验、探索分数、渐进式采样以及 AND 子目标的回传处理。多个目标带有需要谨慎处理的元变量依赖时，代码不会一律拆成独立 AND 子目标。渐进式扩展还会保留既有子节点并合并重复候选。[TreeSearch.lean](https://github.com/frenzymath/reap/blob/c6980f3418bc33eed291577aaf7830ab5de561e8/Reap/Tactic/TreeSearch.lean)

**那“价值头”在哪里？** endpoint 是服务地址，不是模型结构。服务背后可能是共享基座上的一个价值头，也可能是独立网络；即使后端输出 64 个价值类别的概率，也完全可以先算成标量再通过接口返回。因此，“接口只有一个 float”不能证明“没有 categorical value head”；反过来，接口名字叫 value，也不能证明后端已经实现并训练了它。[Generator.lean 的 `ValueResult` 与调用路径](https://github.com/frenzymath/reap/blob/c6980f3418bc33eed291577aaf7830ab5de561e8/Reap/Tactic/Generator.lean)

这里的 categorical value 指将价值范围划分成离散类别，网络预测各类别概率，再得到所需的价值估计。它的要害不只是“类别数一样”，而是类别对应的数值、训练标签、读出方式和搜索公式是否一致。把成功率分桶与把负证明深度分桶，都可以称为 categorical，却不是同一个目标。

**比有没有一个头更重要的是数值契约。** 该版 Reap 对价值服务分数取负，搜索又对回传值使用基于证明步数的指数变换。由此至少可以确认：后端输出的方向和尺度会实质影响选择策略。假设后端返回的是“成功概率 0.9”，而客户端把它当成“剩余深度 0.9”再取负，代码可以正常运行，但数学含义已经错位。这里只是假设性反例，**不是本次发现某个公开后端确实犯了这个错误**。[Generator.lean](https://github.com/frenzymath/reap/blob/c6980f3418bc33eed291577aaf7830ab5de561e8/Reap/Tactic/Generator.lean) · [TreeSearch.lean 的 `computePUCTScores`](https://github.com/frenzymath/reap/blob/c6980f3418bc33eed291577aaf7830ab5de561e8/Reap/Tactic/TreeSearch.lean)

### 训练闭环边界与复用验证

还有一个工程细节：价值请求持续失败时，该客户端会退回 `-1000.0`。这是异常处理行为，不应悄悄把它当成真实模型预测，更不能把它当成监督标签去训练。实验日志应区分正常价值预测、服务错误与兜底值，否则一次后端故障可能被误解成“模型认为所有状态都极难”。[Generator.lean](https://github.com/frenzymath/reap/blob/c6980f3418bc33eed291577aaf7830ab5de561e8/Reap/Tactic/Generator.lean)

**它与完整 AlphaProof 的差距。** 已读 Reap 仓库提供的主要是 Lean 搜索、服务接入与轨迹输出；这并不等于同时公开了匹配的策略—价值联合训练、成功轨迹回放、matchmaker、目标变体生成、specialist 更新和测试时检查点管理。它可以成为这些系统的环境与搜索部分，但“能生成 RL rollout”与“自己包含完整 RL learner”仍然是两件事。[Reap 官方说明](https://github.com/frenzymath/reap) · [AlphaProof 原论文，Main RL、Focused RL](https://www.nature.com/articles/s41586-025-09833-y.pdf)

**如何判断价值是否好。** 以下是建议的验证，而非公开结果：固定策略模型、采样种子和预算，比较真实价值、常数价值、打乱状态对应关系后的价值；同时检查已完成证明上的深度估计、相近状态的排序，以及最终闭合率和耗时。若数值预测变准而闭合率没有提升，需要分析搜索公式或数据分布；若只是打开服务就改变了总预算，也不能把收益全归给 value。

结论是：**Reap 已有价值驱动搜索的实现；匹配价值模型的训练与质量，需要另查后端证据；完整 AlphaProof 的学习与 TTRL 闭环，则是比搜索客户端更大的对象。**

## DeepSeek-Prover-V2

### 模型定位与输出接口

**先明确版本和输出方式。** 本节只讨论 DeepSeek-Prover-V2，而不是把 DeepSeek-V3、DeepSeek 数学模型与其他版本的 Prover 混为一谈。V2 发布 7B 与 671B 两个规格；671B 基于 DeepSeek-V3-Base，7B 基于 DeepSeek-Prover-V1.5-Base。它们可以按不同提示使用 non-CoT 模式直接输出形式证明，或使用 CoT 模式先组织推理、再给出完整 Lean 证明。[论文](https://arxiv.org/html/2504.21801v1) · [官方仓库](https://github.com/deepseek-ai/DeepSeek-Prover-V2) · [7B 权重](https://huggingface.co/deepseek-ai/DeepSeek-Prover-V2-7B) · [671B 权重](https://huggingface.co/deepseek-ai/DeepSeek-Prover-V2-671B)

这与 StepProver 的“每次输出下一条 tactic”不是同一接口。一个 V2 候选可以包含多个 `have` 引理、自然语言解释和完整证明代码。要把它改作 MCTS 的策略模型，必须重新明确动作边界：一次动作是一条 tactic、一段局部证明，还是一个子目标方案？不能默认把整篇证明提示改成“写下一步”就得到了同等质量的逐步策略。

### 专家迭代、教师分解与蒸馏训练

**训练先解决怎样启动，再解决怎样变强。** V2 的第一部分是非 CoT 证明器的专家迭代：从现成数据和自动形式化题目出发，反复生成证明，把 Lean 验证成功的输出加入 SFT。这样得到的模型既积累形式化技巧，也能作为后面填补子目标的工具。[V2，§2.3，Expert Iteration](https://arxiv.org/html/2504.21801v1#S2.SS3)

第二部分是**大模型规划、小模型补证明**。DeepSeek-V3 提出高层思路，并把思路写成含中间引理的 Lean 草图；7B 证明器尝试完成这些子目标，必要时继续递归分解。原本 7B 无法端到端完成的题，如果分解后所有证明义务都完成，就可以组合出原题的完整形式证明。再将这份已完成的证明与高层推理组织成冷启动样本。[V2，§2.1--2.2](https://arxiv.org/html/2504.21801v1#S2)

这里的 cold start，冷启动，不是从随机权重开始预训练，而是让已有模型先学习“自然语言推理怎样对应到有效 Lean 证明”的示范格式。它降低的是后续 RL 的启动难度。论文还利用分解出的子目标构造训练课程，因此 V2 并非完全没有课程；只是这些课程属于其训练数据生产流程，不能自动等同于每次测试时针对当前目标题更新参数。[V2，§2](https://arxiv.org/html/2504.21801v1#S2)

第三部分在 671B 模型上混合非 CoT 证明数据与 CoT 冷启动数据进行 SFT，然后使用 GRPO，即 Group Relative Policy Optimization，组相对策略优化。它为同一道题生成一组候选，用 Lean 正误反馈形成相对奖励，不需要单独训练一个供该策略梯度使用的 critic。论文还在早期训练加入结构一致性奖励，使思路中的引理分解与最终证明中的 `have` 结构更一致。[V2，Reasoning-oriented Reinforcement Learning、§2.3](https://arxiv.org/html/2504.21801v1#S2)

**7B 最终版不是“原封不动的小工具模型”。** 它先把上下文从 4096 扩展到 32768 tokens，再学习 671B 的 RL rollout 数据，同时混入非 CoT 专家迭代证明；论文明确还对 7B 做了后续 RL。因此应区分用于前期数据收集的小证明器，与最终发布、经历蒸馏和 RL 的 V2-7B。[V2，§2.3，Distillation](https://arxiv.org/html/2504.21801v1#S2.SS3)

**“外部规划器当教师”究竟是什么意思？** 可以比较下面两种设计。在线设计中，每遇到新题，都调用大模型提出方案，再调用小模型补证明；大模型一直是运行链条的一部分。离线教师设计中，则先让大模型处理一批训练题，把规划与成功证明变成数据，用这些数据训练小模型。部署时由小模型自行产生所学的推理模式，不再必须在线调用原来的教师。前者把能力放在运行时的协作中，后者尝试把能力写进参数。

用一个教学例子说明：希望证明某个递推数列满足通项公式。教师提出“先猜通项，再证明递推保持该公式，最后用归纳收尾”；工具模型补齐归纳基和递推代数计算，合成完整证明。这份“为何引入这个中间命题 + 这些中间命题怎样用 Lean 证明”的记录，成为训练样本。学生模型以后遇到类似结构，可能自己提出同类分解。**这里学到的是生成规划的倾向，不是把一个固定规划文件缓存起来，也不保证学生获得教师全部能力。** 此例用于解释蒸馏机制，不是 V2 的公开实验样例。

这条路线对独立小模型的意义在于：训练阶段借助教师，并不逻辑上要求部署阶段继续依赖教师。但代价也很清楚：要付数据生成与训练成本，要筛掉只会复述却不可执行的思路，还要在没有教师参与的新题上测泛化。仅在蒸馏训练集上复现教师答案，不能证明学生已经掌握可迁移的规划能力。

### 与 AlphaProof 的搜索和学习闭环比较

**与 AlphaProof 的 SFT、主 RL 有什么不同？** 第一，示范来源和粒度不同。AlphaProof 的 Mathlib SFT 学习证明状态到 tactic，并初始化相应价值；V2 的关键冷启动样本包含教师组织的高层思路与完整证明。二者都可使用监督学习，但教给模型的对象不一样。[AlphaProof，Training](https://www.nature.com/articles/s41586-025-09833-y.pdf) · [V2，§2.2--2.3](https://arxiv.org/html/2504.21801v1#S2)

第二，主 RL 的改进操作不同。AlphaProof 用学习到的策略、价值和树搜索来发现成功证明，再训练状态—动作与价值；V2 的 GRPO 对完整候选输出按组相对奖励优化。前者将搜索中的局部状态显式用于价值学习，后者不需要同一种 MCTS critic。第三，教师承担的任务不同：V2 的冷启动重点是提供证明分解思路；AlphaProof 的变体生成器重点是提供可用于适应的相关命题，它不必为每个训练题先交出完整证明路线。[AlphaProof，Main RL、Variant generation](https://www.nature.com/articles/s41586-025-09833-y.pdf) · [V2，§2](https://arxiv.org/html/2504.21801v1#S2)

相似性同样不能忽略：二者都以形式验证为可靠性基础，都能从自身或工具产生的成功证明学习，也都利用难度调节来启动训练。所以不能简单总结成“V2 是 SFT，AlphaProof 才是 RL”。**正确差别在于训练样本怎样产生、搜索怎样改进策略、奖励怎样进入更新，以及这些更新发生在通用训练还是当前目标的测试阶段。**

### 测试表现与领域偏好

**竞赛和大学数学表现。** 在论文的修订版 miniF2F-test 上，CoT 模式的 V2-7B 为 pass@32 75.6%、pass@8192 82.0%；V2-671B 为 82.4% 和 88.9%。在 ProofNet-test 的 pass@1024 下，7B 为 29.6%，671B 为 37.1%。这些是各自大小模型的结果，不应把 671B 的成绩放到“可用 7B 基座”标题下而不作区分。[V2，表 1、表 4](https://arxiv.org/html/2504.21801v1#S3)

PutnamBench 更能体现难题边界。论文快照有 658 道题，其中实际运行时去掉 9 道 Lean 版本不兼容题，但成绩表仍保留 658 的分母。pass@1024 下，671B 的 CoT 模式解出 49 道；7B 的 non-CoT 模式解出 23 道，7B 的 CoT 模式反而是 11 道。不同配置结果合并得到的 62 道，不是 671B 单模型解出 62 道。[V2，§3.2](https://arxiv.org/html/2504.21801v1#S3.SS2)

这个反例很有启发：**显式长推理不是每个模型、每类题上都必然更好。** 论文还发现 7B 学会了一些大模型没有表现出的有限基数处理技巧。我们因此可以把小模型视为具有某些互补策略，而不能从一个高总分推断大模型在逐题上包含小模型的全部能力。[V2，Skill Discovery by Reinforcement Learning](https://arxiv.org/html/2504.21801v1#S3.SS2)

**领域偏好需要按证据说。** 作者明确指出训练以初等代数与数论为主，同时通过 ProofNet 展示向大学数学的泛化。它不是为某个研究生分析领域专门训练的模型。CombiBench 包含 100 道组合竞赛题；论文采用已在形式陈述中给出正确答案的 with-solution 设置，过滤后实际尝试 77 题，671B CoT 在 pass@16 下解出 12 题，表中记作 12/100。这说明组合题仍难，也提醒我们“给出答案并证明”不等于“独立发现答案”。[V2，§3.2--3.3](https://arxiv.org/html/2504.21801v1#S3.SS2)

V2 还发布 ProverBench：325 题中有 15 道来自 AIME 2024/2025 的代数、数论题，另有 310 道教材或教程题，覆盖线性代数、抽象代数、微积分、分析等。pass@512 的 CoT 模式下，7B 总体为 51.7%，671B 为 59.1%；AIME 子集分别为 1/15 和 6/15。这个补充说明“miniF2F 很高”仍不等于新竞赛题和所有教材领域都已经解决。[V2，§3.4](https://arxiv.org/html/2504.21801v1#S3.SS4) · [公开 ProverBench](https://huggingface.co/datasets/deepseek-ai/DeepSeek-ProverBench)

### 公开材料与复用判断

**公开范围和复用方式。** 官方明确公开两种规格权重、使用说明、ProverBench 和 miniF2F 解答文件。但本次核查的官方发布材料不足以确认全部 V2 冷启动数据、671B RL 轨迹和各阶段训练配方都完整公开；不能拿 V1 的公开数据来充当 V2 的全量训练数据。[官方仓库](https://github.com/deepseek-ai/DeepSeek-Prover-V2)

从公开研究角度看，V2-7B 适合作为现成的整篇证明或局部证明补全模型，也给出了“先离线蒸馏规划，再研究独立运行”的路线依据；它不是已经附带 AlphaProof 式价值头和目标专项 TTRL 的一体化替代品。是否把它改成逐步 MCTS 策略，需要另测接口转换后是否损失原有能力。

## Kimina：整篇证明与测试时训练

### 版本定位与完整证明接口

**首先把三个发布阶段拆开。** 2025 年 4 月的 Kimina-Prover Preview、7 月的 Kimina-Prover-72B 与 Test-Time RL Search、8 月的开源 Kimina-Prover-RL 训练配方，是相关但不同的材料。4 月论文不能单独为 7 月的 TTRL 实现作证；8 月公开训练代码也不能自动证明 7 月的所有测试时调度逻辑均已开放。[Preview 论文](https://arxiv.org/html/2504.11354v1) · [7 月技术说明](https://huggingface.co/blog/AI-MO/kimina-prover) · [8 月开源说明](https://huggingface.co/blog/AI-MO/kimina-prover-rl)

**它不只有 TTRL。** Kimina 的基础路线是长上下文、整篇证明生成与 RL。模型先生成组织数学思路的推理块，再给出完整 Lean 证明；所谓 Formal Reasoning Pattern，是让非形式推理与 Lean 操作在输出中形成对应关系。Preview 说明强调其基础生成并不依赖 MCTS、搜索价值函数或过程奖励模型。这里“不依赖 value”是另一条可行架构路线，不是“没有任何评测分数”。[Preview 官方说明](https://github.com/MoonshotAI/Kimina-Prover-Preview/blob/master/README.md)

**“生成时没有 Lean 反馈”不能误读成“训练不用 Lean”。** Preview 所指的是单次完整候选生成过程中不逐步调用 Lean；完成候选后，仍用 Lean 判定正确性并产生训练反馈。7 月工作又加入读懂 Lean 错误并修复的能力，因此不能拿 Preview 的单次生成设定，描述整个 Kimina 系列的所有运行模式。[Preview 论文](https://arxiv.org/html/2504.11354v1) · [7 月说明，Error-Fixing](https://huggingface.co/blog/AI-MO/kimina-prover)

### 通用训练：冷启动、RL 与蒸馏

**先看 Preview 怎样准备题目。** 数据生产从 NuminaMath 1.5 的非形式题目开始，经过筛选、自动形式化和人工修订。附录记载，约 10 万道自动形式化题与约 1 万道人工作过标注的题，通过提高人工题的采样频率形成约 20 万条训练条目。这里的 20 万不代表 20 万道互不重复的人工标注题。自动形式化模型还使用编译、语言模型语义判断和人工检查；Lean 能接受命题的写法，不等于自然语言题意已经翻译准确。[Preview，附录 C.1--C.3](https://arxiv.org/html/2504.11354v1#A3)

**冷启动教什么。** 作者让 Claude 3.7 Sonnet 根据已有非形式解答与完整 Lean 证明，组织约 2 万份配对推理样本，并混合 Kimi k1.5 的非形式数学思考数据进行 SFT。因此，它有“已有完整证明，再组织中间思路”的反向数据构造特点；与 V2 的“先提出草图，再补齐证明”顺序不同，但都试图对齐数学思路与可执行证明。[Preview，§2.2](https://arxiv.org/html/2504.11354v1#S2.SS2) · [V2，§2.2](https://arxiv.org/html/2504.21801v1#S2.SS2)

**主 RL 怎样进行。** Preview 论文的一个训练迭代抽取 1000 道题，每题生成 8 份完整候选，再根据 Lean 验证赋予正误奖励，按 Kimi k1.5 风格的策略更新训练模型。还用输出格式过滤和负向样本处理抑制格式崩塌。这里已经包含真正的参数优化，不只是“采样后挑出最好答案”；但也不能因后来开源配方采用 GRPO，就反向把所有早期版本的损失都改写成 GRPO。[Preview，§2.3](https://arxiv.org/html/2504.11354v1#S2.SS3)

7 月发布的 72B 模型基于 Qwen2.5-72B，8B 与 1.7B 蒸馏模型则属于 Qwen3 系列。小模型从大证明器的输出中学习，不能把 72B 的 TTRL 成绩直接视为这些小模型的能力。更不能认为下载一个蒸馏权重，就同时下载到了递归引理管理器和在线训练系统。[72B 模型卡](https://huggingface.co/AI-MO/Kimina-Prover-72B) · [8B 模型卡](https://huggingface.co/AI-MO/Kimina-Prover-Distill-8B) · [1.7B 模型卡](https://huggingface.co/AI-MO/Kimina-Prover-Distill-1.7B)

**7 月又补了哪些非 TTRL 训练。** 作者把大于 30 万题的初始题池筛成约 9 万道偏竞赛的问题，并采用动态筛选和难题分解；还报告约 60 亿 tokens 的 Lean 继续预训练材料，主要包含其 RL 流程产生的验证后轨迹，以及代码与状态转移数据。CPT 是 continued pretraining，即在已有模型上继续做领域训练，不是从随机权重重新训练模型。原文没有给出这些改动全部阶段之间唯一、完整的执行时间线，不能自行补成严格线性的训练顺序。[7 月说明，Other improvement](https://huggingface.co/blog/AI-MO/kimina-prover)

此外还有两类数据构造：把证明截断或挖掉内部片段，让模型补全；把“错误证明、Lean 报错、正确证明”组成修复示范，再训练模型解释并实施修正。后续 RL 会把上一轮的失败组织成下一轮的修复任务，与普通题目混合。这里的 failure replay 是“重新拿失败来练修复”，不是“把错误答案当成功证明模仿”。这些改动与 lemma-enabled pattern——利用输入中候选引理的推理模式——共同影响最终系统，不能把全部提升只归给 TTRL。[7 月说明，Error-Fixing、Other improvement](https://huggingface.co/blog/AI-MO/kimina-prover)

**为什么需要先训练“用引理”。** 给模型附上一堆数学上相关的命题，不代表它会在正确位置应用。Kimina 的训练输入随机加入少量候选引理，并围绕成功证明是否使用这些引理设计反馈。其目的不是让模型机械引用所有附加内容，而是让“附加引理 → 有效中间推理 → 完整证明”成为能够利用的行为模式。候选引理由通用模型产生，再经自动形式化模型转换。[7 月说明，Lemma enabled pattern](https://huggingface.co/blog/AI-MO/kimina-prover)

### 测试时引理搜索及其与 AlphaProof 的异同

接下来才是测试时搜索的核心。**下面关于 7 月 TTRL 的事实来自作者方法说明；未公开充分的优化器、检查点和调度细节，不用 8 月代码替它补齐。**

**第一步：为题目维护一个引理搜索范围。** 对目标 $T$，系统维护 $T$ 及其候选引理集合 $\mathcal L_T$，作者称为 search scope。重点不是单纯重复问同一道题，而是围绕当前目标，持续改变可供模型利用的中间命题。每个引理也可以有自己的 scope，于是搜索可以递归进行。[7 月说明，Test-Time Reinforcement Learning Search](https://huggingface.co/blog/AI-MO/kimina-prover)

**第二步：组合上下文，产生训练输入。** 作者说明，每轮 RL 对每个 scope 构造 $K=10$ 种输入：60% 使用利用评分较高的引理，40% 在这些引理外，再加入一至四条随机引理。这是在“利用已显示有效的内容”和“探索新组合”之间分配尝试。[7 月说明，TTRL Search](https://huggingface.co/blog/AI-MO/kimina-prover)

这里非常容易误读：**这十种 input variants 首先是引理上下文组合的变化，不应直接写成十道改写了结论或假设的数学变体题。** 目标仍可是同一个 $T$，但模型看到的辅助内容不同。这与 AlphaProof 大量生成不同形式命题的课程，相关却不相同。

**第三步：尝试证明，更新利用情况。** 系统跟踪 lemma utilization score，即候选引理在该 scope 中被使用、对证明构造起作用的情况。插入 50 次后仍达不到 0.10 利用评分的引理会被清除。这是作者给出的阈值，不是通用最优参数；公开说明没有完整交代所有评分归一化、计数边界和更新细节，不能自行补出一套精确公式冒充原实现。[7 月说明，TTRL Search](https://huggingface.co/blog/AI-MO/kimina-prover)

这个评分与 value 网络应分开：它衡量“某条引理对某个搜索范围有没有用”，而不是预测“任意 Lean 状态距离完成还需要多少步”。它也不等同于 AlphaProof matchmaker：后者主要调度题目与搜索预算，这里主要维护引理组合的选择。二者都分配算力，但选择对象和信息来源不同。

**第四步：卡住时继续分解。** 当某个定理或引理经历 128 次尝试仍未找到证明，系统生成新的候选子引理，并行的子引理生成过程也持续工作。于是，一个原目标不再只有“成功或失败”两种后续，而可以转入“提出辅助命题、证明辅助命题、重新组织原证明”的递归过程。[7 月说明，TTRL Search](https://huggingface.co/blog/AI-MO/kimina-prover)

**第五步：排除已证伪的候选。** 新引理会接受证否尝试；若其否定被证明，就丢弃该引理。需要补充一个逻辑上不可省略的边界：**没有证明出否定，不等于已经证明了引理为真。** 作者把证否筛选描述为提升可靠性的方法，但仅靠这一过程，不能推出所有保留引理都正确。[7 月说明，Negation proving](https://huggingface.co/blog/AI-MO/kimina-prover)

用一个抽象例子说明：系统想通过 $L_1$、$L_2$ 证明 $T$。即使已经验证了 $L_1\to L_2\to T$，只要 $L_1$、$L_2$ 尚未被证明，就仍然只是条件性证明。最终必须补齐所有真正使用的引理，并检查完整的 $T$ 不依赖未证明的占位符或新增假设。这个例子是逻辑解释和验收要求，**不是声称已逐行检查 Kimina 的完整内部引理合成器**。

**第六步：测试阶段确实包含训练，而不仅是缓存。** 作者明确把上述输入组合放在 RL training iteration 中，并称之为可训练的测试时搜索框架。因此应承认它报告了测试时 RL 机制，而不是因为规模或架构不同就说“没有 TTRL”。但公开方法说明没有完整提供：每轮确切更新哪些参数、目标之间是否共用同一条检查点链、优化器状态怎样继承、是否有回退，以及全部训练样本如何从引理搜索记录进入优化器。对这些问题，应写“尚未由所核查材料确认”。[7 月说明](https://huggingface.co/blog/AI-MO/kimina-prover)

**与 AlphaProof 的相同点。** 两者都针对当前困难目标组织相关任务，利用形式验证产生反馈，并在测试阶段继续调整证明模型，而不只是增加固定模型的采样次数。两者也都需要避免将错误的辅助命题当作真，并需要一定数量的可成功尝试才能启动有效学习。[AlphaProof，Focused RL](https://www.nature.com/articles/s41586-025-09833-y.pdf) · [Kimina，TTRL Search](https://huggingface.co/blog/AI-MO/kimina-prover)

**与 AlphaProof 的第一项差别：相关任务怎样服务目标。** AlphaProof 的课程变体主要通过训练改变策略和价值，然后再尝试原题；变体不必是原题的必需引理。Kimina 强调候选引理组合和递归子引理，许多中间结果被设计为证明原题的显式工具。因此，前者更突出“从相关题学习”，后者更突出“边学习边构造可复用的证明部件”；这不是互斥分类，AlphaProof 的变体也可以包含引理或分解。[AlphaProof，Variant generation](https://www.nature.com/articles/s41586-025-09833-y.pdf) · [Kimina，TTRL Search](https://huggingface.co/blog/AI-MO/kimina-prover)

**第二项差别：底层证明与评价。** AlphaProof 的核心是逐步 tactic、共享策略—价值网络和 AND–OR MCTS；Kimina 的基础模型侧重推理后生成完整证明，上层组织引理。已核查 Kimina 材料不能证明它使用了 AlphaProof 的 64 类负证明深度 value、同一 MCTS 回传，或同一 matchmaker。不能把引理利用评分直接改名为原版 value，也不能把 scope 改名为原版搜索节点就认定结构一致。[AlphaProof，Prover agent、Tree search](https://www.nature.com/articles/s41586-025-09833-y.pdf) · [Kimina Preview](https://arxiv.org/html/2504.11354v1) · [Kimina 7 月说明](https://huggingface.co/blog/AI-MO/kimina-prover)

**第三项差别：失败样本怎样使用。** AlphaProof 原文的网络学习使用成功证明和证否；失败尝试用于调度。Kimina 的公开 RL 配方则使用组相对奖励，并把失败响应与 Lean 错误构造成修复输入。失败不是被标成“正确证明”，而可以成为负向相对反馈或下一次需要修复的任务。**“原版 AlphaProof 只从成功轨迹更新”是一个具体算法选择，不是所有形式化 RL 都不得使用失败反馈的定律。** 但 8 月配方的具体损失，不能未经证据直接归给 7 月 TTRL 的每个内部阶段。[AlphaProof，Main RL](https://www.nature.com/articles/s41586-025-09833-y.pdf) · [Kimina 开源启动脚本](https://github.com/project-numina/kimina-prover-rl/blob/e16b605e8186614c685875c9b57eb19e841b521a/recipe/kimina_prover_rl/kimina_prover_1.7B.sh)

用一个简化的组相对奖励例子解释差别：同题四个候选奖励为 $(1,1,0,0)$，组均值为 $0.5$。仅看中心化奖励 $A_i=r_i-\bar r$，成功候选对应正值，失败候选对应负值。失败因而可以参与降低相对概率，但没有被当成正确示范。若全组都是 0，这个简化的任务奖励信号就全为 0；“只会生成错误答案”仍不能凭空提供成功证明技巧。这是对组相对学习的教学解释，不是完整 GRPO 损失，更不是对 Kimina 7 月内部实现的还原。

**第四项差别：公开细节与实测口径。** AlphaProof 有论文、伪代码和系统方法描述，但不等于完整生产训练系统开放；Kimina 有权重、方法说明和后续开源 RL 配方，但后者也不等于完整 TTRL 闭环。两者应比较“已经公开哪一层”，不能一方按论文全部能力计，另一方只按眼前脚本计。性能比较同样必须计入变体或引理生成、搜索、训练和验证成本，而不是只数对原题的直接尝试。

### 开源训练代码与数据公开边界

**8 月代码到底公开了什么。** 这次补充核查了 [`project-numina/kimina-prover-rl`](https://github.com/project-numina/kimina-prover-rl/tree/e16b605e8186614c685875c9b57eb19e841b521a)，固定提交为 `e16b605e8186614c685875c9b57eb19e841b521a`。真正的配方目录是 `recipe/kimina_prover_rl`，使用下划线；不能仅阅读仓库根目录的通用 verl README，就判断没有 Kimina 训练代码。

源码中的 `kimina_prover_1.7B.sh` 以已经蒸馏过的 1.7B 模型为起点，配置组相对优势估计、每题多个 rollout、32K 总序列长度和错误修复样本混合，调用 verl 的训练入口。verl 是用于组织大模型强化学习的训练框架；其中出现 `ppo` 的通用入口名，也不表示实际必定启用了独立 PPO critic，应看具体 `adv_estimator=grpo` 等配置。[启动脚本](https://github.com/project-numina/kimina-prover-rl/blob/e16b605e8186614c685875c9b57eb19e841b521a/recipe/kimina_prover_rl/kimina_prover_1.7B.sh)

`reward.py` 实际向 Kimina Lean Server 提交候选，记录验证结果，再将证明正确性奖励与格式检查结果相乘。`dataset.py` 则维护多轮数据缓存，让部分训练输入来自先前失败响应及其 Lean 反馈，而不是只提供一段“未来支持修复”的注释。这里可以确认已公开**训练入口、验证奖励和错误修复数据路径**；本次没有实际启动这套 GPU 训练，也没有完成底层 verl 的逐行审计。[奖励代码](https://github.com/project-numina/kimina-prover-rl/blob/e16b605e8186614c685875c9b57eb19e841b521a/recipe/kimina_prover_rl/kimina_prover_rl/reward/reward.py) · [数据代码](https://github.com/project-numina/kimina-prover-rl/blob/e16b605e8186614c685875c9b57eb19e841b521a/recipe/kimina_prover_rl/kimina_prover_rl/dataset.py)

这套配方的普通训练与评测题目是分开配置的。它证明 Kimina 不是“只有博客、没有公开训练管线”，但这些已读入口本身不能证明完整发布了 7 月那套 scope、引理利用评分、递归生成、测试目标适应和检查点继承逻辑。更准确的定位是：**公开的通用 RL 与修复训练配方可供研究；完整 7 月测试时引理搜索系统的可复现程度，仍需单独核查。**[配方 README](https://github.com/project-numina/kimina-prover-rl/blob/e16b605e8186614c685875c9b57eb19e841b521a/recipe/kimina_prover_rl/README.md)

**数据和工具能拿到哪些。** 权重包括 72B、蒸馏小模型及后续 RL 小模型；公开数据有 [NuminaMath-LEAN](https://huggingface.co/datasets/AI-MO/NuminaMath-LEAN) 和 [Kimina-Prover-Promptset](https://huggingface.co/datasets/AI-MO/Kimina-Prover-Promptset)，公开验证环境有 [Kimina Lean Server](https://github.com/project-numina/kimina-lean-server)，公开训练配方如上。Promptset 主要提供用于产生 rollout 的问题输入，不意味着每一道题都附带原训练全部成功、失败和修复轨迹。完整 72B 训练数据、所有蒸馏样本、各目标 TTRL 运行轨迹，也不能因为某个数据集已公开就一并视为公开。[开源发布说明](https://huggingface.co/blog/AI-MO/kimina-prover-rl)

### 测试表现、任务侧重与复用判断

**能力表现必须分模式。** 4 月 Preview 的代表结果是其修订版 miniF2F-test 上约 80.7% pass@8192，不能与后续 TTRL 数字混成同一模型同一预算。[Preview，§3](https://arxiv.org/html/2504.11354v1#S3) 7 月作者说明报告，72B 在 miniF2F-test 上 pass@32 为 84.0%，加入一次错误修复为 86.4%，pass@1024 为 87.7%。完整 TTRL Search 报告 92.2%，但同时估计其 pass 预算上界约为 42,000，并承认大量尝试用于无用或冗余引理。**92.2% 不能写成冻结 72B 的 pass@1024，也不能据此声称比其他方法更省算力。** [7 月结果说明](https://huggingface.co/blog/AI-MO/kimina-prover)

小模型成绩还有需要保留的来源差异：7 月博客给 8B 的 pass@32 为 78.3%，对应模型卡写 77.86%；8 月 RL-1.7B 的模型卡与配方 README 写 76.63%，但博客结果表写 76.23%。这些差别不大，却真实存在；本次没有取得足够逐次评测信息解释原因，不能擅自认定是随机波动、修订题集或笔误。组会上引用时应指定“模型卡报告”或“博客某表报告”，不要合并成一个伪精确数值。[8B 模型卡](https://huggingface.co/AI-MO/Kimina-Prover-Distill-8B) · [RL-1.7B 模型卡](https://huggingface.co/AI-MO/Kimina-Prover-RL-1.7B) · [8 月结果表](https://huggingface.co/blog/AI-MO/kimina-prover-rl)

**领域偏好与未证实之处。** 其模型卡明确面向竞赛型 Lean 证明，最充分的公开成绩集中于 miniF2F。更具体地，Preview 的自动形式化候选筛选排除了几何和组合题，7 月题池又进一步强调竞赛来源；这是训练分布偏好的直接依据，但不能扩张成“整个系列所有版本完全没接触过组合或几何”。同样，也不能据此推断大学分析、拓扑或研究级抽象代数的相对优势。[Preview，附录 C.1](https://arxiv.org/html/2504.11354v1#A3.SS1) · [7 月题池说明](https://huggingface.co/blog/AI-MO/kimina-prover) Preview 在 CombiBench 的结果属于较早版本，也不能直接代表 7 月 72B 或 8 月 RL 小模型。当前可说的是“具有强的竞赛型证明证据，以及目标相关引理训练的方法报告”，不是“所有大学数学领域均强”。[模型卡](https://huggingface.co/AI-MO/Kimina-Prover-72B) · [Preview 论文](https://arxiv.org/html/2504.11354v1)

因此，Kimina 对本主题有两种不同价值：一是研究“显式引理组织如何与测试时参数更新结合”；二是利用已公开的小模型、问题集和 RL 配方，研究可验证奖励与错误修复。第一种路线仍有工程公开边界，第二种已有训练入口、验证奖励与修复样本路径的源码依据。

## LEAP：证明依赖图代理

### 定位与问题分解

**先给出区别：LEAP 更强调管理一张证明依赖图，Nexus 的完整配置更强调维护、比较和改进一批候选证明草图。** 两者都把大模型放在推理和工具调用链条中，都可以拆子目标、复用成果；但它们不是同一个搜索算法，也不是两个可下载的小型证明基座。下面先解释共同问题，再分别看实现思路。

**为什么“拆成简单题”还不够。** 为证明 $T$，模型提出两个看起来容易的命题 $A$、$B$。至少有三件事必须分开检查：$A$、$B$ 能不能证明；即使它们成立，是否足以推出 $T$；这种分解是否真的比直接证明 $T$ 更省资源。第一、二项最终需要形式证明，第三项则是搜索效率问题。仅凭语言模型说“这两个引理应该有帮助”，三项都没有完成。

**LEAP 怎样把第二项先固定下来。** LEAP 全称 LLM-in-Lean Environment Agentic Prover。它先用通用大模型直接尝试证明，借助 Lean 报错修订、检索库引理；直接尝试失败后，再生成自然语言蓝图，并翻译成含辅助引理的 Lean 草图。要求是：主目标的证明体不能留 `sorry`，占位符只能留在新提出的辅助引理处。这样，草图先建立“如果这些辅助引理完成，主目标就随之完成”的可检查依赖。[LEAP，§2](https://arxiv.org/html/2606.03303v2#S2)

这里 `sorry` 表示尚未补齐的证明，不是合法完成结果。可以把这一步理解为先验证了 $A\to B\to T$，然后继续履行 $A$、$B$ 的证明义务。它不是“已经证明了 $T$”，而是把原先含糊的思路变成了一份准确的待办清单。若最后补不上 $B$，就必须继续处理它或换一个方案，不能因为主证明段落已经能编译就宣布完成。

### 依赖图、回溯与方案审查

**AND–OR DAG 怎样组织这份清单。** DAG 是 directed acyclic graph，即有向无环图。LEAP 中，OR 节点是一项待证明目标，可以有多个替代方案；AND 节点是其中一个分解方案，要求它引用的全部子目标都完成。相同子引理可以被多个方案共享，而不是复制成互不相干的任务。[LEAP，§2.3](https://arxiv.org/html/2606.03303v2#S2.SS3)

例如有两条路线：路线甲依赖 $A,B$，路线乙依赖 $A,C$。证明 $T$ 不必同时完成两条路线，所以路线之间是 OR；选定路线甲后，$A$ 和 $B$ 都不可缺，所以该路线内部是 AND。已经证出的 $A$ 可以同时供两条路线使用。若 $B$ 很难、$C$ 较容易，搜索可以转向乙，而不必重新证明 $A$。这是依赖图的教学例子，不是论文中的具体题目。

**防循环与防无效分解不是一回事。** 如果证明 $A$ 又依赖 $B$，证明 $B$ 又依赖 $A$，没有独立已知依据就不能完成它们；无环检查阻止把这种循环计作进展。但即使没有环，把 $T$ 换成一个只改了名字、难度几乎不变的 $T'$，仍然没有实质帮助。因此 LEAP 另用大模型 reviewer——方案审查器——评估子目标的相关性、难度和可行性。它是启发式过滤器，不是保证子目标更容易的数学定理。[LEAP，§2.5](https://arxiv.org/html/2606.03303v2#S2.SS5)

这就解释了“子目标真的支持原目标”：不是要求模型的自然语言判断绝对正确，而是用形式草图约束**逻辑支持关系**，用审查和实际搜索评估**难度下降**。二者相互配合，不能互相替代。图中还可以保存暂时没有被当前证明使用的预备引理，但这些预备引理不应被误记为完成当前方案必须解决的义务。[LEAP，§2.3--2.5](https://arxiv.org/html/2606.03303v2#S2)

### 与 AlphaProof 的搜索和参数学习比较

**它是不是 MCTS 或 TTRL？** 论文描述的图搜索采用 DFS，即深度优先搜索，并允许回溯；不能看到 AND–OR 就改称 AlphaProof 的 MCTS。其运行中变化的是草图、依赖图、已验证引理和反馈上下文；所述方法没有通过当前目标的梯度更新来训练一个新的 LEAP specialist。因此它属于测试时代理搜索，而不是论文已经报告了 AlphaProof 式 TTRL。[LEAP，§2.5](https://arxiv.org/html/2606.03303v2#S2.SS5)

### 测试表现与研究级案例边界

**LEAP 的能力与范围。** 论文使用 Gemini 3.1 Pro 作为后端。在 Lean-IMO-Bench 的 30 道 Basic 题上解出 25 道，在 30 道 Advanced 题上解出 17 道，合计 42/60。Basic 从预 IMO 到 IMO 中等难度，Advanced 延伸至 IMO 高难度；两组均覆盖代数、组合、数论与几何。论文的代理评测设置为每题两次完整 rollout，不是仅调用两次大模型。[LEAP，§3--4](https://arxiv.org/html/2606.03303v2#S3)

领域结果也有必要展开：这批小样本中的代数和数论均全部解决；Advanced 的组合为 2/8，几何为 1/8。它说明该实验中的能力分布不均衡，但不足以推出“所有代数题都会”或“组合能力普遍只有 25%”。论文另报告 Putnam 2025 的 12 题全部解决，公开了对应 Lean 证明；成本表显示单题可以包含几十到数千次模型调用。**这是整个代理系统在给定预算下的能力，不是把 Gemini 当作冻结单次生成器时的能力。**[LEAP，表 2--4](https://arxiv.org/html/2606.03303v2#S4) · [官方证明结果](https://github.com/google-deepmind/superhuman/blob/main/leap/README.md)

研究级案例还应区分“发现新结论”和“形式化已有思路”。公开材料包含 Knuth 相关问题的一个关键子问题，以及 Erdős 457 的形式证明；个别案例提供了非形式背景或证明线索。不能仅凭文件放在 Open-Problems 目录，就说系统在完全无提示的条件下独立解决了该研究方向的整个开放问题。[LEAP，§6 与附录 C](https://arxiv.org/html/2606.03303v2) · [结果目录说明](https://github.com/google-deepmind/superhuman/blob/main/leap/README.md)

### 公开结果与复用判断

[LEAP 官方目录](https://github.com/google-deepmind/superhuman/tree/main/leap)提供论文配套 Lean 证明与题目入口，不能据此认定代理实现、模型权重及其完整训练数据已开放。可复用的是草图验证与证明义务管理思路；若目标是独立小 prover，需要另行收集、蒸馏并评估规划记录。

## Nexus：草图演化与证明工具协作

### 定位与草图演化搜索

**Nexus 怎样组织搜索。** Nexus 全功能配置让多个证明子代理围绕候选 Lean 文件工作。子代理可以补证明、调整草图、提出或处理子目标；候选草图进入共享种群，评分代理比较其策略清晰程度、剩余目标可行性等，再汇总成 Elo 式相对评分，引导后续选择。这里 evolutionary search，演化搜索，指让候选方案经选择、修改和保留不断变化，不是用梯度更新模型权重。[Nexus，§2、附录 A.1](https://arxiv.org/html/2605.22763v2#S2)

Nexus 的完整配置还可调用 AlphaProof 作为局部证明工具，获得证明、证否或预算内未解决的反馈。但论文也研究了不带 AlphaProof 的基础代理；附录明确说调用的是 AlphaProof 较低成本的树搜索推理模式。因此，**不能把 Nexus 的每个子目标调用想象成又启动一次完整 AlphaProof TTRL**，也不能说所有 Nexus 配置都必须有 AlphaProof。[Nexus，AlphaProof Budget、Related Work](https://arxiv.org/html/2605.22763v2#A1)

### 与 LEAP 和 AlphaProof 的异同

**二者真正相似与不同在哪里。** 相似处是用自然语言提出方案、用 Lean 约束结果、对失败进行反馈处理，并保留中间进展。不同处是 LEAP 把“哪个目标依赖哪些子目标”作为显式组织核心，Nexus 完整版把“一批草图中接下来改进哪一份”作为演化调度核心；LEAP 的论文配置用通用模型完成规划与形式证明，Nexus 完整版还整合专门的 AlphaProof 工具。这不是说 Nexus 完全没有依赖管理，也不是说 LEAP 不能有多个候选计划，而是各自设计着力点不同。[LEAP，§2](https://arxiv.org/html/2606.03303v2#S2) · [Nexus，§2、附录 A.1](https://arxiv.org/html/2605.22763v2#S2)

可以用一个比喻理解，但不要把它当作源码描述：LEAP 更像维护证明工程的依赖清单，确保做完哪些工作就能交付；Nexus 更像保留多份正在发展的证明草稿，持续选择有希望的草稿交给代理修改。两种组织方式可以结合，但结合之后仍需要独立设计调度、去重、验证和预算，不能认为把两个仓库接起来就自然完成了。

**它们有没有在测试时学习？** 广义地说，两者都利用反馈改变后续行为；但这里主要是上下文、缓存和搜索状态的变化，不能与参数更新混用。Nexus 方法描述没有把当前测试过程组织成“收集轨迹 → 更新证明模型参数 → 发布新检查点”的训练链。其成果是证明文件和搜索结果，不是一个由这次任务新训练出的、可以脱离原模型后端的 Nexus 权重。[Nexus，Materials and Methods](https://arxiv.org/html/2605.22763v2#A1)

### 研究级测试表现与成本边界

**Nexus 的能力证据属于另一种评测。** 论文报告，在当时尝试的 353 个已形式化 Erdős 开放问题中解决 9 个，在 492 个 OEIS 猜想中解决 44 个。OEIS 是整数数列百科；这批任务经过筛选与形式化，不是从所有数学开放问题中均匀随机抽样。成功后还需要专家核对 Lean 命题是否忠实表达原问题。因此，它提供研究级任务证据，却不能直接换算成 miniF2F 或大学教材的总体水平。[Nexus，§3、附录 B.2](https://arxiv.org/html/2605.22763v2#S3)

论文对那 9 个已成功 Erdős 问题做事后比较时，基础代理也能解决它们，但困难问题成本更高；这个比较只覆盖挑出的已成功集合，不能据此推断基础代理在全部 353 题上和完整版一样。关于“是否更便宜”，还必须计入未解题的筛选成本、AlphaProof 计算和重试；部分图表单列或未计入 AlphaProof 推理成本，不能拿其中一个美元数字直接与原版 TTRL 比较。[Nexus，Comparative Analysis、Cost and Variance](https://arxiv.org/html/2605.22763v2#S5)

### 公开结果与小模型复用边界

**公开范围与借鉴方式。** LEAP 的已读公开目录是论文和解答材料；Nexus 的结果仓库提供成功 Lean 证明、部分自然语言证明以及尝试题目入口。它们不等于完整代理源码、模型权重和训练数据一并开放。[LEAP 结果目录](https://github.com/google-deepmind/superhuman/tree/main/leap) · [Nexus 结果仓库](https://github.com/google-deepmind/alphaproof-nexus-results)

从模型研究角度看，可以借鉴它们的证明义务管理，也可以尝试把成功的“规划---子目标---完整证明”记录蒸馏成训练数据。但后者是**从这些框架出发提出的研究路线**，不是两篇论文已经证明“一个小模型经这样蒸馏便能达到同样研究能力”。能在线调度大模型，与能把能力迁移到小模型，是不同实验。

## HTPS / Evariste：搜索与在线训练前驱

### 定位与超图证明搜索

**它为什么值得详细看。** HTPS 是 HyperTree Proof Search，超树证明搜索，出自 2022 年工作；Evariste 是相应系统与代码仓库的名称。它早于 AlphaProof，不是后来的复制项目。相较于纯代理草图系统，它在“学习到的策略和 critic 引导搜索、搜索反过来产生训练数据”这一层，与 AlphaProof 的问题设置更直接相关。[HTPS 原论文](https://arxiv.org/pdf/2205.11491) · [Evariste 仓库](https://github.com/facebookresearch/Evariste)

**什么是超树。** 普通有向边连接一个起点与一个终点；证明中的一条 tactic 却可能把一个目标变成多个都必须完成的子目标。HTPS 用一条 hyperedge，超边，把“父目标、这条 tactic、整组子目标”一起表示。不同 tactic 是替代路线，同一 tactic 产生的子目标则是共同义务；搜索图还能共享相同目标。最终挑选出的完整证明结构称为 proof hypertree。[HTPS，§4](https://arxiv.org/pdf/2205.11491)

例如一条 tactic 把 $T$ 变成 $A$、$B$、$C$，其成功条件是三者全部完成，而不是从三者任选一个。这与 AND–OR 的基本逻辑一致；但“表达同样的逻辑关系”不代表各系统使用相同价值函数、搜索选择和回传公式。

### 策略、critic 标签与回传语义

**策略、critic 与训练标签。** 该工作使用共享参数的编码器—解码器网络：策略输出 tactic；critic 通过两个特殊输出类别 PROVABLE 和 UNPROVABLE 的概率形成评分，意图衡量目标可完成的程度。它不是 AlphaProof 的负剩余分支深度。训练时，成功节点、无效节点、充分访问的中间节点采用不同标签；中间节点可以使用搜索估计的软目标，而不是把所有“本次未证明”都标成零。[HTPS，§5](https://arxiv.org/pdf/2205.11491)

为什么这点重要？“没有搜到”既可能意味着模型很弱、预算不足，也可能意味着当前路径真的不好。把三种原因统统变成“不可证明”，会得到过于悲观的训练信号。但采用搜索估计也不是获得了绝对真值：估计仍受当前模型与搜索预算影响，必须靠实验检验其引导效果。这是软目标的用途与边界，不是说它一定优于任何其他价值目标。

**回传也不能混用。** HTPS 对一个所选证明分解的子目标评分使用乘积式组合，AlphaProof 的负分支深度则涉及 AND 节点的最小回报。举例，若两子目标的完成概率估计分别为 0.8 和 0.5，乘积为 0.4；若两子目标的剩余深度为 2 和 7，最长分支深度是 7、对应负回报为 -7。前者尝试表达全部完成的可能性，后者表达最长分支代价；即便都叫 value，也不能替换数值后沿用另一套解释。乘积作为精确联合概率还需要额外依赖假设，搜索中它首先是一种组合估计。[HTPS，§4.3](https://arxiv.org/pdf/2205.11491) · [AlphaProof，Tree search](https://www.nature.com/articles/s41586-025-09833-y.pdf)

### 在线训练与目标适应

**闭环是怎样运行的。** HTPS 从能输出合理 tactic 的监督模型启动，随后让异步 prover 搜索，训练器从搜索图提取策略与 critic 样本，prover 再取得更新后的模型。策略样本取自根目标的成功证明，并偏向其最小证明结构；值得注意的是，Lean 实验的“最小”按 tactic 总 CPU 时间定义，不是默认按最少行数或最短字符串定义。[HTPS，§5](https://arxiv.org/pdf/2205.11491)

**它已经有测试目标上的训练吗？** 论文明确研究 transductive online training，即训练时提供待解决目标的陈述，让系统在这些目标上搜索、产生数据并继续训练。因而在宽泛的“针对当前待解目标继续更新参数”意义上，它是相关先例；不能说所有早期工作都只有冻结搜索。但是，它不等于已经具有 AlphaProof 后来的大规模目标变体生成与 specialist TTRL 全套机制。[HTPS，§6.3、§7.1.1](https://arxiv.org/pdf/2205.11491)

### 测试表现与历史环境边界

**成绩必须分清训练目标与未见测试题。** 论文直接将 miniF2F-valid 陈述纳入在线训练后，累计解决率是 58.6%；随后最终模型在未参与该在线训练的 miniF2F-test 上得到 41.0% pass@64。58.6% 是训练期间曾成功过的累计覆盖，不是冻结模型在完全未见测试集上的成绩。相应实验也使用了较大并行资源，不应把“早期开放代码”误读成“低成本即可复现”。[HTPS，表 2 与 §7.1.1](https://arxiv.org/pdf/2205.11491)

作者还在 Metamath 和 Equations 环境中实验：前者是另一种形式证明系统，后者是用于研究等式推导的实验环境。它们的高成功率不能当作 Lean 竞赛题成绩，更不能推出大学分析或抽象代数的专长。本文引用的 Lean 成绩主要反映当时的库证明与 miniF2F 分布。[HTPS，§3、§7](https://arxiv.org/pdf/2205.11491)

### 归档代码与复用判断

**今天能利用什么。** Evariste README 明确说明代码属于归档材料，移除了 Meta 内部系统引用后，并不能直接开箱运行。它适合阅读超图搜索、软 critic 标签、异步采样—训练和数据筛选机制；从其历史 Lean 环境迁移到新的 Lean 4 系统仍是一项工程。已公开论文与代码，不等于完整权重、全部训练轨迹、内部基础设施和一键复现实验均已公开。[仓库说明](https://github.com/facebookresearch/Evariste/blob/main/README.md)

## LeanDojo / ReProver：数据基础设施与检索证明

### 平台定位与模型架构

**先区分平台与模型。** LeanDojo 是数据提取、程序化 Lean 交互和评测基础设施；ReProver 是建立在其上的检索增强证明器。两者不是另一个拥有 AlphaProof 全套训练机制的系统，却很好地展示了如何从数学库得到可训练数据，以及如何在使用引理时控制可见范围。[论文](https://arxiv.org/html/2306.15626v2) · [LeanDojo](https://github.com/lean-dojo/LeanDojo) · [ReProver](https://github.com/lean-dojo/ReProver)

**模型怎样工作。** ReProver 用检索器挑选与状态相关的库引理，再把引理与状态一起输入 tactic 生成器，配合 best-first search 找证明。原论文的生成器基于约 299M 参数的 ByT5-small；检索器使用编码器部分，生成器使用编码器与解码器。ByT5 按文本字节处理输入，避免某些数学 Unicode 符号无法被词表良好表示的问题，但字节序列也可能更长。[ReProver，§5、附录 C.1](https://arxiv.org/html/2306.15626v2#S5)

### 监督训练与 AlphaProof 的异同

**它怎样训练，为什么不是 value。** 检索器学习把证明状态与有用引理表示成相近向量；策略模型则监督学习人工证明中的 state–tactic 对。原论文没有靠在线 RL 不断更新，也没有用一个 AlphaProof 式价值模型指导回传。检索匹配分数表示“这条引理与当前状态相关”，不是“完成当前证明的回报”。这一点与 REAL-Prover 的检索模块相同，不能把二者的检索器直接当成 critic 使用。[ReProver，§5](https://arxiv.org/html/2306.15626v2#S5)

### 训练数据与前提可见范围

**数据设计有一项特别值得借鉴。** 原版 LeanDojo Benchmark 从 Mathlib 提取 98,734 个定理与证明，并设置 novel-premises 划分：测试证明需要使用训练证明中未用过的引理。这里并不是禁止测试时从库中检索这些引理，而是在问“模型能否运用没有在训练用法中见过的库知识”。若只随机划分相近定理，模型可能凭记忆相似证明取得较高成绩，无法充分测量这一能力。[LeanDojo，§4](https://arxiv.org/html/2306.15626v2#S4)

举例，训练轨迹没有使用某个连续映射的闭集原像定理，测试时该定理仍然是已经证明、可供使用的库知识。模型需要根据当前目标检索到它，并匹配假设完成应用。这是“新引理使用”的泛化，不是让模型在测试时偷偷读到了当前目标的答案。相反，若允许直接检索到当前待证明定理本身、或者依赖它才成立的下游定理，就会破坏评测；因此可访问前提的确定不是一个无关紧要的实现细节。

### 测试表现与预算

**能力和评测边界。** 原论文在 miniF2F-test 上得到 26.5% pass@1，但一次 attempt 包含最多十分钟的 best-first 搜索，不是只生成一条 tactic。ProofNet 的 13.8% 对应该论文当时可运行的 349 题中解出 48 题，不能与后来仅在 186 道 test 题上的数字直接比较。这些结果属于早期、较小的检索型模型，没有充分分学科证据支持“分析比代数强”等结论。[ReProver，§6、附录 C.4](https://arxiv.org/html/2306.15626v2#S6)

### 公开材料与复用判断

**公开程度比较具体。** 当前 ReProver README 提供模型权重、数据下载、检索器训练、生成器训练和评测入口；主分支面向 Lean 4，旧 Lean 3 工作放在 legacy 分支。因此必须把原论文历史结果与后来主分支的环境分开。它的主要复用价值是清楚的数据---检索---交互流程，而不是用旧成绩宣称今天的最强基座。[官方 README](https://github.com/lean-dojo/ReProver/blob/main/README.md) · [Lean 4 检索增强模型](https://huggingface.co/kaiyuy/leandojo-lean4-retriever-tacgen-byt5-small)

## match maker 是否还没有开源复现？

### 明确回答与原版职责

**不能再回答“还没有开源复现”。** 原版 AlphaProof 的生产 matchmaker 未在已核查材料中形成可下载的完整实现证据，但 nanoproof 的固定提交已经提供一个实际接入 RL 的、只做证明的 AlphaProof 风格实现。这里要区分官方生产源码、开源机制实现和完整实验复现三个结论。

原版 matchmaker 从训练题库选题，分配证明或证否任务，再按近期结果调整优先级和每次搜索预算；它不是给一个 Lean 状态预测 value 的网络。其职责贯穿主 RL 和目标变体 TTRL，原目标成功后还会停止该目标及相关变体的新任务。[原论文 Main RL、Focused RL](https://www.nature.com/articles/s41586-025-09833-y)

### nanoproof：已有代码与原版差异

固定提交 `4f410c6e7a820557501848f954402514201322a5` 的 [`experience_collection.py`](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/experience_collection.py) 定义 `MatchmakerConfig`、`TheoremStats` 和 `Matchmaker`。`next_assignment()` 返回题目与 simulation 预算，`send_result()` 更新结果历史；[`rl.py`](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/rl.py) 实例化并安装它，[`prover.py`](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/prover.py) 在收集阶段取任务并回报结果。因此证据不止是一个类名或 TODO；本次确认的是静态控制流，没有重跑分布式训练。

它按历史结果给未尝试、尝试不足或有成功但未连续掌握的题较高采样权重，降低始终未成功或已掌握题的权重；预算按近期未证明次数倍增并封顶。实现仍有明确差异：只做证明，不随机切换证否；选题中“是否曾成功”使用全部已决定历史，而原论文强调近期窗口；它还有单步成功后降权和连续两次环境错误后排除的规则。不能仅凭注释写了 AlphaProof 就声称与原版逐项一致。它也不自行提供目标变体生成与原目标成功后的整组 TTRL 停止逻辑。

### 相关工作：题目课程与证明方案调度

| 工作与来源 | 相关机制 | 可借鉴范围与区别 |
| --- | --- | --- |
| [Evariste 优先标签采样器](https://github.com/facebookresearch/Evariste/blob/main/formal/evariste/backward/remote/prioritized_label_sampler.py) | 按证明结果、critic 误差等可配置信号采样定理，并混合久未采样的优先级 | 直接相关的题目课程代码；不是原版证明 / 证否、预算、TTRL 停止规则的同一实现 |
| [StepProver 论文 §2.2](https://arxiv.org/html/2410.15700v2#S2.SS2) | 专家迭代中的题目筛选、重试和预算提升 | 可研究从成功经验推进课程；不代表完整中央 matchmaker 已开放 |
| [STP 论文](https://arxiv.org/abs/2502.00212)与[官方实现](https://github.com/kfdong/STP) | 迭代提出猜想、尝试证明，再把正确证明加入下一轮训练 | 相关的自生成课程路线；候选命题生产与任务调度应分别实现 |
| [Kimina 7 月方法](https://huggingface.co/blog/AI-MO/kimina-prover) | 按引理利用情况选择上下文组合，卡住时递归生成子引理 | 调度对象主要是 scope 与引理组合；公开通用 RL 配方不等于此完整系统 |
| [Nexus 论文 §2](https://arxiv.org/html/2605.22763v2#S2) | 对草图种群评分、选择并演化 | 证明方案的推理调度；不等于训练题库的 matchmaker，也不包含代理参数更新 |

### 对两个复用目标的意义

若目标是 AlphaProof 机制复现，可从 nanoproof 的任务分配、结果回报和可恢复统计出发，逐项补齐证否、近期窗口、目标变体分组与停止规则；是否值得直接移植还取决于现有 actor 和 learner 接口。若目标是一般的小 prover，可以先采用简单的题目采样、成功证明回放和固定预算，测量自动课程的额外收益，再引入更复杂的 matchmaker。它能改变算力投放位置，却不能替代足够强的起始策略、可靠验证或有效的训练样本。

## 汇总比较：定位与测试

### 模型定位

这些项目覆盖不同层次。下表中的“接近”指机制或接口相似，不表示已经复现全部 AlphaProof。

| 系统 | 主要接口与对象 | 可直接借用的部分 | 与 AlphaProof 的关键关系 |
| --- | --- | --- | --- |
| StepProver | 状态、历史 → 下一条 tactic；独立 critic | 7B 逐步策略、1.8B 状态排序器 | 具备专家迭代与搜索评价；偏好分数不等于负证明深度，搜索不是原版 MCTS |
| REAL-Prover | 状态、检索前提 → tactic | 7B 逐步策略、前提检索、Lean 交互 | 可用于策略和环境层；原论文没有原版 value 与主 RL 调度闭环 |
| Reap | Lean 状态 ↔ 外部生成与评分服务 | Lean 搜索、状态处理、后端服务接口 | 有价值驱动搜索代码；不自动包含匹配模型的训练与 TTRL |
| DeepSeek-Prover-V2 | 命题 → 推理与完整证明 | 7B/671B 权重；教师分解、学生蒸馏路线 | 使用形式奖励 RL；整篇生成与 GRPO 不等于逐步策略–价值 MCTS |
| Kimina | 命题、可选引理或错误反馈 → 完整证明 | 小模型、公开 RL 与修复配方、验证服务 | 有目标相关 TTRL 方法报告；开源通用配方不是完整 7 月引理调度系统 |
| LEAP | 代理规划 → 草图、依赖图、已证子目标 | 证明义务管理方法与成功证明文件 | AND–OR 图与回溯搜索；所述流程不更新 specialist 参数 |
| Nexus | 候选草图种群 → 代理修改、工具证明 | 草图演化方法与结果材料 | 完整配置调用 AlphaProof 推理工具；代理演化本身不是 TTRL |
| HTPS / Evariste | 目标 → tactic 超边、critic、在线训练 | 超图搜索、critic 目标、异步训练机制 | AlphaProof 的相关前驱；价值目标和环境不同，归档代码不能直接开箱运行 |
| LeanDojo / ReProver | 可见库前提、状态 → tactic | 数据提取、检索、训练与评测流程 | 较小的监督检索证明器；适合数据基础设施，不是完整 RL 复现 |

### 发布时间与引用版本

时间用于定位技术演进，不当作当前排行榜。论文首次提交、权重发布和方法后续修订是不同日期；没有核实到首次发布日期的项目不填猜测值。

| 工作 | 时间依据 | 本文使用的材料 |
| --- | --- | --- |
| HTPS | 2022 年 5 月，[论文记录](https://arxiv.org/abs/2205.11491) | 原论文与归档 Evariste |
| LeanDojo / ReProver | 2023 年 6 月，[论文记录](https://arxiv.org/abs/2306.15626) | 历史论文成绩；当前 Lean 4 主分支单独说明 |
| StepProver | 论文 2024-10-21；官方发布记录 2024-10-22 | [论文 v2](https://arxiv.org/abs/2410.15700v2)，2025-10-21 修订，含 critic 实验 |
| DeepSeek-Prover-V2 | 2025 年 4 月，[论文记录](https://arxiv.org/abs/2504.21801) | v1、官方 V2 权重与结果 |
| REAL-Prover | 2025-05-27，[论文记录](https://arxiv.org/abs/2505.20613v1) | v1，不能默认替换为后续 v3 的实验口径 |
| Kimina | Preview：2025 年 4 月；72B / TTRL：7 月；RL 开源配方：8 月 | 三阶段材料分别引用，避免把方法和数据跨版本归并 |
| Nexus | 2026 年 5 月，[论文记录](https://arxiv.org/abs/2605.22763) | v2 与官方结果仓库 |
| LEAP | 2026 年 6 月，[论文记录](https://arxiv.org/abs/2606.03303) | v2 与官方结果目录 |
| Reap | 本文不主张一个未经核实的首次发布日 | 固定提交 `c6980f3` 的搜索与接口代码 |

### 测试与跑分
竞赛与教材基准结果

全部为作者报告，未由本文重新跑分。逐步搜索的多次 pass 和整篇生成的多次候选不同；空缺表示本节未引用可对应的结果，不表示成功率为零。

| 系统 / 配置 | miniF2F-test | ProofNet | 预算与来源 |
| --- | --- | --- | --- |
| StepProver，BF+CG | 65.9% | 27.0%，valid+test 合并 | `256×32×600`；[论文 §3.1](https://arxiv.org/html/2410.15700v2#S3.SS1) |
| StepProver，CG | 65.6% | 不与上一配置合并 | 同上；critic 单独引导配置 |
| REAL-Prover | 54.1% | test：23.7% | 64 次搜索、每步 64 候选；[论文 §4](https://arxiv.org/html/2505.20613v1#S4) |
| V2-7B，CoT | 75.6% / 82.0% | test：29.6% | miniF2F：32 / 8192 候选；ProofNet：1024；[论文 §3](https://arxiv.org/html/2504.21801v1#S3) |
| V2-671B，CoT | 82.4% / 88.9% | test：37.1% | 同上一行，各自模型独立成绩 |
| Kimina Preview | 约 80.7% | 本节未引用 | 8192 候选；[Preview §3](https://arxiv.org/html/2504.11354v1#S3) |
| Kimina 72B，冻结生成 | 84.0% / 87.7% | 本节未引用 | 32 / 1024 候选；[7 月说明](https://huggingface.co/blog/AI-MO/kimina-prover) |
| Kimina 72B，一轮错误修复 | 86.4% | 本节未引用 | pass@32 配合修复，成本包含额外生成 |
| Kimina 72B，TTRL Search | 92.2% | 本节未引用 | 作者估计 pass 预算上界约 42,000；含引理搜索和训练 |
| Kimina Distill-8B | 77.86%（模型卡）；78.3%（博客） | 本节未引用 | pass@32；[模型卡](https://huggingface.co/AI-MO/Kimina-Prover-Distill-8B)，差异未解释 |
| Kimina RL-1.7B | 76.63%（模型卡 / 配方）；76.23%（博客） | 本节未引用 | pass@32；[模型卡](https://huggingface.co/AI-MO/Kimina-Prover-RL-1.7B)，差异未解释 |
| HTPS，最终模型 | 41.0% | 本节未引用 | pass@64；valid 纳入在线训练，累计 valid 覆盖 58.6% 另计；[论文 §7.1.1](https://arxiv.org/pdf/2205.11491) |
| 原论文 ReProver | 26.5% | 历史 349 题：48/349，13.8% | pass@1 含最多 10 分钟搜索；[论文 §6](https://arxiv.org/html/2306.15626v2#S6) |

### 专项与研究级任务结果

| 系统 | 任务与结果 | 应如何理解 |
| --- | --- | --- |
| REAL-Prover | FATE-M：56.7%，141 道大学代数题 | 代数专项证据；同预算无检索配置为 44.7%，但两配置训练也不同 |
| V2-7B | PutnamBench：non-CoT 23/658；CoT 11/658，pass@1024 | 小模型的长推理在该配置上并未更好 |
| V2-671B | PutnamBench：49/658，CoT，pass@1024 | 实际可运行 649 题，表格保留历史分母；7B 与 671B 合并 62 题不是单模型成绩 |
| V2-671B | CombiBench：12/100，pass@16，with-solution；实际尝试 77 题 | 已给答案后的证明任务，不能说独立发现全部答案 |
| V2-7B / 671B | ProverBench：51.7% / 59.1%，CoT，pass@512 | AIME 子集分别 1/15、6/15；包含教材分析与代数不等于有领域专长 |
| LEAP | Lean-IMO-Bench：Basic 25/30、Advanced 17/30；Putnam 2025：12/12 | 每题两次完整代理 rollout，内部多次工具和模型调用；不是两次单候选生成 |
| Nexus | 已尝试 Erdős 问题 9/353；OEIS 猜想 44/492 | 经筛选、形式化的研究任务，不能与教材或 miniF2F 百分比直接排名 |

结果的来源与具体限制见各系统“测试表现”小节。总体上，StepProver、DeepSeek V2、Kimina 的竞赛证明证据较充分；REAL-Prover 有大学代数的直接训练与评测证据；ReProver 更适合研究库知识使用。LEAP、Nexus 展示代理级困难题与研究任务能力，但不能把这些系统的成绩归给一个可下载小模型。分析、代数、拓扑的相对强弱，仍需逐题、分领域、同预算实验。

## 数据公开情况
 数据公开情况：下载入口、对应模型与完整性

下表的“全部”必须限定到某个训练阶段和某个版本。预训练、继续预训练、SFT、专家迭代、蒸馏、RL 和 TTRL 可能使用不同材料；公开其中一个题库，无法证明发布了全部历史训练输入。本文将肯定证据写为“已公开”，没有覆盖清单的写为“不能确认全量”，不把未找到的材料断言成永远未公开。训练过程在线产生的数据，还需要区分可生成与原运行轨迹可下载。

### 公开训练数据

| 数据 / 直接入口 | 公开内容与范围 | 本文对应的使用者 | 是否等于该模型全部训练数据 |
| --- | --- | --- | --- |
| [Mathlib 4](https://github.com/leanprover-community/mathlib4) | 数学库定理与证明源码；state–tactic 和价值标签通常需另行提取 | AlphaProof、StepProver、REAL、HTPS 的相应历史库版本、ReProver；STP 也提取 Mathlib 4 | 否。公开源码不等于每个项目处理后的训练快照、全部证明轨迹或回放数据 |
| [Lean-Workbook / Plus](https://huggingface.co/datasets/internlm/Lean-Workbook) | 数据卡给出 57,231 / 82,893 个问题的来源划分，含陈述、答案和可获得的证明；官方发布记录链接约 1.4 万份搜索证明 | StepProver；REAL 使用检索增强后的 Workbook；V2 的非 CoT 数据来源涉及 Workbook；STP 使用 Workbook | 否。题目数、证明数和逐步样本行数不同；不能把当前 viewer 某一配置的行数当作完整专家迭代集 |
| [Lean-Github](https://huggingface.co/datasets/internlm/Lean-Github) | GitHub Lean 项目提取数据，含源仓库、commit、文件、状态与 tactic 字段 | StepProver | 否。不是其全部专家迭代证明或 critic 偏好对；也不等于其他项目自行收集的同名 GitHub 语料 |
| [REAL state_tactic_pairs](https://huggingface.co/datasets/FrenzyMath/state_tactic_pairs) | [官方仓库](https://github.com/frenzymath/REAL-Prover#data)说明约 5 万对 | REAL-Prover | 不能认定全量。论文最终口径为 210,420 对，未给出公开 5 万对覆盖所有训练来源与处理阶段的证据 |
| [LeanDojo Benchmark 4](https://zenodo.org/doi/10.5281/zenodo.8040109)；[下载脚本](https://github.com/lean-dojo/ReProver/blob/main/scripts/download_data.py) | 版本化 Mathlib 数据、random 与 novel-premises 划分；按 README trace 源仓库，再产生检索增强输入 | 当前 Lean 4 ReProver | 对当前公开 ReProver 配方，其数据入口和构建流程公开；不能扩张为包含 ByT5 基础预训练的全部历史数据。原论文 Lean 3 快照与 Benchmark 4 要分开 |
| [原版 LeanDojo Benchmark](https://doi.org/10.5281/zenodo.8016385) | 历史 Lean 3 Mathlib 快照，98,734 个定理与证明及前提标注 | 原论文 ReProver | 是原论文公开证明器的数据来源；不能用它代替当前 Lean 4 数据，也不包含基础模型预训练全集 |
| [NuminaMath-LEAN](https://huggingface.co/datasets/AI-MO/NuminaMath-LEAN) | 数据卡约 100K 竞赛题；含自然语言、formal_statement、formal_proof 与验证状态等，部分证明可为空或未验证 | 数据卡明确对应 Kimina 72B；Promptset 由它筛出 | 否。不能代表全部 CPT、CoT 冷启动、蒸馏、RL 及目标 TTRL 数据；也不能把整个集合称为每题均有人类完整证明 |
| [Kimina-Prover-Promptset](https://huggingface.co/datasets/AI-MO/Kimina-Prover-Promptset) | 由 NuminaMath-LEAN 去易题、用 Gemini 生成变体、重复难题加权所得的 RL 输入集 | 公开 Kimina RL-1.7B 配方；具体脚本指定该入口 | 是公开配方指定的离线 RL 题目输入，但不是模型全部训练数据；在线生成的正确、错误及修复 rollout 不等于此表本身 |
| [NuminaMath 1.5](https://huggingface.co/datasets/AI-MO/NuminaMath-1.5) | 非形式题目与解答，上游来源，**不是 Lean 证明轨迹集** | Kimina Preview 的自动形式化来源；REAL 论文也使用 NuminaMath 来源题 | 否。自动形式化、筛选、人工修订、验证与轨迹提取后的材料需要另外确认 |
| [STP_Lean_0320](https://huggingface.co/datasets/kfdong/STP_Lean_0320)；[官方说明](https://github.com/kfdong/STP#2-model-and-dataset) | Mathlib 提取样本、Workbook 正确证明、自博弈猜想正确证明 | STP；V2 论文明确使用 STP 的公开合成数据作为 SFT 来源 | STP README 明确说明发布模型在该集上训练一轮，可对应最终微调阶段；不是继承基座和历轮自博弈的全部历史数据，更不是 V2 全量训练集 |
| [Goedel 的 Lean-workbook-proofs](https://huggingface.co/datasets/Goedel-LM/Lean-workbook-proofs)；[官方说明](https://github.com/Goedel-LM/Goedel-Prover#3-model-and-dataset-downloads) | 官方报告约 29.7K 份 Workbook 成功证明，提供 problem_id 与完整证明 | Goedel-Prover 生成；V2 §2.3 引用 Goedel 工作作为公开数据来源之一，但没有给出与当前下载快照逐条对应的清单 | 否。公开 Workbook 解答不等于 Goedel 的全部自动形式化与历轮训练数据，也不能指定为 V2 全量集 |

REAL 论文的 210,420 对可以进一步对应为：教材代数专家迭代 25,818 对、NuminaMath 专家迭代 30,589 对、196 道人工代数题提取 18,669 对、Mathlib 92,152 对、检索增强 Workbook 43,192 对。这五项相加符合论文总数；公开 5 万对与它们的逐项覆盖关系没有在 README 中说明，不能擅自猜测是哪一项或简单把差额全叫作“私有数据”。[REAL 论文 §3.1](https://arxiv.org/html/2505.20613v1#S3.SS1)

另有两个下载边界需要说明：LeanDojo 的 DOI 由论文或 GitHub 下载入口确认，但本次两项 DOI 的直连均返回 403，未实际下载归档；保留 DOI 与脚本入口，不能据此认定数据已撤回。STP 的 `0320` 是当前官方 README 指向的公开快照，V2 论文引用的是 STP 数据来源，本文不把两者未经核实地等同为完全相同的训练版本。

### 公开训练数据完整性

| 系统 / 版本 | 已确认公开的资产 | 全部训练数据结论与待补缺口 |
| --- | --- | --- |
| AlphaProof 原版 | 论文方法、补充材料、部分题目及证明结果 | 没有覆盖约 80M 自动形式化题库、主 RL replay 和目标变体全部轨迹的公开清单；不能作为全量数据复现起点 |
| StepProver | Workbook、Github 数据入口、部分搜索证明、prover / critic 权重 | **部分公开，不能确认全量。** 论文附录的最终 454K critic 偏好对及历轮专家迭代样本没有在已核实入口形成完整对应清单 |
| REAL-Prover v1 | 约 5 万逐步样本、库及 Workbook 来源、FATE-M、代码与权重 | **不能确认最终 210,420 对全量公开。** 需检索增强后的训练快照和来源覆盖说明 |
| Reap | 搜索客户端、服务接口、轨迹输出机制 | **不是独立发布的已训练模型。** 训练数据完整性随所接后端而定，不能给框架统一打“数据全公开”标签 |
| DeepSeek-Prover-V2 | 7B / 671B 权重、ProverBench、miniF2F 结果、论文指出的公开上游来源 | **不能确认全量。** 自建专家迭代、V3 分解冷启动、671B RL、7B 蒸馏与后续 RL 没有完整下载清单；ProverBench 是评测集 |
| Kimina Preview / 72B / Distill | 权重、NuminaMath-LEAN、方法描述 | **部分公开，不能确认全量。** Kimi 思考混合数据、CoT 冷启动、CPT、全部蒸馏与 TTRL 轨迹需分别核实 |
| Kimina RL 小模型公开配方 | Promptset、蒸馏起点权重、GRPO / 修复训练代码与验证环境 | **该配方离线题目与生成流程可获取。** 不意味着重现同一随机运行轨迹，也不意味着原 72B 或继承蒸馏基座的全部训练集公开 |
| HTPS / Evariste | 归档训练与搜索代码、论文、对应环境来源 | **不能确认全量且不能开箱运行。** 内部基础设施已移除，旧 Lean 环境与历史数据需迁移 |
| ReProver | Benchmark 下载、trace、检索器和生成器训练与评测代码、权重 | **公开配方的数据链较完整。** 可按指定划分和版本构建；基础预训练数据、历史论文与当前 Lean 4 版本仍需分开 |
| LEAP / Nexus | 公开成功证明、题目与结果入口 | **公开的是结果材料。** 未形成完整代理程序、后端基础模型全部训练数据及全程搜索日志的发布证据 |

### 公开评测集

这些入口便于复测或分析，不能自动当作训练许可或训练来源。若用于训练，必须另划独立测试集并记录污染风险。

| 数据 / 入口 | 内容、侧重 | 本文使用关系 |
| --- | --- | --- |
| [miniF2F 原项目](https://github.com/openai/miniF2F)；[Lean 4 移植](https://github.com/yangky11/miniF2F-lean4) | 初等数学与 AMC / AIME / IMO / MATH 题；valid/test 常各 244 | 多数模型评测；StepProver critic、HTPS 在线训练涉及 valid，不能再称该 split 完全未见 |
| [ProofNet](https://github.com/yangky11/ProofNet)；[V1.5 仓库中的 Lean 4 版本入口](https://github.com/deepseek-ai/DeepSeek-Prover-V1.5) | 大学纯数学教材：分析、代数、拓扑等 | Step、REAL、V2、ReProver；版本与 split 差异须保持，V2 使用的 STP 数据含 valid 变体 |
| [PutnamBench](https://github.com/trishullab/PutnamBench) | 跨年份大学高难竞赛题，持续更新 | V2 的历史 658 分母；不等于某一年 12 题 |
| [FATE-M JSONL](https://github.com/frenzymath/REAL-Prover/blob/main/Realprover/data/fate_m.jsonl) | 141 道大学代数题 | REAL 原论文专项评测，不是公开最终训练集 |
| [ProverBench](https://huggingface.co/datasets/deepseek-ai/DeepSeek-ProverBench) | 325 题，含 15 道近期 AIME 与 310 道教材 / 教程题 | V2 发布的评测集，不能称为其部分 RL 训练轨迹 |
| [CombiBench](https://github.com/MoonshotAI/CombiBench) | 100 道组合题；答案已给与答案待发现的设置须区分 | V2、Kimina Preview 的组合评测 |
| [Lean-IMO-Bench 题目 CSV](https://github.com/google-deepmind/superhuman/blob/main/imobench/lean_proof_bench.csv)；[LEAP 证明](https://github.com/google-deepmind/superhuman/tree/main/leap/solutions/LEAN-IMO-Bench) | Basic / Advanced 各 30 题 | LEAP 评测与结果；不是其后端模型训练全集 |
| [V2 miniF2F 解答 ZIP](https://github.com/deepseek-ai/DeepSeek-Prover-V2/blob/main/minif2f-solutions.zip) | 成功形式证明文件 | 结果分析与验证，不是冷启动或 RL 全集 |
| [LEAP 官方目录](https://github.com/google-deepmind/superhuman/tree/main/leap) | Putnam 2025、竞赛与研究案例的结果 | 可检查完成的证明，不能据此运行原代理 |
| [Nexus 官方结果](https://github.com/google-deepmind/alphaproof-nexus-results) | 成功 Lean 证明、部分非形式证明；README 链接全部尝试 OEIS 题及未解 Erdős 形式化 | 研究任务分析；成功结果不能替代失败日志、选择过程和完整训练数据 |

复用训练数据时，应锁定模型、Lean、Mathlib 与数据 commit，并实际检查 `sorry`、未验证证明、循环或目标自身作为前提的情况。数据卡提供字段和来源证据，最终可训练性仍由选定版本的编译、轨迹提取和划分检查决定。

## 阅读公开结果时需要避免的误读

critic、value network 与 value head 的关系

critic 是评价功能的称呼，value network 是预测价值函数的网络，value head 是网络输出结构。StepProver 的 critic 能承担搜索状态评价，AlphaProof 的 value 则有特定的训练目标和回传语义。比较时应核对输入、标签、输出尺度和搜索公式，不能只凭名字判断可替换性。

有搜索组件不等于完整 AlphaProof

拥有 Lean 环境、MCTS、policy 和 value 接口，只能确认已有这些组件。完整闭环还需核实价值如何训练、成功轨迹如何进入 replay、matchmaker 如何调度、specialist / TTRL 如何启动，以及新参数怎样返回后续搜索。公开工程实现与论文级结果复现也要分开。

 数据公开必须限定版本和层级

原始题目、state–tactic 对、成功证明、搜索轨迹、训练输入、完整生成程序是不同资产。上面的索引逐项给出入口与边界；“来源可获得”“配方可重跑”和“原模型全部训练数据可下载”不能合并成一个结论。

## 是否值得复用：面向两个目标的判断

### 两种目标下的复用优先级

下面是根据前文公开接口与数据证据提出的选型建议，不是新的性能实测。对“复现 AlphaProof”，重点是还原机制；对“做出效果好的小 prover”，重点是固定成本下的证明覆盖与可训练性。

| 目标 | 值得优先复用 | 需要自行补齐或验证 | 不宜直接许诺的结果 |
| --- | --- | --- | --- |
| AlphaProof 逐步搜索与价值闭环 | StepProver / REAL 的 tactic 策略；Reap 搜索；HTPS 机制；nanoproof matchmaker | 负剩余最长分支深度标签、AND–OR 搜索一致性、成功回放、模型更新发布与目标 TTRL | 拼接这些部件后自动得到原版能力或计算效率 |
| 独立整篇生成的小 prover | V2-7B；Kimina Distill / RL 小模型；公开 Kimina 训练与修复配方 | 同版本 Lean 验证、固定预算基线、领域外测试、数据去重与污染检查 | 将 671B / 72B / 代理系统成绩归给小模型 |
| 库依赖较强的大学题，尤其代数 | REAL 与 LeanSearch-PS；ReProver 的数据和检索流程 | 与无检索配置比较总 token、验证时间及闭合率；补充目标学科样本 | 由 FATE-M 总分推断分析、拓扑或研究代数均强 |
| 小模型学习规划与子目标 | V2 教师分解蒸馏路线；STP 正确证明数据；LEAP / Nexus 的证明义务方法 | 自建可验证的规划轨迹，训练后在无教师的新题上测独立能力 | 公开结果文件已等于完整蒸馏课程或原代理复现 |

若近期产出是一个可用的小 prover，公开权重加可靠验证通常能更快形成基线；若研究贡献在 AlphaProof 的搜索、value 或 TTRL，则应保留逐步状态和可解释的价值目标，把整篇模型作为教师或局部工具。两条路线可以共享数据与 Lean 环境，但验收标准应分别制定。

### 接口探针与后续实验边界

前面列出的工作并不要求选出一个“全方面最佳”。研究目标不同，需要替换的部件就不同。以下是基于已核查接口提出的实验建议，不是已经做过的模型排名。

**研究逐步搜索与价值学习时，优先保留逐步接口。** StepProver 提供现成的 tactic 生成与独立 critic；REAL-Prover 提供 tactic 与检索；Reap 提供可接后端的 Lean 搜索实现。应分别验证策略输入、评分语义、状态恢复和证明回放，而不是只看三个组件能否连通。要学习更接近 AlphaProof 的训练---搜索闭环，可以结合 HTPS 的机制研究，以及独立的 nanoproof 源码讲义。

**研究独立小模型的规划能力时，区分在线合作与离线蒸馏。** DeepSeek-Prover-V2 提供“先借助教师生产可验证分解数据，再训练学生”的路线；Kimina 的蒸馏模型与公开 RL 配方提供另一个起点。LEAP、Nexus 展示了运行时组织规划、草图与子目标的方法，但它们的高分不能直接当作学生蒸馏后能力的证据。学生最终必须在没有教师参与的新题上评测。

**研究测试时适应时，不能只问有没有梯度更新。** 至少要定义：针对哪个目标或哪批目标适应；适应数据从哪里来；奖励与更新规则是什么；训练前后搜索预算是否相同；是否允许跨测试题共享新参数；最后评的是累计发现过的证明，还是最终检查点重新搜索的能力。Kimina 与 HTPS 提供不同的相关机制，但方法名字的相似不应代替这些定义。

**用一个接口契约把候选约束住。** 输入需要当前 Lean 状态、原定理、历史证明还是检索引理？输出是一条 tactic、一段局部证明还是整篇证明？模型能否提供与采样一致的动作对数概率？最大输入与输出长度怎样分配？模型、tokenizer、Lean 和 Mathlib 版本是否匹配？权重能否本地训练？训练脚本是否实际使用预期奖励？这些回答决定接入成本，不能靠模型参数量猜测。

**先做小型但有代表性的探针，再决定训练。** 探针应包含普通闭合、需要库引理、较长局部假设、多子目标、简单原创构造，以及最终目标所在领域的少量真实题。记录合法输出率、真正推进状态的比例、固定预算闭合率、token 和验证耗时。按题目家族划分训练与测试，避免把几乎同构的题拆到两侧。数值大小、难度标签或总榜单分数，都不能替代这一轮接口与领域检查。

**没有价值头，并不等于无法自举；有价值头，也不等于已经具备 AlphaProof。** 整篇生成加形式奖励可以不使用搜索 critic；逐步搜索若要采用原版数值逻辑，则必须训练并校准匹配的价值信号。相似地，没有百万变体不等于不能做 TTRL，但若目标附近完全没有可验证成功、奖励也没有其他有效学习信号，仅添加一个训练循环不会自动教会模型新的证明技巧。

最后可以把这些路线归结为三个互补问题：**策略从哪里获得基本 Lean 能力，搜索怎样更有效地发现证明，新的经验怎样改变以后解决问题的能力。** 从现成权重出发可以省去重新学习 Lean 的一部分成本；从现成搜索器出发可以省去一部分工程；从已验证的训练流程出发可以减少算法试错。但每省去一层工作，仍要检查它与其余层的接口和证据是否匹配，不能把“可下载、能调用、含有相关名字”合并成“完整复现”。
