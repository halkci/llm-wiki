---
title: Gemma 4 Function Calling
aliases: [gemma4-function-calling, gemma-4-tools, gemma4-tool-use]
tags: [gemma-4, function-calling, tool-use, agents, harness]
sources:
  - sources/gemma4-function-calling.md
  - https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4
related: [[gemma-4]], [[harness-engineering]], [[gemma-4-thinking]], [[deer-flow]]
created: 2026-04-05
updated: 2026-04-06
---

# Gemma 4 Function Calling

## 摘要

Gemma 4 指令微调变体(E2B-it / E4B-it / 26B-A4B-it / 31B-it)原生支持 **function calling**:宿主应用在 `apply_chat_template` 时传入工具定义,模型输出结构化调用请求(由特殊 token 包裹),由宿主执行并把结果回传,模型再合成最终自然语言回答。结合 `enable_thinking=True` 可在调用前走一段推理链,提升选用准确度。[^1]

## 核心内容

### 支持范围

仅 `-it`(instruction-tuned)变体支持,全部四个尺寸都可用:

| 变体 | ID |
|---|---|
| E2B-it | `google/gemma-4-E2B-it` |
| E4B-it | `google/gemma-4-E4B-it` |
| 26B A4B-it | `google/gemma-4-26B-A4B-it` |
| 31B-it | `google/gemma-4-31B-it` |

即参数量最小的 E2B 也具备完整 function calling 能力。[^1]

### 工具定义两种写法

**1. 显式 JSON Schema** —— 对复杂/嵌套参数推荐。

```python
weather_function_schema = {
    "type": "function",
    "function": {
        "name": "get_current_temperature",
        "description": "Gets the current temperature for a given location.",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "The city name, e.g. San Francisco",
                },
            },
            "required": ["location"],
        },
    },
}
```

**2. 原生 Python 函数** —— 框架根据 type hints + docstring 自动生成 schema,要求 docstring 遵循 **Google Python Style Guide**(`Args:` 小节、类型、`choices: [...]` 等)。

```python
def get_current_weather(location: str, unit: str = "celsius"):
    """
    Gets the current weather in a given location.

    Args:
        location: The city and state, e.g. "San Francisco, CA" or "Tokyo, JP"
        unit: The unit to return the temperature in. (choices: ["celsius", "fahrenheit"])
    """
    return {"temperature": 15, "weather": "sunny"}
```

> ⚠️ 自动 schema 对**自定义对象参数**常常只写成通用 `"object"`,嵌套字段会丢。只要参数类型稍复杂,就应回退到手写 JSON Schema。[^1]

### 三阶段工作流

1. **Model's turn**:模型收到 prompt + tools,产出调用对象(函数名 + 参数 JSON)。
2. **Developer's turn**:宿主解析输出,**校验**函数名与参数,然后执行。
3. **Final response**:把执行结果按 `tool_responses` 结构回传,模型据此合成自然语言回答。

"When you generate code with function calling, you must run the generated code yourself or run it as part of your application." —— 模型不执行工具,执行边界完全在宿主侧。[^1]

### Chat Template 用法

```python
text = processor.apply_chat_template(
    message,
    tools=[weather_function_schema],
    tokenize=False,
    add_generation_prompt=True,
)
```

`tools=` 参数与 [[gemma-4]] 的 `apply_chat_template` 是同一入口;开启 thinking 时追加 `enable_thinking=True`,并用 `processor.parse_response(output)` 把输出拆成思考段 + 调用段(返回字典含 `thinking` / `content` / `role` / `tool_calls` 四个字段,详见 [[gemma-4-thinking]])。[^1]

### Tool Response 回传格式

单次调用:

```python
"tool_responses": [
    {"name": function_name, "response": function_response},
]
```

并行多调用:

```python
"tool_responses": [
    {"name": function_name_1, "response": function_response_1},
    {"name": function_name_2, "response": function_response_2},
]
```

即支持一次 turn 内多工具并行调用。[^1]

### 安全要点

- 执行前校验函数名白名单与参数 schema。
- 不要直接 `globals()[name](**args)` 动态分发,这是生产环境下常见的注入入口。
- 模型通过特殊 token 输出调用,务必让 processor 负责 tokenize / parse,不要手动拼字符串。[^1]

## 与其他主题的关系

- 与 [[gemma-4]]:本页是 [[gemma-4]] 能力章节的深入展开。所有四个 `-it` 变体共享同一套 function calling 接口,调用入口即 [[gemma-4]] 已介绍的 `apply_chat_template`,仅多了 `tools=` 与可选 `enable_thinking=`。
- 与 [[harness-engineering]]:function calling 是典型的 harness 层职责 —— 模型只吐结构化意图,"工具注册表、参数校验、执行沙箱、结果回填格式" 全部由模型外的代码决定。`tool_responses` 的 schema、特殊 token 的解析、thinking 段的剥离,都是 harness 工程里可沉淀的组件。
- 与 [[deer-flow]]:Gemma 4 的 function calling 能力通过 Ollama 侧的内置 `RENDERER gemma4` / `PARSER gemma4` 成为 OpenAI 兼容 `/v1` 接口上的一等公民,使得 DeerFlow 之类直接走 `langchain_openai:ChatOpenAI` 的 agent harness 能不加 patch 地让 Gemma 4 承担 researcher/coder 角色。那页记录了从配置到上线的完整链路验证。

## 来源引用

- [^1]: [Function Calling with Gemma 4](https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4),镜像于 `sources/gemma4-function-calling.md`(抓取日期 2026-04-05)
