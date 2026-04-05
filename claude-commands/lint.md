---
description: 对 ~/knowledge-base 本地知识库做健康检查
---

你现在要对位于 `/home/admin/knowledge-base/` 的本地 LLM Wiki 知识库做健康检查。

**重要**:当前工作目录很可能**不是**知识库目录。所有操作都用绝对路径 `/home/admin/knowledge-base/...`。不要碰用户当前所在项目目录的任何文件。

## 步骤

1. **建立上下文**:Read `/home/admin/knowledge-base/CLAUDE.md` 和 `/home/admin/knowledge-base/SCHEMA.md`。

2. **盘点**:
   - Glob `/home/admin/knowledge-base/wiki/**/*.md` 拿到全部页面
   - Read `/home/admin/knowledge-base/INDEX.md`
   - 对比两者

3. **逐项检查**并分类:

   | 类别 | 定义 |
   |---|---|
   | **孤儿页** | 存在于 wiki/ 但未登记到 INDEX,或无任何 `[[...]]` 入链 |
   | **死链** | `[[target]]` 指向不存在的 wiki/target.md |
   | **冲突** | 两个或多个页面对同一事实陈述不一致 |
   | **空洞** | 正文 < 10 行或只有 frontmatter |
   | **重复** | 标题或核心内容高度相似,疑似可合并 |
   | **过期** | `updated` 距今 > 90 天(用今天的真实日期计算) |
   | **格式违规** | frontmatter 缺字段或不符合 SCHEMA |

4. **汇报**:按分组列表给出全部发现。**先不要动手修**。格式参考 SCHEMA 里的示例。

5. **等待指示**:用户确认要修复哪些之后再动手。
   - 能自动修的(加 INDEX、补 frontmatter、修明显错别字链接):直接做。
   - 涉及合并/删除/解决冲突:逐项确认。

6. **结束**:把本次 lint 的动作追加到 `/home/admin/knowledge-base/LOG.md`。

## 禁止

- ❌ 未经确认删除或合并任何 wiki 页面。
- ❌ "顺手"重写内容——lint 的目的是结构健康,不是重构文字。
- ❌ 碰用户当前所在项目目录。
