---
title: DeerFlow
aliases: [deer-flow, deerflow, bytedance-deer-flow]
tags: [agent-framework, harness, local-llm, ollama, integration-notes]
sources:
  - https://github.com/bytedance/deer-flow
  - https://github.com/bytedance/deer-flow/blob/main/config.example.yaml
  - https://github.com/bytedance/deer-flow/blob/main/Install.md
related: [[gemma-4]], [[gemma-4-function-calling]], [[gemma-4-thinking]], [[harness-engineering]]
created: 2026-04-06
updated: 2026-04-06
---

# DeerFlow

## 摘要

**DeerFlow** 是字节开源的长时程 SuperAgent harness:带沙箱、记忆、工具、skill、子代理和消息网关,负责"researcher / coder / reporter"多角色协作,覆盖分钟到小时级任务。本页只沉淀 **用本地 Ollama 驱动 DeerFlow 的接入经验**,不复述上游文档 —— 这些是实战踩过、文档里不会直说的坑。[^1]

## 核心内容

### 版本形态要先分清

DeerFlow 有两套**互不兼容**的配置格式,接入前必须确认当前仓库属于哪一代:

| 代 | 配置文件 | 模型字段 | 识别方法 |
|---|---|---|---|
| 1.x | `conf.yaml` | 顶层 `BASIC_MODEL` / `REASONING_MODEL` / `VL_MODEL`,底层走 **litellm**,model 名需 `ollama_chat/` 前缀 | 仓库根有 `conf.yaml.example` |
| **2.x(当前 main)** | `config.yaml` | `models:` 列表,每项含 `use: <class path>`,底层走 **langchain**,model 名原样传 | 仓库根有 `config.example.yaml` + `backend/` + `frontend/` 两个目录 |

中文区大量教程和 ByteDance 官方 docs 里的 Ollama 例子**都是 1.x 的**(含 `BASIC_MODEL:` 的 YAML),直接照抄到 2.x 会得到"no models configured"。[^2]

### 2.x 的最小 Ollama 接入

`config.yaml` 里 `models:` 列表加一条即可,第一条自动成为默认模型:

```yaml
models:
  - name: gemma4-31b-local
    display_name: Gemma 4 31B (Local Ollama, 256K ctx)
    use: langchain_openai:ChatOpenAI      # 注意:不是 litellm,不需要 ollama_chat/ 前缀
    model: gemma4:31b-ctx256k              # 原样传给 Ollama,必须与 `ollama list` 里的 tag 一致
    api_key: ollama                        # Ollama 不校验,但 langchain 的 ChatOpenAI 要求非空
    base_url: http://host.docker.internal:11434/v1   # Docker 部署下必须用 host.docker.internal,见下
    request_timeout: 600.0
    max_retries: 2
    max_tokens: 8192
    temperature: 0.7
    supports_vision: true                  # Gemma 4 31B 原生多模态,开启后 view_image 工具可用
```

### Linux Docker 部署的两个必踩坑

DeerFlow 推荐路径是 Docker(`make docker-init && make docker-start`,见 `Install.md`)。在 Linux 宿主 + 本地 Ollama 的组合下,有两件事文档不提但必做:

**1. Ollama 默认只绑 `127.0.0.1`,容器访问不到。** 必须让 Ollama 监听 `0.0.0.0`。加 systemd drop-in(不要直接改主 unit 文件):

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf > /dev/null <<'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
EOF
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

验证:`ss -tlnp | grep 11434` 应显示 `*:11434` 而不是 `127.0.0.1:11434`。

**2. `base_url` 必须写 `host.docker.internal`,不能写 `localhost`。** localhost 在容器里指容器自己。DeerFlow 的 `docker/docker-compose-dev.yaml` 已经给 `gateway` / `langgraph` 服务加了 `extra_hosts: ["host.docker.internal:host-gateway"]`,所以这个域名在 Linux 上也能解析到宿主网关 IP,直接用就行。

### Ollama 的 `num_ctx` 默认只有 2048 —— 必须派生新 tag

无论用什么模型,Ollama 默认 `num_ctx=2048`,跑 DeerFlow 这种会吞上下文的 agent 必被截断。用 `ollama show --modelfile <tag>` 看基础 tag 是否设置了 `PARAMETER num_ctx`,没有就派生:

```bash
cat > /tmp/ctx.Modelfile <<'EOF'
FROM gemma4:31b
PARAMETER num_ctx 262144   # Gemma 4 31B 原生最大 256K,见 [[gemma-4]]
EOF
ollama create gemma4:31b-ctx256k -f /tmp/ctx.Modelfile
```

派生 tag 复用 base 的 GGUF 权重 blob,**不会多占磁盘**,只多一份 manifest 和 param layer。

### Gemma 4 thinking 模式对 chat completions 输出的影响

