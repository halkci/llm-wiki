---
title: Gemma 4 Thinking 模式
aliases: [gemma4-thinking, gemma-4-reasoning, enable_thinking]
tags: [gemma-4, thinking, reasoning, chat-template]
sources:
  - sources/gemma4-thinking-cookbook.md
  - https://github.com/google-gemma/cookbook/blob/main/docs/capabilities/thinking.ipynb
related: [[gemma-4]], [[gemma-4-function-calling]], [[deer-flow]]
created: 2026-04-05
updated: 2026-04-06
---

# Gemma 4 Thinking 模式

## 摘要

Gemma 4 原生支持一种可显式开关的 **thinking(推理)模式**:在 `apply_chat_template` 时传 `enable_thinking=True`,processor 会在 prompt 里注入相应的思考 token,模型先产生一段推理再给出最终回答。该能力同时适用于纯文本与多模态(图像+文本)任务,并可与 function calling 组合。不同尺寸的变体在 token 层有差异:小模型(E2B/E4B)用 `<|think|>` 段,大模型(26B A4B / 31B Dense)的 chat template 始终带一个**空的** thinking token 以稳定输出并抑制"幽灵思考通道"。[^1]

## 核心内容

### 如何启用

```python
text = processor.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
    enable_thinking=True,
)
```

一行参数即可,不需要单独的系统提示词来"教"模型怎么推理——思考段由 processor 通过特殊 token 注入,走的是模型训练时对齐过的通道。[^1]

### 变体差异

| 变体 | 思考 token 处理 |
|---|---|
| E2B / E4B | 思考段由 `<|think|>` 标记;即使 `enable_thinking=False`,仍可能出现"ghost" 思考通道 |
| 26B A4B / 31B Dense | chat template 始终注入一个**空的** thinking token,用来稳定输出、压制幽灵思考通道 |

> ⚠️ 工程含义:对 E2B/E4B 的输出做下游解析时,不能假设"关了 thinking 就不会出现思考 token"。解析器应无条件走 `parse_response()`,把可能存在的 thinking 段剥离后再交给下游。[^1]

### 解析输出

```python
result = processor.parse_response(response)
# result: {
#   'thinking':   "...模型的推理链...",
#   'content':    "...面向用户的最终答复...",
#   'role':       "assistant",
#   'tool_calls': [...]     # 仅当同时用了 function calling
# }
```

`parse_response()` 是官方推荐的出口,避免手工正则匹配特殊 token。它把思考、正文、角色、工具调用四类信息一次性结构化。[^1]

### 与多模态、function calling 的组合

- **多模态**:thinking 模式对图像+文本输入同样生效,无额外 API。[^1]
- **Function calling**:`enable_thinking=True` 可与 `tools=[...]` 同时传入,模型先推理再决定调什么工具;解析时 `parse_response` 同时给出 `thinking` 与 `tool_calls` 字段。详见 [[gemma-4-function-calling]]。

### 运行环境

Cookbook notebook 在 **T4 GPU** 上演示,依赖 `transformers` + PyTorch。没有额外的 thinking-only 依赖。[^1]

## 与其他主题的关系

- 与 [[gemma-4]]:本页是 [[gemma-4]] 摘要里"Thinking 模式"一行的展开——解释 `enable_thinking` 参数在 token 层实际做什么、不同尺寸变体为何行为不一致。
- 与 [[gemma-4-function-calling]]:function calling 专页提到 "`enable_thinking=True` 可在调用前走一段推理链,提升选用准确度",但未展开输出结构。本页给出 `parse_response()` 返回字典的完整 schema(`thinking` / `content` / `role` / `tool_calls`),两页互补。
- 与 [[harness-engineering]]:thinking 段的剥离是典型的 harness 层职责——模型吐混合流,是否把推理展示给用户、是否记入对话历史、是否用于评估,全由宿主代码决定。`parse_response` 提供了标准化的切分出口,正是 harness 里值得沉淀的工具函数。
- 与 [[deer-flow]]:Ollama 在调 Gemma 4 时,通过内置 `PARSER gemma4` 把 thinking 段自动拆到 chat completions 响应的 `reasoning` 字段、最终答复留在 `content` 字段。这意味着 DeerFlow 这类只读 `content` 的 langchain 客户端能天然拿到"干净答复",不需要手工调 `parse_response()`,但也因此需要把 `max_tokens` 设大(否则 reasoning 会吃光预算导致 `content` 为空)。

## 来源引用

- [^1]: [Gemma Cookbook: Thinking Capability notebook](https://github.com/google-gemma/cookbook/blob/main/docs/capabilities/thinking.ipynb),镜像于 `sources/gemma4-thinking-cookbook.md`(抓取日期 2026-04-05)
