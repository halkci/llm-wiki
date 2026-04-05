---
description: 把任意来源摄入 ~/knowledge-base 本地知识库
argument-hint: <文件路径或 URL>
---

你现在要执行 **ingest** 操作,把以下源材料并入位于 `/home/admin/knowledge-base/` 的本地 LLM Wiki 知识库:

$ARGUMENTS

**重要**:当前工作目录很可能**不是**知识库目录。所有对知识库的读写都必须使用绝对路径 `/home/admin/knowledge-base/...`。用户当前所在的项目目录与本操作无关,不要碰它。

## 步骤

1. **建立上下文**(每次都要做,因为 CLAUDE.md 不会自动加载):
   - Read `/home/admin/knowledge-base/CLAUDE.md`(行为准则)
   - Read `/home/admin/knowledge-base/SCHEMA.md`(格式与规则)
   - Read `/home/admin/knowledge-base/INDEX.md`(当前都有哪些页面)
   - Read `/home/admin/knowledge-base/LOG.md` 最近 20 行

2. **解析参数,获取源材料**:
   - 如果参数是 URL:用 WebFetch 获取正文,把精炼后的 Markdown 存到 `/home/admin/knowledge-base/sources/<kebab-case-name>.md`,首行写清原始 URL 与抓取日期。
   - 如果参数是**绝对路径**:Read 该文件;然后把它(或其文本化版本)**复制**到 `/home/admin/knowledge-base/sources/` 下(用 `cp` via Bash,或 Read+Write)。原文件**不要动**。
   - 如果参数是**相对路径**:先用 Bash 的 `pwd` 确定当前目录,再拼成绝对路径处理,同样复制到 sources/。
   - 如果参数看起来像项目里的一段代码片段或一段文字(不是文件也不是 URL),把它作为文本直接写入 `/home/admin/knowledge-base/sources/<timestamp>-<slug>.md`,并在文件头记录来源上下文(例如"来自项目 X 的 foo.py 第 30-50 行")。

3. **提取要点**:列出这份材料里的核心事实、概念、实体、论点,以及可能与现有 wiki 页面的关联。

4. **决策**:每个要点是 (a) 更新已有页面 / (b) 新建页面 / (c) 跨多个页面插入。参考 SCHEMA 的"创建新页 vs. 更新已有页"规则。

5. **执行写入**(全部用绝对路径 `/home/admin/knowledge-base/wiki/xxx.md`):
   - 严守 frontmatter 格式、`[[wiki-link]]` 交叉引用、`[^n]` 来源脚注。
   - 来源脚注统一指向 `/home/admin/knowledge-base/sources/...`(或外部 URL)。
   - 每改一页,考虑是否要在相关页加反向链接。

6. **同步元文件**:
   - 更新 `/home/admin/knowledge-base/INDEX.md`
   - 追加一行到 `/home/admin/knowledge-base/LOG.md`(用今天真实日期)

7. **汇报**(按 CLAUDE.md 的汇报格式):列出新建/更新的文件、发现的冲突,保持简短。

## 禁止

- ❌ 修改用户当前所在项目目录的任何文件。
- ❌ 修改 `/home/admin/knowledge-base/sources/` 里已有的文件(只能新增)。
- ❌ 编造来源或 URL。
- ❌ 在没读 SCHEMA/INDEX 的情况下就动手写 wiki。

源材料太大或主题跨度太广时,先暂停,列出处理计划让用户确认。
