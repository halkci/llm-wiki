# Gemma 4 Model Card

- 原始 URL: https://ai.google.dev/gemma/docs/core/model_card_4#getting_started
- 抓取日期: 2026-04-05
- 抓取方式: WebFetch(两次,正文 + getting started)

---

## 模型总览

Gemma 4 是 Google DeepMind 发布的一组开源多模态模型,共 4 个尺寸:

| 变体 | 类型 | 参数量 | 说明 |
|---|---|---|---|
| E2B | Dense | 2.3B 有效(含 embedding 5.1B) | 支持文本、图像、音频 |
| E4B | Dense | 4.5B 有效(含 embedding 8B) | 支持文本、图像、音频 |
| 26B A4B | MoE | 25.2B 总 / 3.8B active | 文本 + 图像 |
| 31B Dense | Dense | 30.7B | 文本 + 图像 |

## 架构特征

- **Hybrid attention**:交错 local sliding window attention 与 full global attention。
- **上下文窗口**:E2B/E4B 为 128K tokens;26B A4B 与 31B 为 256K tokens。
- **视觉编码器**:小模型约 150M 参数,大模型约 550M 参数。
- **音频支持**:仅 E2B 与 E4B,最长 30 秒输入,用于 ASR 与翻译。

## 能力

文本生成、可变分辨率图像理解、视频分析、function calling、代码、140+ 语言多语种、音频处理(仅 E2B/E4B)。

## 性能基准(节选)

- MMLU Pro:31B 85.2%,26B A4B 82.6%,E4B 69.4%
- AIME 2026:31B 89.2%,26B A4B 88.3%
- LiveCodeBench:31B 80.0%,26B A4B 77.1%

## 训练数据

"Large-scale, diverse collection" — 网页文档(140+ 语言)、代码、数学、图像。数据截止 **2025 年 1 月**。

## 预期用途

内容创作、对话机器人、摘要、图像信息抽取、研究、语言学习工具。

## 限制

官方原文:"Models generate responses based on information from training datasets" — 模型不是知识库,可能产生过时或不准确的事实。

## Getting Started

### 安装

```
pip install -U transformers torch accelerate
```

### 加载模型

```python
import torch
from transformers import AutoProcessor, AutoModelForCausalLM

MODEL_ID = "google/gemma-4-E2B-it"

processor = AutoProcessor.from_pretrained(MODEL_ID)
model = AutoModelForCausalLM.from_pretrained(
    MODEL_ID,
    dtype=torch.bfloat16,
    device_map="auto"
)
```

### 基本用法

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Write a short joke about saving RAM."},
]

text = processor.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
    enable_thinking=False
)
inputs = processor(text=text, return_tensors="pt").to(model.device)
outputs = model.generate(**inputs, max_new_tokens=1024)
response = processor.decode(outputs[0], skip_special_tokens=False)
```

注意 `enable_thinking=False` 参数 —— 暗示 Gemma 4 支持 thinking 模式。

### 框架支持

Hugging Face Transformers + 兼容库(Ollama、LM Studio 等)。

### License

**Apache 2.0**,适用于所有 Gemma 4 发布物。
