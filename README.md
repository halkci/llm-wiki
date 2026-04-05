# 个人 LLM Wiki 知识库

基于 Andrej Karpathy 的 [LLM Wiki 模式](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 搭建的本地个人知识库。核心理念:**让 LLM 把原始资料持续地"消化"成一份不断演化的个人 wiki,而不是每次查询都从头检索文档**。

## 目录结构

```
knowledge-base/
├── sources/          ← 放原始资料(PDF / md / txt / 网页保存)
├── wiki/             ← LLM 维护的知识页面(你可以读、也可以手写)
├── SCHEMA.md         ← 页面格式 / 命名 / 交叉引用规则
├── INDEX.md          ← 全部页面的主题索引
├── LOG.md            ← 变更历史
├── CLAUDE.md         ← Claude Code 行为准则
└── .claude/commands/ ← /ingest /query /lint 三个 slash 命令
```

## 使用方式

在此目录打开 Claude Code:
```bash
cd /home/admin/knowledge-base
claude
```

然后使用三个命令之一:

### 摄入资料
```
/ingest sources/karpathy-llm-wiki.md
/ingest https://example.com/some-article
```
Claude 会读取源材料,提取要点,决定是新建还是更新已有 wiki 页,维护交叉引用,最后汇报改动。

### 查询
```
/query 什么是 LLM Wiki 模式?它和 RAG 有什么区别?
```
Claude 优先从 `wiki/` 合成答案(而不是从原始 `sources/` 重新检索),并标注出处。如果答案有保存价值,会问你要不要沉淀回 wiki。

### 健康检查
```
/lint
```
扫描死链、孤儿页、矛盾、重复、过期内容,给出修复建议。

## 手动操作

- `sources/` 是不可变的,你直接把文件拖进去即可。
- `wiki/` 你可以手写、手改,Claude 会尊重人写的内容。
- `SCHEMA.md` 是规则,你可以按自己的偏好修改(例如改变页面 frontmatter 格式),改完下次会话自动生效。

## 备份建议

```bash
cd /home/admin/knowledge-base
git init
git add . && git commit -m "init"
```
之后每次重大 ingest 后提交一次,就有了完整的演化历史。
