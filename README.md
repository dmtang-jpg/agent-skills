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
| nju-box-upload | `skills/nju-box-upload/SKILL.md` | 上传文件到 NJU Box 云盘并生成分享链接（JWT 认证 + 代理绕过） |
| wsl-distro-lifecycle | `skills/wsl-distro-lifecycle/SKILL.md` | WSL 7×24 常驻保活：空闲回收诊断 + 三层计划任务架构（夜王实证） |

## 参考文档

| 文档 | 说明 |
|------|------|
| `docs/wsl-keepalive-solution.md` | WSL 长活完整方案：原理、脚本原文、部署步骤、验证方法 |

## 新增技能规范

- 目录结构：`<skill-name>/SKILL.md`
- 必须含 YAML frontmatter：`name`（小写+连字符）、`description`（以 "Use when ..." 开头）、`version`、`author`、`license`
- 结构建议：Overview → When to Use → 正文（精确命令）→ Common Pitfalls → Verification Checklist
