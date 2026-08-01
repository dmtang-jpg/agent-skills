---
name: github-push
description: "Use when pushing local code to a GitHub repository: first push, new repos, fixing failed pushes. 推代码到GitHub仓库的完整流程。"
version: 1.0.0
author: dmtang-jpg (Hermes Agent)
license: MIT
metadata:
  hermes:
    tags: [GitHub, git, push, deploy, 推代码, 仓库]
    related_skills: [github-auth, github-repo-management, github-pr-workflow]
---

# GitHub 推仓库 (Push to GitHub)

## Overview

把本地代码推到 GitHub 的标准流程：从 `git init` 到远程仓库创建再到 `git push -u origin main`，以及失败排查。优先 `gh` CLI（已装时一条命令完成建仓+推送），无 `gh` 时用纯 `git` 手动流程。

## When to Use

- 用户说"推到 GitHub / 建个仓库 / push 一下 / 让大家看到代码"
- 本地已有代码目录，需要首次推到远程
- `git push` 报错需要排查（non-fast-forward、认证失败、master/main 不匹配、大文件被拒等）
- 交付物需要共享给团队或做成公开仓库

**Don't use for**: 只读操作（clone/fork/查仓库）→ `github-repo-management`；认证配置（token/SSH/gh login）→ `github-auth`；改已有仓库代码走 PR → `github-pr-workflow`。

## 0. 前置检查（30 秒）

```bash
gh auth status 2>/dev/null && echo "✓ gh 已认证" || echo "✗ 需要认证，见 github-auth 技能"
git config --global user.name    # 应有值
git config --global user.email   # 应有值
# 缺身份时补上：
git config --global user.name "your-github-username"
git config --global user.email "you@example.com"
```

本机（dmt 服务器）已就绪：gh 认证 dmtang-jpg、`credential.helper=store`、SSH 密钥齐备，通常可直接用。

## 1. 最快速路径：gh 一键建仓 + 推送（推荐）

```bash
cd /path/to/project
git init -b main                       # 默认分支直接叫 main，避免 master/main 不匹配
git add .
git commit -m "Initial commit"
gh repo create <repo-name> --public --source . --push
# --public / --private 二选一；--push 自动 add remote 并 push
```

完成后远程已建好、本地 origin 已指过去，后续直接 `git push` 即可。

## 2. 标准手动流程（无 gh 或需要精细控制）

```bash
cd /path/to/project
git init -b main
git add .
git commit -m "Initial commit"

# 2a. 在 GitHub 网页建空仓库（不要勾选 README/license/.gitignore，否则要先 pull）
# 2b. 或用 API 建仓（无 gh 时）：
# curl -X POST -H "Authorization: token *** \
#   https://api.github.com/user/repos \
#   -d '{"name":"<repo-name>","private":false}'

git remote add origin https://github.com/<owner>/<repo-name>.git
git push -u origin main                # -u 记住上游，之后直接 git push
```

## 3. 日常更新（已有仓库）

```bash
git add -A
git commit -m "描述本次改动"
git push                               # 已设上游后无需参数
```

推送前自查：`git status` 干净、无 secrets、无大文件。

## 4. 验证推送成功

```bash
git ls-remote origin                    # 能列出 refs = 认证 + 连通 OK
gh repo view <owner>/<repo-name>        # 查看仓库信息
curl -s https://api.github.com/repos/<owner>/<repo-name> | jq .full_name
```

## Common Pitfalls

1. **master vs main 分支不匹配**：本地 master、远程 main（或反之）→ `git branch -M main` 后重新 push。新仓一律 `git init -b main`。
2. **non-fast-forward / failed to push some refs**：远程已有提交（如建仓时勾了 README）→ 先 `git pull --rebase origin main` 再 push。不要轻易 `--force`（会覆盖他人提交，危险）。
3. **认证失败 / 要求输密码**：GitHub 已禁用密码认证。用 `gh auth setup-git` 或 `credential.helper store` 存 token；token 必须有 `repo` scope。
4. **token 泄露**：不要把 token 写进 `git remote set-url origin https://user:token@...` 并 commit 到公开仓。一旦 commit 里出现密钥 → 视为已泄露，去 GitHub 撤销重建 token。
5. **大文件被拒（>100MB）**：GitHub 拒绝单文件 >100MB（提醒阈值 50MB）。用 `git lfs` 跟踪大文件，或从历史移除（`git filter-repo`）。至少先加 .gitignore 把 data/、模型权重挡在外面。
6. **误提交垃圾文件**：提交前检查 `git status`；.gitignore 至少包含 `__pycache__/`、`*.pyc`、`.env`、`venv/`、`node_modules/`、`dist/`、`*.log`。
7. **代理导致 push 超时/失败**：需要代理时 `git config --global http.proxy http://<proxy>:<port>`；不需要时 `unset http_proxy https_proxy all_proxy`（注意环境变量里残留的代理会干扰）。
8. **"remote origin already exists"**：用 `git remote set-url origin <新url>` 修改，而不是重复 add。
9. **Author identity unknown 报错**：`git config user.name / user.email` 必须在 commit 前设置。
10. **push 卡住不动**：多半是大文件或网络；先 `git count-objects -vH` 看仓库体积，超 1GB 考虑瘦身。

## Verification Checklist

- [ ] `git status` 干净（没有未提交改动）
- [ ] `git push` 无报错，输出 `main -> main`
- [ ] `git ls-remote origin` 能看到最新 commit 的 ref
- [ ] `gh repo view` 或网页能看到文件
- [ ] 没有 secrets、大文件、垃圾文件进仓库
- [ ] 公开仓库确认 `.env`/密钥类文件已被 .gitignore 排除

## One-Shot Recipes

- **新建项目直接上 GitHub**：
  ```bash
  git init -b main && git add . && git commit -m "Initial commit" && gh repo create my-proj --public --source . --push
  ```
- **已有目录推到自己账号新私有仓**：
  ```bash
  cd dir && git init -b main && git add . && git commit -m "Initial commit" && gh repo create <name> --private --source . --push
  ```
- **修复"push 被拒"**：
  ```bash
  git branch -M main && git pull --rebase origin main && git push -u origin main
  ```
