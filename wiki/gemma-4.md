---
title: Gemma 4
aliases: [Gemma4, gemma-4, google-gemma-4]
tags: [model, open-weights, multimodal, google, deepmind]
sources:
  - sources/gemma4-model-card.md
  - https://ai.google.dev/gemma/docs/core/model_card_4
related: [[harness-engineering]], [[gemma-4-function-calling]], [[gemma-4-thinking]], [[deer-flow]]
created: 2026-04-05
updated: 2026-04-06
---

# Gemma 4

## 摘要

Gemma 4 是 Google DeepMind 发布的开源多模态模型家族,包含 4 个尺寸(E2B、E4B、26B A4B MoE、31B Dense),统一采用 **hybrid attention**(局部滑窗 + 全局注意力交错)。小模型(E2B/E4B)同时支持文本、图像、音频,大模型仅支持文本与图像。全系列使用 Apache 2.0 协议,权重发布于 Hugging Face。[^1]

## 核心内容

### 变体矩阵

| 变体 | 类型 | 参数 | 上下文 | 模态 |
|---|---|---|---|---|
| **E2B** | Dense | 2.3B 有效(5.1B 含 embedding) | 128K | 文本 + 图像 + 音频 |
| **E4B** | Dense | 4.5B 有效(8B 含 embedding) | 128K | 文本 + 图像 + 音频 |
| **26B A4B** | MoE | 25.2B 总 / 3.8B active | 256K | 文本 + 图像 |
| **31B Dense** | Dense | 30.7B | 256K | 文本 + 图像 |

E 前缀代表"effective"(有效参数),区分了 embedding 之外的计算参数量。[^1]

### 架构要点

- **Hybrid attention**:交错使用 local sliding window attention 与 full global attention —— 在长上下文下平衡计算成本与全局信息流。[^1]
- **视觉编码器**:小模型约 150M 参数,大模型约 550M;支持可变分辨率输入。[^1]
- **音频路径**:仅 E2B/E4B,最长 30 秒,用于 ASR 与翻译。[^1]
- **Thinking 模式**:`apply_chat_template` 有 `enable_thinking` 参数,模型原生支持推理链显式开关;不同尺寸的 token 处理方式有差异,详见 [[gemma-4-thinking]]。[^2]

### 训练

数据来自"large-scale, diverse collection":140+ 语言的网页文档、代码、数学、图像。**数据截止 2025-01**。[^1]

### 性能(官方基准)

| Benchmark | 31B Dense | 26B A4B | E4B |
|---|---|---|---|
| MMLU Pro | 85.2 | 82.6 | 69.4 |
| AIME 2026 | 89.2 | 88.3 | — |
| LiveCodeBench | 80.0 | 77.1 | — |

[^1]

### Getting Started

安装:
```
pip install -U transformers torch accelerate
```

最小加载 + 生成示例:
```python
import torch
from transformers import AutoProcessor, AutoModelForCausalLM

MODEL_ID = "google/gemma-4-E2B-it"
processor = AutoProcessor.from_pretrained(MODEL_ID)
model = AutoModelForCausalLM.from_pretrained(
    MODEL_ID, dtype=torch.bfloat16, device_map="auto"
)

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Write a short joke about saving RAM."},
]
text = processor.apply_chat_template(
    messages, tokenize=False,
    add_generation_prompt=True, enable_thinking=False
)
inputs = processor(text=text, return_tensors="pt").to(model.device)
outputs = model.generate(**inputs, max_new_tokens=1024)
```

兼容框架:Hugging Face Transformers、Ollama、LM Studio 等。[^2]

### License

**Apache 2.0**,全系列适用。[^1]

## 与其他主题的关系

- 与 [[harness-engineering]]:Gemma 4 的 chat template、`enable_thinking` 开关、processor 抽象本身就是 harness 层的一部分 —— 模型权重之外决定"怎么存/取/呈现上下文"的代码。
- 与 [[deer-flow]]:把 Gemma 4 31B 真正接到一个 agent harness(DeerFlow)里跑起来的实战记录——包括怎么派生大 `num_ctx` 的 Ollama tag、怎么处理 thinking 模式下 `reasoning`/`content` 拆分、怎么解决 Linux Docker 下的 `host.docker.internal` 问题。
- 与 [[gemma-4-function-calling]]:全部四个 `-it` 变体原生支持 function calling,细节见专页。
- 与 [[gemma-4-thinking]]:`enable_thinking` 参数的 token 层实现、`parse_response()` 输出结构、小/大模型的幽灵思考通道差异都在那一页展开。

## 来源引用

- [^1]: [Gemma 4 Model Card](https://ai.google.dev/gemma/docs/core/model_card_4),镜像于 `sources/gemma4-model-card.md`
- [^2]: [Getting Started 节](https://ai.google.dev/gemma/docs/core/model_card_4#getting_started),镜像于 `sources/gemma4-model-card.md`
