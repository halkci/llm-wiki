# LOG — 变更日志

按时间倒序记录对知识库的每次操作。格式:
```
## YYYY-MM-DD
- [操作类型] 简述 → 影响的文件
```

操作类型:`ingest` / `update` / `new` / `merge` / `lint` / `delete`

---

## 2026-04-06
- [new] DeerFlow 本地 Ollama 接入实战 → 新建 wiki/deer-flow.md,更新 INDEX.md 加入"框架与集成"小节;与 [[gemma-4]]/[[gemma-4-function-calling]]/[[gemma-4-thinking]]/[[harness-engineering]] 建立反向链接(待在各页追加)

## 2026-04-05
- [ingest] Gemma cookbook thinking.ipynb (github.com/google-gemma/cookbook) → 新增 sources/gemma4-thinking-cookbook.md,新建 wiki/gemma-4-thinking.md,更新 wiki/gemma-4.md 与 wiki/gemma-4-function-calling.md(加反向链接)
- [ingest] Gemma 4 function calling 文档 (ai.google.dev) → 新增 sources/gemma4-function-calling.md,新建 wiki/gemma-4-function-calling.md,更新 wiki/gemma-4.md(加反向链接)
- [ingest] Gemma 4 model card (ai.google.dev) → 新增 sources/gemma4-model-card.md,新建 wiki/gemma-4.md
- [ingest] sources/meta-harness-2603.28052v1.pdf → 新建 wiki/meta-harness.md, wiki/harness-engineering.md
- [init] 知识库初始化,按 Karpathy LLM Wiki 模式搭建三层架构
