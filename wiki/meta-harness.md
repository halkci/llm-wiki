---
title: Meta-Harness:面向模型 harness 的端到端优化
aliases: [Meta-Harness, meta-harness, 元 harness]
tags: [llm, harness-engineering, agent, optimization, paper]
sources:
  - sources/meta-harness-2603.28052v1.pdf
  - https://yoonholee.com/meta-harness/
  - https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact
related: [[harness-engineering]]
created: 2026-04-05
updated: 2026-04-05
---

# Meta-Harness:面向模型 harness 的端到端优化

## 摘要

**Meta-Harness** 是 Stanford / MIT / KRAFTON 团队(Yoonho Lee, Roshen Nair, Qizheng Zhang, Kangwook Lee, Omar Khattab, Chelsea Finn)在 2026 年 3 月发布的外层循环系统(arXiv:2603.28052v1)[^1]。它把 LLM 应用中的 **harness** —— 即决定"给模型看什么、何时取、怎么展示"的周边代码 —— 本身作为搜索目标,用一个 coding-agent 提议者(proposer)在**整棵历史文件系统**上反复检索、诊断、改写 harness 代码,实现自动化 harness 工程[^1]。

核心主张:**"更有价值的不是搜索代码,而是带选择性访问先前诊断经验的搜索。"** 提议者不再被限制在标量奖励或压缩摘要上,而是可以用 `grep`/`cat` 直接读原始代码、执行轨迹、失败记录,从而形成因果假设并改写 harness[^1]。

## 核心内容

### 1. 问题设定:harness 优化

给定固定语言模型 $M$ 与任务分布 $\mathcal{X}$,一个 harness $H$ 是包装 $M$ 的**有状态程序**,决定每步看到什么。目标是:

$$H^* = \arg\max_H \; \mathbb{E}_{x \sim \mathcal{X},\, \tau \sim p_M(H,x)} \; r(\tau, x)$$

多目标时(例如 accuracy 与 context cost)按 Pareto 支配评价[^1]。

### 2. 搜索循环(Algorithm 1)

1. 初始化 harness 种群 $\mathcal{H}$ 与空文件系统 $\mathcal{D}$。
2. 评估所有初始 harness,把 (代码, 分数, 执行轨迹) 全部存入 $\mathcal{D}$。
3. 每次迭代:proposer 查询 $\mathcal{D}$ → 提出 $k$ 个新 harness → 接口验证通过就评估并把完整日志写回 $\mathcal{D}$。
4. 返回 Pareto 前沿。

无父代选择规则、无固定搜索启发式 —— 诊断与编辑决策完全下放给 coding agent[^1]。

### 3. 相对于先前文本优化器的差异

| 方法 | 历史访问 | 反馈内容 | MTok/iter |
|---|---|---|---|
| OPRO | 窗口 | (解, 分数) 对 | 0.002 |
| TextGrad | 最近一条 | 对当前 artifact 的文字反馈 | 0.015 |
| AlphaEvolve | 窗口 | 程序库 + eval 分数 | 0.022 |
| GEPA | 摘要 | rollout 轨迹的反射反馈 | 0.008 |
| Feedback Descent | 摘要 | 对比 + 文字反馈 | 0.012 |
| TTT-Discover | 窗口 | 前一解片段 | 0.026 |
| **Meta-Harness** | **全量** | **所有日志与分数** | **10.0** |

Meta-Harness 的每步上下文规模比已有文本优化器大**三个数量级**,单次评估可产生多达 $10^7$ tokens 的诊断信息[^1]。

### 4. 实验结果

**在线文本分类**(LawBench / Symptom2Disease / USPTO-50k,GPT-OSS-120B 分类器):
- Meta-Harness 48.6% avg acc,ACE 40.9%,MCE 40.0%(分别高 7.7 / 8.6 分)
- 使用 11.4K context,远低于 ACE 的 50.8K / MCE 的 28.5K
- 仅用 **4 次评估**就追上 OpenEvolve/TTT-Discover 跑 60 次的最终分数,快 10×[^1]
- OOD 9 数据集平均 73.1%,7/9 领先 ACE[^1]

**检索增强数学推理**(200 IMO 级题,5 个 held-out 模型):
- 单一搜索出的 retrieval harness 相对无检索平均 **+4.7 分**,跨模型一致[^1]

