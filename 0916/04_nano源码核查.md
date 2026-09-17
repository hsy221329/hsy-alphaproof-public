# nano源码核查

本篇的对象是 **Kripner/nanoproof**，固定在提交 `4f410c6e7a820557501848f954402514201322a5`，不是所有同名项目，也不是任何后来更新。结论来自 README、实际控制流、训练样本构造与模型配置的静态核查；本次没有重新训练该模型，也没有运行完整 Lean 基准。[固定版本](https://github.com/Kripner/nanoproof/tree/4f410c6e7a820557501848f954402514201322a5)

最重要的原则是：**实际执行代码优先于旧 TODO；可定位的实现问题与效果尚未验证的设计选择分开。**

## 训练与TTRL

仓库明确包含预训练、Lean 代码中间训练、监督微调，以及多 GPU 的 RL 循环。其 `rl.py` 调用 `prover.collect(...)` 收集 MCTS 经验，之后切换训练模式，并存在实际 `optimizer.step()`。因此，“它没有任何强化学习”与当前源码不符。[README](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/README.md) · [训练脚本](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/rl.py)

这个循环按收集与训练阶段交替运行：actor 线程进行搜索，成功树被转成训练样本，模型更新后再收集下一阶段经验。它能回答“新的成功尝试如何让模型变强”，但这本身是**一般训练闭环**，还不是自动完成“测试目标相关适应”的证明。[N2](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/README.md)[N3](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/rl.py)（RL Loop；经验处理见 N5）

**本次能确认的边界：**所核查训练题池加载 Lean-Workbook、NuminaMath-LEAN、DeepSeek-Prover 等数据；miniF2F 与 ProofNet 被列作 benchmark。没有在这条公开主流程中核实到围绕每个测试目标生成相关变体、初始化专项模型、在该局部题池更新并重新尝试原目标的完整 TTRL 调度路径。因此应说：**有 RL；未核实到 AlphaProof 式目标相关 TTRL 的完整公开实现。**

即便不采用大量变体，也可能设计测试时训练。例如在测试题组上搜索，拿解出的题训练，再重试未解题，可以构成跨测试题的适应；用相关引理训练再解原题，也可以构成目标适应。但“原则上可以这么改”不能替代“仓库已经这么执行”。评测函数里出现搜索、日志或分批处理，也不能自动当作参数更新。

若只把一题反复搜索，且始终没有任何成功轨迹，就无法从这个正样本训练路径获得新证明样本。要启动它，需要更容易的问题、历史经验、可验证子目标或其他已定义的反馈机制；不是把运行时间拉长就必然产生梯度信息。这是对训练闭环的机制分析，不是该仓库已解决全部冷启动问题的结论。

## 价值到底有没有问题

当前 `run_mcts` 从模型得到 tactics、动作 log probabilities 和 value，然后把预测的正深度取负后用于搜索。训练数据构造器则把每个正样本拆成两种输入：状态后接 `<|tactic|>` 来预测 tactic；状态后接 `<|value|>` 来预测深度对应的 token。这是一种**共享语言模型、不同提示任务的价值预测实现**，不等同于原版独立分类头，但也不是没有学习的固定常数。[搜索代码](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py) · [数据构造](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/data/sft/leantree_dataloader.py)

这里的“head”容易产生误会。仓库 README 使用 learned policy and value head 的描述；精确到实现，价值通过专门的 token 任务训练和读取。讨论架构差别时，应说清实际张量/输出接口，不能仅靠 README 中一个名词断定两套系统完全同构。

`compute_value_target` 的当前规则是：终止状态为 0；OR 节点在已解决子分支中取最佳负步数，再减 1；AND 节点取所有子目标负步数的最小值。[N5](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/experience_collection.py)（compute_value_target）

概念上可写成 $G_{\mathrm{terminal}}=0$、$G_{\mathrm{OR}}=-1+\max G_{\mathrm{solved\ child}}$、$G_{\mathrm{AND}}=\min_iG_i$。这与“选择较短成功证明、由最长必需分支决定 AND 难度”的方向一致。**不能因为 AND 取最小值、先处理较难子目标，或预测深度后取负，就判定数学符号错了。** 详见 06。

当然，它仍有值得研究的限制：成功树提供的只是找到的证明深度，不保证全局最短；样本集中在可解状态，不直接监督所有死路；深度也不是固定预算下的成功概率；有限分桶会压缩较长证明的差异。但这些首先是监督目标、选择偏差与校准问题，不是从一个 `max` 或 `min` 就能证明的代码 bug。

原版 AlphaProof 也学习成功证明的回报。因此，“只依据找到的成功证明，未求出全局最优深度”不能单独构成“nano 不像原版”的决定性批评。更有区分力的检查是：标签是否与搜索中的 reward 约定一致、AND 的伪动作是否额外扣步、价值输入是否匹配训练输入，以及价值引导相对于不使用价值是否真正增加闭合率。

**旧 TODO 中存在不能照抄的结论。** 它写到动作先验均为 1，但当前扩展代码已使用 `exp(logprob / temperature)`；它还声称当 value 为较大的负数时，$\gamma^{-1-V}$ 会爆炸。对于 $0<\gamma<1$ 且 $V\le -1$，指数 $-1-V\ge0$，所以这个量不超过 1：例如 $\gamma=0.98,V=-10$ 时是 $0.98^9$，约为 0.834，而不是爆炸。[旧 TODO](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/TODO.md) · [当前实现](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py)

TODO 也包含“所有失败都丢弃”的说法；当前经验处理和数据构造已支持失败 tactic 的负样本路径。不过，**支持负样本不等于所有默认运行都启用它**，更不等于把所有最终未解题都当成负证明监督。有效但暂未走通的动作，与 Lean 执行直接报错的动作不是一类标签。[N5](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/experience_collection.py) [N6](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/data/sft/leantree_dataloader.py)

## 渐进采样的具体错误

这部分发现了比“我觉得采样有问题”更强的源码依据，需要分开检查触发与扩展。

`progressive_sample` 的判定是：OR 节点满足 `evaluations <= ps_c * visit_count ** ps_alpha` 时，再进行一轮采样。`evaluations` 是这个节点已经被扩展/查询的次数，不是子节点个数。这个触发形式与原版渐进式增加动作采样的思路相符。[N4](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py)（progressive_sample）

问题出在 `expand_node`：它每次执行都会先运行 `node.children = {}`，然后才把本轮采到的动作建立为子节点。`run_mcts` 又允许对已经扩展的节点调用这个函数。因此，**只要触发再次采样，原来的子节点字典及其下面的搜索树会被替换，而非保留后补充新候选。**[重复进入扩展的控制流](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py#L516-L560) · [清空子节点的位置](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py#L658-L706)

这是一个可以从控制流直接推出的结论，不依赖训练效果猜测。假设节点原来有动作 A、B；B 已被访问 20 次，其下有一段接近成功的证明。下一轮只采到 C。正确的累计扩展至少应保留 A、B 并加入 C；当前代码会只留下 C。父节点的既有访问与价值统计并未同时归零，于是还可能出现“父节点带着旧搜索统计，下面却换成新子树”的不一致。

下面是这一机制的最小示意，不是完整仓库复现，也没有调用 Lean：

```python
from dataclasses import dataclass, field

@dataclass
class TinyNode:
    children: dict[str, dict[str, int]] = field(default_factory=dict)

# 仅重现“无条件重建 children”这一控制流。
def replacing_expand(node: TinyNode, actions: list[str]) -> None:
    node.children = {}
    for action in actions:
        node.children[action] = {"visits": 0}

node = TinyNode({"A": {"visits": 3}, "B": {"visits": 20}})
replacing_expand(node, ["C"])
assert set(node.children) == {"C"}  # A、B 和已有访问数据均消失。
```

**它与重复动作处理还存在关联。** 当前代码先用字典整理本轮 policy；又刚刚把 `children` 清空，所以“若动作已在旧 children 中则累计先验”这段分支无法完成跨采样轮次的复用。不能只修触发系数、增加探索温度，便以为保留旧搜索进展的问题解决了。

一个最低限度的修复方向是只在 `children is None` 时初始化字典；新增动作才新建子节点，旧动作保留其子树、访问计数和价值统计。重复动作的先验如何累计与归一化，还需与原始采样分布定义一致。上述是修复建议，本次没有向上游提交修改。

**这能证明多少性能损失？** 不能从静态审计直接回答。若搜索预算小到从未触发再次采样，相关路径就不发生；若经常再次采样，影响才可能明显。应统计每次搜索的重复扩展数、被清除的子树大小，并对同模型、同预算、同随机种子比较修复前后的结果。不能未经实验声称“修复后必涨多少分”。

## 参数与上下文口径

当前搜索默认配置包括：`ps_c=0.1`、`ps_alpha=0.6`、`value_discount=0.98`、`prior_temperature=200`、`c_and=64`，以及 `pb_c_base=200`、`pb_c_init=0.001`。这些是**这个固定版本的源码默认值**，不是 AlphaProof 生产超参数，也不代表所有已发布 nano 结果都按默认值运行。[N4](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py)（SearchConfig.defaults）

例如节点已扩展一次，下一次触发要求 $1\le0.1N^{0.6}$，所以至少约 47 次访问才满足条件。调大 `ps_c` 可能让清空子树的问题更频繁，而不是简单增加健康探索。这个例子说明，超参数调节必须以实现正确为前提。

`prior_temperature=200` 处理的是**完整动作的 log probability 在树中的先验权重**；不能把它理解为语言模型每个 token 都以温度 200 生成。两种温度的作用位置和数值尺度不同，详见 06。

你提到“1.03B 模型最多 768 tokens”，当前源码不足以支持把这个数字当作不可突破的架构硬上限。`NetworkConfig.sequence_len` 是配置项；模型还设置 `rotary_seq_len = config.sequence_len * 10`。数据加载器又按 `GLOBAL_CONFIG.max_seq_len` 筛掉过长样本。[模型代码](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/model.py) · [数据构造](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/data/sft/leantree_dataloader.py)

因此至少要分开：某个训练配方使用的序列长度、某个权重真正学过的长度分布、推理实现允许的长度、对长输入仍有效的能力范围。**位置编码缓存允许更长输入，不证明模型在长状态上训练充分；训练时常用短序列，也不自动证明推理硬上限就是该数字。**

本次没有取得与你所述 1.03B 检查点严格对应的完整模型卡、tokenizer、训练参数与 768 限制证据，因此不把这组数字判为已核实。也不反过来宣称这个模型拥有可靠的长上下文能力。

tokenizer，即分词器，同样需要与具体权重绑定。当前项目说明使用 GPT-2 的 BPE（byte-pair encoding，字节对编码）分词器，加 Lean/数学专用 token。它是一个明确的工程选择，但是否显著损害效率，要测量同一批 Lean 状态和 tactic 的 token 长度与截断率。不能仅因它不是另一个大模型的 tokenizer，就断言它无法做 Lean。

## 成绩与可复用资产

当前固定版本 README 写的是 **MiniF2F-Valid 上 512 simulations 达到 52.7%**。这与“miniF2F-test、pass@256、51.3%”不是同一陈述。可能存在历史版本或不同配方，但本次未把你所述数字定位到精确的检查点与评测命令，所以不能用于严格横向排名。[N2](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/README.md)

必须分别记录 split、模型检查点、每步候选数、搜索模拟次数、允许的自动 tactic、重试数、最终验证规则和总计算。当前源码还有禁用部分 solver tactic 与评测时注入 `grind` 的可选路径；这种设置可能改变模型与符号工具各自承担的任务。**不能在不知道运行配置时，把某一成绩全归给语言模型本身。**[可选 solver 路径](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/nanoproof/search.py#L658-L706)

公开程度也应降到具体资产。README 链接了数据获取脚本、LeanTree 监督数据来源、RL 题库和训练流程；其中 Nemotron 数据需要接受访问条款，Lean GitHub 原始语料没有直接发布到 Hugging Face，需要按来源仓库自行构建。因此“训练管线公开、许多来源可获取”有依据，“全部成品训练数据完全无门槛下载”则不准确。[N2](https://github.com/Kripner/nanoproof/blob/4f410c6e7a820557501848f954402514201322a5/README.md)（Datasets）

值得复用的不只是 MCTS 名称：包括 LeanTree 的状态交互、成功树到 transition 的提取、并行收集与训练阶段切换、回放缓冲区、题目历史统计、日志与验证路径。这些有助于把一个想法变成可观测的实验。复用前必须修正或隔离上述搜索扩展问题，并按目标领域检查数据和长度限制。

**最终判断：**它是一个有参考价值、包含真实学习循环的开放工程；不能因其模型成绩不处于前沿就认为工程无价值，也不能因目录里有 policy、value、MCTS 和 RL 就称其完成原版 TTRL。将它用作机制实验平台与将它选为最强能力基座，是两个不同决策。
