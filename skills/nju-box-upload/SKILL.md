---
name: nju-box-upload
description: "Use when uploading files to NJU Box cloud drive (box.nju.edu.cn): file upload, share link creation, auth. 上传文件到南大云盘的完整流程（含 JWT 认证、代理绕过、分享链接）。"
version: 1.1.0
author: dmtang-jpg (Hermes Agent 树上的AI)
license: MIT
metadata:
  hermes:
    tags: [box, upload, cloud, NJU, 云盘, 上传, 分享链接]
    related_skills: [nju-box-cloud-drive-setup, github-push]
---

# NJU Box 云盘上传 (Upload to NJU Box)

## Overview

把本地文件上传到南京大学云盘 **box.nju.edu.cn**（Seafile Pro），并生成可分享的下载链接。**认证用 JWT token（可靠），上传走 upload-link API，分享用 PUT shared-link**。这是经过真实测试验证的完整流程（2026-08-01 验证通过）。

## When to Use

- 用户说"上传到 box / 云盘 / 生成下载链接 / 发个文件链接"
- 需要把交付文件（docx、pdf、tar.gz）分享给用户或其他人
- 飞书无法直接发文档附件时的替代方案（bot 无 drive:file:upload 权限）

## 前置条件

- Python 3 + `requests`（`pip install requests`）
- 账号密码：默认 `0410037` / 环境变量 `BOX_USER`、`BOX_PASS` 可覆盖
- 仓库 ID：`26fa0b5f-a7e0-429f-9d7f-8ecda8ef1a66`（环境变量 `BOX_REPO_ID` 可覆盖）
- ⚠️ **必须绕过系统代理**：box 在校内网络，`https_proxy="" http_proxy=""` 或 `trust_env=False`

## 完整流程（已验证，2026-08-01）

```python
import requests, urllib3
urllib3.disable_warnings()
import os
# 关键：绕过代理（NJU Box 在校内直连）
for k in ('HTTP_PROXY','HTTPS_PROXY','http_proxy','https_proxy'):
    os.environ.pop(k, None)

BASE = 'https://box.nju.edu.cn'
REPO_ID = os.environ.get('BOX_REPO_ID', '26fa0b5f-a7e0-429f-9d7f-8ecda8ef1a66')

s = requests.Session()
s.verify = False
s.trust_env = False  # 绕过系统代理

# Step 1: JWT 认证（可靠，不需要 CSRF）
r = s.post(f'{BASE}/api2/auth-token/',
           json={'username': '0410037', 'password': 'njuee366'}, timeout=20)
r.raise_for_status()
token = r.json()['token']
s.headers['Authorization'] = f'Token {token}'

# Step 2: 获取上传链接（⚠️ 必须 GET + p 参数，返回纯文本 URL）
resp = s.get(f'{BASE}/api2/repos/{REPO_ID}/upload-link/', params={'p': '/'}, timeout=20)
resp.raise_for_status()
upload_url = resp.text.strip().strip('"')  # 如 https://box.nju.edu.cn/seafhttp/upload-api/{token}

# Step 3: 上传文件（POST upload_url?ret-json=1 + parent_dir + filename）
with open(local_file_path, 'rb') as f:
    up = s.post(f'{upload_url}?ret-json=1',
                files={'file': (filename, f, 'application/octet-stream')},
                data={'parent_dir': '/'}, timeout=120)
if up.status_code not in (200, 201):
    raise RuntimeError(f'Upload failed: {up.status_code} {up.text[:200]}')
print('Upload OK:', up.json())

# Step 4: 创建分享链接（⚠️ 必须 PUT + JSON body p，201 → Location header）
share = s.put(f'{BASE}/api2/repos/{REPO_ID}/file/shared-link/',
              json={'p': f'/{filename}'},
              headers={'Content-Type': 'application/json'}, timeout=20)
if share.status_code == 201:
    link = share.headers.get('Location', '')
    print('LINK:', link)  # 如 https://box.nju.edu.cn/f/{token}/
else:
    # 兜底：直接下载 URL
    dl = s.get(f'{BASE}/api2/repos/{REPO_ID}/file/', params={'p': f'/{filename}'}, timeout=20)
    link = dl.text.strip().strip('"')
    print('LINK:', link)
```

## 命令行快速版（已有工具脚本时）

```bash
# 上传（需先有 tools/nju_box_api.py）
https_proxy="" http_proxy="" python3 /home/dmt/workspace/tools/nju_box_api.py upload /local/file.docx /目标目录/file.docx
```

## Common Pitfalls（血泪教训）

1. **upload-link 必须是 GET**：`POST /api2/repos/{id}/upload-link/` 返回 **405 Method Not Allowed**（2026-08-01 真实踩坑：chat/app.py 因此上传全挂）。正确：`GET ...?p=/` 返回**纯文本 URL**（带引号，需 `.strip('"')`）。
2. **不能用 `resp.json().get("upload_url")` 解析**：响应是纯文本 URL 不是 JSON dict。
3. **必须绕过代理**：系统 HTTPS 代理（114.212.x.x:8888 或 Day8 VPN 的 fake-ip）会劫持 DNS/阻断连接。`os.environ.pop('HTTP_PROXY'...)` + `s.trust_env = False` 缺一不可。
4. **JWT 优先于 Django session**：box 登录页改版后 `/accounts/login/` 的 CSRF 解析会失败（`AttributeError: 'NoneType' has no attribute 'group'`）。JWT `/api2/auth-token/` 不需要 CSRF，最可靠。
5. **分享链接是 PUT 不是 POST**：`POST /api/v2.1/share-links/` 返回 500 或 `permissions invalid`。正确：`PUT /api2/repos/{repo}/file/shared-link/` + JSON `{'p': '/path'}`，201 时链接在 **Location header**（body 为空）。
6. **上传到子目录**：`upload-link/?p=/subdir` 拿链接后，POST 时 `parent_dir='/subdir'` + `filename='just_name.ext'`（不是全路径，否则落根目录）。
7. **假失败信号**：`curl -s -o /dev/null -w '%{http_code}'` 返回 400 可能是 body 转义问题（PowerShell 引号），不代表网络不通。用 Python requests 实测认证最准。
8. **环境变量 BOX_REPO_ID 门控陷阱**：代码里 `if os.environ.get("BOX_REPO_ID"):` 会因环境变量未设置而**永远跳过上传**。去掉门控或提供默认值。

## Verification Checklist

- [ ] 认证返回 200 且有 token
- [ ] GET upload-link 返回 URL（不是 405）
- [ ] 上传响应 `[{"id": "...", "name": "...", "size": N}]` 200
- [ ] 分享链接 HTTP 200 可访问
- [ ] 测试完删除测试文件（DELETE /api2/repos/{id}/file/?p=/name）

## One-Shot Recipes

- **最小上传**（文件在根目录）：
  ```python
  s.get(f'{BASE}/api2/repos/{REPO_ID}/upload-link/', params={'p': '/'})
  s.post(upload_url + '?ret-json=1', files={'file': (name, f)}, data={'parent_dir': '/'})
  ```
- **上传并立即分享**：Step 1-4 连跑，返回 `https://box.nju.edu.cn/f/{token}/`
