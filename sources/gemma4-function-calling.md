# Function Calling with Gemma 4

- 原始 URL: https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4
- 抓取日期: 2026-04-05

## Supported Models

Gemma 4 function calling is available through the following instruction-tuned variants:
- `google/gemma-4-E2B-it`
- `google/gemma-4-E4B-it`
- `google/gemma-4-31B-it`
- `google/gemma-4-26B-A4B-it`

## Core Concept

"When you generate code with function calling, you must run the generated code yourself or run it as part of your application." 模型本身不执行工具,只产出结构化调用请求,执行与返回结果由宿主应用负责。

## Tool Definition Methods

### 1. JSON Schema(手动)

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
    }
}
```

### 2. Raw Python Functions

直接把 Python 函数作为 tool 传入,框架会根据 type hints + docstring 自动生成 JSON schema。要求 docstring 遵循 **Google Python Style Guide**。

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

## 三阶段工作流

1. **Model's Turn**: 模型收到 user prompt + tools 列表,生成函数调用对象(含 name 与 arguments)。
2. **Developer's Turn**: 应用解析输出、取出函数名与参数、**校验后**再执行。
3. **Final Response**: 将工具执行结果回传模型,模型据此合成自然语言答案。

## Chat Template 集成

工具通过 `apply_chat_template` 的 `tools` 参数传入:

```python
text = processor.apply_chat_template(
    message,
    tools=[weather_function_schema],
    tokenize=False,
    add_generation_prompt=True,
)
```

## Tool Response 格式

把执行结果按以下结构附回对话:

```python
"tool_responses": [
    {
        "name": function_name,
        "response": function_response,
    }
]
```

并行多调用:

```python
"tool_responses": [
    {"name": function_name_1, "response": function_response_1},
    {"name": function_name_2, "response": function_response_2},
]
```

## Function Calling with Thinking

开启思考链以提高调用准确度:

```python
text = processor.apply_chat_template(
    message,
    tools=tools,
    tokenize=False,
    add_generation_prompt=True,
    enable_thinking=True,
)
```

解析模型输出(区分思考段与调用段):

```python
result = processor.parse_response(output)
```

## 注意事项

- **自定义对象参数**:自动 schema 生成对复杂对象通常只标成通用 "object",嵌套属性丢失。对复杂参数类型应手动编写 JSON schema,显式声明嵌套字段。
- **安全**:执行前必须校验函数名与参数。生产环境中用 `globals()` 动态分发函数是危险做法。
- **特殊 token**:模型通过特殊 token 输出函数调用,框架会自动处理。
- **System message**:可选,但推荐。
- **硬件**:官方 notebook 示例使用 T4 GPU。
- **依赖**:Hugging Face Transformers。
