---
title: Harness 工程(Harness Engineering)
aliases: [harness engineering, harness 工程, llm harness]
tags: [llm, agent, concept]
sources:
  - sources/meta-harness-2603.28052v1.pdf
related: [[meta-harness]], [[deer-flow]]
created: 2026-04-05
updated: 2026-04-06
---

# Harness 工程(Harness Engineering)

## 摘要

**Harness**(中文常译作"外壳""装具")指 LLM 应用中**围绕固定模型权重**的那层代码 —— 它决定存什么、什么时候取、怎么组织上下文呈现给模型。一个同模型的 6× 性能差距往往不来自模型本身,而来自 harness[^1]。**Harness 工程**就是迭代改进这层代码以提升整体系统表现的实践;到 2026 年为止,它仍然大量依赖人工检视失败、调启发式、手改 prompt[^1]。

## 核心内容

### 为什么重要

- harness 在**长时程**上起作用:一次"存什么"的决定可能在许多推理步之后才影响行为;调试需要回溯大跨度的因果链[^1]。
- 在 Meta-Harness 论文研究的任务上,仅改 harness 就能在同一 benchmark 上带来 6× 性能差距[^1]。
- 同一 harness 上几十个并行团队同时在做专项调优(如 TerminalBench-2 上的 Terminus, Terminus-KIRA, ForgeCode, Claude Code, Goose, Mini-SWE-Agent, Droid, Mux, TongAgents, MAYA-V2 等)[^1]。

### 组成部分

一个典型的 harness 至少决定:

1. **Prompt 构造** —— 指令、系统消息、few-shot 样例选择与排序。
2. **检索策略** —— 检索什么语料、用什么 embedding / 词法器、top-k、重排。
3. **记忆管理** —— 哪些对话/轨迹片段保留、压缩、丢弃、再使用。
4. **控制流** —— 工具调用协议、验证循环、完成条件、错误恢复。
5. **状态更新** —— 如何在一轮轮调用间更新内部状态。

### 现状:仍主要是手工

尽管重要,harness 工程**主要仍由人完成**:工程师看失败案例、猜原因、改代码、再跑一次。已有的文本优化器(OPRO、TextGrad、GEPA、AlphaEvolve、OpenEvolve、Feedback Descent、TTT-Discover)主要针对 prompt 或小段文本 artifact,**反馈被重度压缩**(分数、摘要、短模板),不适合 harness 这种需要跨代码/轨迹/分数做长程诊断的对象[^1]。

### 自动化的尝试

[[meta-harness]] 是目前最系统的自动化 harness 工程尝试:把整个 harness 历史(源码 + 执行轨迹 + 分数)放进文件系统,让 coding agent 作为 proposer 自由 grep/cat,自行诊断、自行改代码。它在在线文本分类、检索增强数学推理、TerminalBench-2 agentic coding 三个领域都超过了最佳人工 harness[^1]。

### 相关资源(作者列出的外部参考)

- OpenAI 博客 *Harness engineering: leveraging Codex in an agent-first world*, 2026-02[^1]
- Anthropic Engineering *Effective harnesses for long-running agents*, Justin Young, 2025-11[^1]
- Martin Fowler 博客 *Harness engineering*, Birgitta Böckeler, 2026-03[^1]
- Can Bölük *I improved 15 LLMs at coding in one afternoon. Only the harness changed.*, 2026-02[^1]

## 与其他主题的关系

- [[meta-harness]] 是这个概念的自动化实例;它论证了"harness 搜索"在 2026 年的 coding agent 能力下已经可行。
- [[deer-flow]] 是一个具体的工业级 harness 样本(ByteDance 开源):沙箱、记忆、subagents、消息网关、guardrails 全是模型外的"上下文工程"代码。那页里记录的所有接入坑(`num_ctx` 派生、`host.docker.internal` 映射、`reasoning`/`content` 拆分)都是 harness 层工程,而非模型能力问题 —— 正好印证了本页"性能差距往往来自 harness 而非模型"的论点。

## 来源引用

- [^1]: `sources/meta-harness-2603.28052v1.pdf` — 引言、Related Work、参考文献 §9、§10、§21、§36。
