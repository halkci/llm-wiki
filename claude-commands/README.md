# Claude Code Slash Commands

这三个命令是全局 slash 命令,让你在**任何目录**打开 Claude Code 都能直接操作本知识库:

- `ingest.md` — `/ingest <文件或URL>`:把源材料摄入知识库
- `query.md` — `/query <问题>`:基于 wiki 层合成答案
- `lint.md` — `/lint`:知识库健康检查

## 安装

把这三个文件复制(或软链)到 `~/.claude/commands/`:

```bash
mkdir -p ~/.claude/commands
cp claude-commands/*.md ~/.claude/commands/
# 或者软链,以便后续更新同步:
# ln -sf "$PWD/claude-commands/"*.md ~/.claude/commands/
```

## 路径假设

命令里写死了知识库路径 `/home/admin/knowledge-base/`。**如果你的知识库不在这个路径,需要先把三个文件里的路径改掉**,然后再复制到 `~/.claude/commands/`。

建议的一次性替换:
```bash
KB=/your/path/to/knowledge-base
sed -i "s|/home/admin/knowledge-base|$KB|g" ~/.claude/commands/{ingest,query,lint}.md
```
