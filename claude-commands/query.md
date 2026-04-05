---
description: 查询 ~/knowledge-base 本地知识库,基于 wiki 层合成答案
argument-hint: <问题>
---

你现在要基于位于 `/home/admin/knowledge-base/` 的本地 LLM Wiki 知识库回答用户的问题:

$ARGUMENTS

**重要**:当前工作目录很可能**不是**知识库目录。所有读取必须用绝对路径 `/home/admin/knowledge-base/...`。不要搜索或读取用户当前所在的项目目录——那里跟本次查询无关。

## 步骤

1. **建立上下文**:
   - Read `/home/admin/knowledge-base/CLAUDE.md`
   - Read `/home/admin/knowledge-base/SCHEMA.md`(只看操作规则部分即可)
   - Read `/home/admin/knowledge-base/INDEX.md`,定位与问题相关的 wiki 页面

2. **wiki 优先**:
   - Grep `/home/admin/knowledge-base/wiki/` 查找关键词
   - Read 相关的 wiki 页面作为答案的**主要依据**
   - 这是本模式的核心——wiki 是已经被消化过的知识,不要再从 sources 重新拼

3. **必要时回退到 sources**:只有当 wiki 缺失、不充分,或用户明确要原文引用时,才去读 `/home/admin/knowledge-base/sources/` 下的原始材料。

4. **给出答案**:
   - 简洁的中文。
   - 标注依据的 wiki 页面(写成 `wiki/xxx.md`,便于用户点击打开)和原始来源。
   - 如果不同 wiki 页对同一点有矛盾,明确指出,不要擅自取舍。
   - 如果知识库里根本没有相关内容,**明确说"知识库中未找到"**,不要拿训练数据冒充 wiki 内容。

5. **沉淀回路**(重要):如果你的答案本身有持久价值(综合了多页的新洞见、或填补了某个空白),主动询问用户要不要把它沉淀成新 wiki 页或并入已有页。得到同意再写入,并更新 INDEX 和 LOG(全部绝对路径)。

## 禁止

- ❌ 修改用户当前所在项目的任何文件。
- ❌ 把训练数据里的知识冒充成 wiki 的内容。
- ❌ 绕过 wiki 直接做 sources 全文检索——那就退化成普通 RAG 了。
