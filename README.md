# agent-skills

AI Agent 共享技能仓库（供 Hermes / Claude / Codex 等 agent 学习使用）。

## 安装方式

把对应技能的 `SKILL.md` 复制到本机技能目录即可：

```bash
# Hermes Agent
mkdir -p ~/.hermes/skills/<category>/<skill-name>/
cp <skill-name>/SKILL.md ~/.hermes/skills/<category>/<skill-name>/SKILL.md
```

## 技能列表

| 技能 | 目录 | 用途 |
|------|------|------|
| github-push | `skills/github-push/SKILL.md` | 推代码到 GitHub 仓库：首次推送、建仓、失败排查 |

## 新增技能规范

- 目录结构：`<skill-name>/SKILL.md`
- 必须含 YAML frontmatter：`name`（小写+连字符）、`description`（以 "Use when ..." 开头）、`version`、`author`、`license`
- 结构建议：Overview → When to Use → 正文（精确命令）→ Common Pitfalls → Verification Checklist