Ollama 为 gemma4 内置了专用 `RENDERER gemma4` / `PARSER gemma4`(在 `ollama show --modelfile` 里可见),实测行为:

- 模型**默认开启 thinking**,返回的 `chat.completions` JSON 里 `message` 同时有两个字段:
  - `reasoning`: 完整的推理链(Gemma 4 的原生 thinking 段,见 [[gemma-4-thinking]])
  - `content`: 最终对用户可见的回答
- langchain 的 `ChatOpenAI` **只读 `content`**,所以 DeerFlow 拿到的是干净的答案,不会被推理链污染。
- 但 `max_tokens` 必须足够大 —— 实测 `max_tokens=64` 时 64 token 全被 reasoning 吃完,`content` 留空。生产配置用 2048~8192 比较安全。

### Tool calling 无需额外工作

网上关于 "gemma 不支持 tool calling" 的说法都是 gemma 2/3 时代的。Gemma 4 原生支持(见 [[gemma-4-function-calling]]),**而且 Ollama 侧是代码级支持**:Modelfile 里的 `RENDERER gemma4` / `PARSER gemma4` 两行就是 Ollama 内置的 gemma4 函数调用渲染器和解析器,负责把 OpenAI 风格的 `tools=[...]` 翻译成 gemma4 的 `<|tool>` / `<|tool_call>` / `<|tool_response>` 特殊 token,再把模型输出解析回 OpenAI 格式的 `tool_calls`。DeerFlow 的 researcher / coder 节点直接用不需要任何 patch。

### 验证接入是否成功的最小序列

按依赖顺序执行,任何一步失败都能定位问题层:

1. `ss -tlnp | grep 11434` → `*:11434`(Ollama 监听正确)
2. `curl -sS http://localhost:11434/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"<tag>","messages":[{"role":"user","content":"hi"}],"stream":false,"max_tokens":512}'` → `content` 非空(模型本体可用 + num_ctx 足够)
3. `sudo docker exec deer-flow-gateway curl -sS http://host.docker.internal:11434/api/tags` → 列出 tag(容器到宿主网络打通)
4. `curl http://localhost:2026/api/models` → 返回的 JSON 里能看到 `config.yaml` 注入的模型条目(DeerFlow 识别配置)
5. 浏览器打开 `http://localhost:2026`,提交任务观察 researcher 节点能正常触发工具调用

### `ollama ps` 的读数要看懂

加载后 `ollama ps` 的 `SIZE` 列包含权重 + KV cache + 激活,**跟 disk 上的 `ollama list` SIZE 差异巨大**。Gemma 4 31B GGUF ~19 GB,但 `num_ctx=262144` 加载后显示约 **72 GB**,差出来的 50+ GB 几乎全是 KV cache。这是估算长上下文硬件预算的实际依据,不要看 disk size。

## 与其他主题的关系

- 与 [[gemma-4]]:本页是"把 [[gemma-4]] 里记录的 256K 上下文、hybrid attention 等能力**真正跑起来**"的落地笔记。变体矩阵、架构细节不重复记录,查 [[gemma-4]]。
- 与 [[gemma-4-function-calling]]:DeerFlow 能把 researcher/coder 节点直接跑在 Gemma 4 上,依赖的正是那篇记录的原生 function calling 能力。上游实现细节(`<|tool>` 特殊 token、`tool_responses` 回传格式)看那页。
- 与 [[gemma-4-thinking]]:Ollama 返回体拆成 `reasoning` + `content` 的现象,是 gemma4 thinking 模式被 Ollama `PARSER gemma4` 自动解析的结果。thinking 本身的原理看那页。
- 与 [[harness-engineering]]:DeerFlow 整个项目就是一个庞大的 harness 样本 —— 沙箱、记忆、subagents、消息网关、guardrails 全是模型外的"上下文工程"代码。本页踩的坑(`num_ctx` 派生、`host.docker.internal` 映射、`reasoning`/`content` 拆分)都是 harness 层的典型工作,并非模型能力问题。

## 来源引用

- [^1]: [bytedance/deer-flow README](https://github.com/bytedance/deer-flow)(抓取日期 2026-04-06)
- [^2]: 1.x vs 2.x 差异来自比对 `config.example.yaml`(2.x)与中文区现存 `conf.yaml` 教程 —— 见 [deer-flow-cn 配置指南](https://github.com/drfccv/deer-flow-cn/blob/main/docs/configuration_guide.md)(1.x 格式)。DeerFlow 2.x 的配置样板见 [config.example.yaml](https://github.com/bytedance/deer-flow/blob/main/config.example.yaml)。
- 其余事实(Modelfile 内容、`ollama ps` 输出、`reasoning`/`content` 拆分、docker-compose 的 `extra_hosts` 映射)均为 2026-04-06 本地实测,无外部来源。