**TerminalBench-2 agentic coding**(89 任务):
- Opus 4.6 上 **76.4%**,在 Opus 4.6 agent 排行榜上仅次于无法复现的 ForgeCode,超过 Terminus-KIRA (74.7%)
- Haiku 4.5 上 **37.6%**,超过 Goose (35.5%) 拿到第一[^1]

### 5. 发现的 harness 设计(Appendix B)

- **文本分类 — Label-Primed Query Harness**:单次调用,prompt 由 label primer(列出所有有效标签) + coverage block(每 label 一个 query-相关示例) + 查询锚定对比对 组成;TF-IDF + query-anchored pairing 选对比样本[^1]。
- **文本分类 — Draft Verification Harness**:两步调用,先根据 5-NN 示例产出草稿标签,再用该草稿条件化地检索 confirmers / challengers 做验证[^1]。
- **Math Retrieval Harness**:4 路 BM25 词法路由器(组合 / 几何 / 数论 / 默认),路由由关键词 + 正则门控,每路不同的 rerank 策略[^1]。
- **TerminalBench-2 Harness**:在 Terminus-KIRA 基础上加上 **Environment Bootstrap** —— 在首次 LLM 调用前运行一条复合 shell 命令,抓取 pwd / `/app` 列表 / 可用语言版本 / 包管理器 / 内存快照,并注入初始 prompt。约 80 行代码,节省 2–4 轮探索[^1]。

### 6. 定性行为:因果推理而非随机变异(Appendix A.2)

TerminalBench-2 的搜索日志显示 proposer **识别混淆因子**并故意回避。前 2 次迭代把结构性修复与 prompt 清理打包在一起,双双回归;第 3 次迭代 proposer 显式写下: *"Root cause of regressions: Prompt template changes (cleanup directives) caused the agent to delete necessary state before task completion. The structural bugfixes were confounded with harmful prompt changes."* 于是它隔离两项结构修复,放弃 prompt 重写,继续尝试后六次全部回归后改为**纯加性**的 `evo_env_bootstrap` 策略(仅在循环开始前追加环境快照,不动原逻辑),成为本次运行的最佳候选[^1]。作者将这种**基于完整历史的因果假设**视为压缩反馈优化器无法支持的关键能力。

### 7. 文件访问统计(Appendix A.1)

在 TerminalBench-2 的 10 次迭代中,proposer 每次迭代中位读取 **82 个文件**(范围 69–99):harness 源码 41% / 执行轨迹 40% / 分数摘要 6% / 其他 13%。访问模式明显是**非马尔可夫**的 —— 它不只看上一步,而是遍历整个历史[^1]。

### 8. 实用实现建议(Appendix D)

- **写好 skill 文本** —— 它对搜索质量的杠杆比迭代数/种群大小更大;应约束输出与安全,而不是规定诊断流程。
- **从 hard-for-baseline 的搜索集开始**,~50–100 样本,保证一次完整运行 50 次评估内能跑完。
- **日志机器可读**(JSON、结构化目录、便于 regex 的命名)。
- **做廉价 validation 预筛**,在跑昂贵 benchmark 前过滤 malformed 候选。
- **让 proposer 之外的脚本做评估**,不要让 proposer 亲自跑 eval。
- proposer 在实验中是 **Claude Code + Opus 4.6**[^1]。

## 与其他主题的关系

- 属于 [[harness-engineering]] 范畴的一次**自动化**尝试:传统 harness 工程由人手工调 prompt / 检索 / 记忆,Meta-Harness 把这部分外包给 coding agent。
- 与 AlphaEvolve / OpenEvolve / GEPA / TextGrad / OPRO / TTT-Discover / Feedback Descent 等**文本优化器**同属一类但尺度不同 —— 它们每步用 KB 级反馈,Meta-Harness 每步用 MB 级全历史[^1]。
- 底层依赖 Claude Code 这类能在文件系统上自由导航的 coding agent,作者注明此工作流"只在 2026 年初 coding agent 能力大幅提升后才变得实用"[^1]。

## 来源引用

- [^1]: `sources/meta-harness-2603.28052v1.pdf` — Lee, Nair, Zhang, K. Lee, Khattab, Finn. *Meta-Harness: End-to-End Optimization of Model Harnesses*. arXiv:2603.28052v1, 2026-03-30. 项目页 https://yoonholee.com/meta-harness/,代码 https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact。
