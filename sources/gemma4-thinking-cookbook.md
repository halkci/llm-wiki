# Gemma 4 Thinking Capability — Cookbook Notebook

- 原始 URL: https://github.com/google-gemma/cookbook/blob/main/docs/capabilities/thinking.ipynb
- 抓取日期: 2026-04-05
- 抓取方式: WebFetch(GitHub blob 页面的 notebook 渲染)

---

## Overview

Gemma 4 includes a thinking mode that enables the model to generate reasoning
processes before providing final answers. This capability works for both
text-only and multimodal (image-text) tasks.

## Enabling Thinking Mode

To activate thinking in Gemma 4, pass `enable_thinking=True` when using the
processor:

```python
text = processor.apply_chat_template(
    message,
    tokenize=False,
    add_generation_prompt=True,
    enable_thinking=True,
)
```

The processor automatically inserts the correct thinking tokens into the
prompt, instructing the model to reason before responding.

## Model Variants

Different Gemma 4 model sizes handle thinking differently:

- **E2B / E4B**: use `<|think|>` tokens to mark thinking sections.
- **26B / 31B**: the chat template always includes an *empty* thinking token
  to stabilize output and suppress "ghost" thought channels that may appear
  even when thinking is disabled.

## Parsing Output

Use the processor's `parse_response()` utility to split the completion into
reasoning vs. final content:

```python
result = processor.parse_response(response)
# → dict with keys: 'thinking', 'content', 'role',
#                   and optionally 'tool_calls'
```

## Runtime Notes

- Notebook is demonstrated on a T4 GPU.
- Requires `transformers` and PyTorch.

## Key Caveat

E2B/E4B models may surface "ghost" thought channels even with thinking
disabled; larger 26B/31B models use an empty thinking token to suppress this
and stabilize outputs.
