---
name: local-vlm-vision
description: 调用本地部署的 Qwen2.5-VL-3B 轻量看图模型（10.218.208.65:8091）识别图片文字、弹窗、报错框、描述图片内容。OpenAI 兼容 API，无需 GPU，CPU 即可运行。弹窗/报错识别比 tesseract 更准（保留语义），也是 OCR 方案的补充。
---

# 本地 VLM 看图辅助模型 (Qwen2.5-VL-3B)

本机（树上的AI，10.218.208.65）已部署轻量视觉语言模型 **Qwen2.5-VL-3B-Instruct**（GGUF Q4_K_M，llama.cpp 运行），提供 OpenAI 兼容 API。

## 服务信息

- **地址**: `http://10.218.208.65:8091/v1/chat/completions`
- **模型名**: `qwen2.5-vl-3b`（任意值均可）
- **状态**: 开机自启（crontab @reboot），随时可用
- **硬件**: 无 GPU，纯 CPU（32 核），单图识别 1~5 秒

## 一键使用（推荐）

```bash
# 识别图片中的所有文字（弹窗/报错/截图）
python3 /home/dmt/workspace/tools/vlm_ask.py 图片.png

# 指定问题（描述内容/识别按钮等）
python3 /home/dmt/workspace/tools/vlm_ask.py 图片.png "这张图里有什么按钮？"

# Linux 桌面截屏后识别
python3 /home/dmt/workspace/tools/vlm_ask.py --screen "识别弹窗内容"
```

## API 调用（curl 示例）

```bash
B64=$(base64 -w0 图片.png)
curl -s http://10.218.208.65:8091/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"qwen2.5-vl-3b\",\"messages\":[{\"role\":\"user\",\"content\":[
    {\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/png;base64,$B64\"}},
    {\"type\":\"text\",\"text\":\"识别图中所有文字，逐行输出\"}
  ]}],\"max_tokens\":500}"
```

## Python 调用（注意代理坑！）

⚠️ **必须绕代理**：本机/局域网存在 `HTTP_PROXY` 环境变量（如 10.218.208.236:1088），会把 127.0.0.1/内网请求劫持到代理导致断连。用 `requests.Session()` 并设 `s.trust_env = False`：

```python
import base64, requests
s = requests.Session()
s.trust_env = False  # 关键！否则 ProxyError / RemoteDisconnected
b64 = base64.b64encode(open('图片.png','rb').read()).decode()
r = s.post('http://10.218.208.65:8091/v1/chat/completions', json={
    'model': 'qwen2.5-vl-3b',
    'messages': [{'role': 'user', 'content': [
        {'type': 'image_url', 'image_url': {'url': f'data:image/png;base64,{b64}'}},
        {'type': 'text', 'text': '识别图中所有文字'}
    ]}],
    'max_tokens': 500
}, timeout=120)
print(r.json()['choices'][0]['message']['content'])
```

## 实测效果（2026-08-01）

| 场景 | 结果 |
|------|------|
| 报错弹窗（中文+错误码） | 「错误/无法连接到测量设备/请检查 USB 连接后重试/错误码: 0x8007045D/确定」100% 准确，0.8s |
| 软件更新界面（12 项文字+按钮） | 全部文字+按钮理解正确，4.9s |
| 自然场景图（山/树/太阳） | 描述准确，3.2s |

## 与 tesseract 的分工

- **tesseract (ocr_dialog.py)**：纯文字提取，离线、轻量、快（<1s）
- **VLM (vlm_ask.py)**：语义理解 + 识别，弹窗/报错/复杂界面更准，能回答"这是什么界面"

## 重新部署（万一服务挂了）

```bash
# 检查状态
curl -s http://127.0.0.1:8091/health   # 期望 {"status":"ok"}
# 重启服务
/home/dmt/models/vlm/start_vlm_server.sh 8091
```

模型文件：`~/models/vlm/qwen2.5-vl-3b-q4_k_m.gguf` (1.9G) + `mmproj-qwen2.5-vl-3b-f16.gguf` (1.3G)
服务二进制：`~/llama.cpp/build/bin/llama-server`
