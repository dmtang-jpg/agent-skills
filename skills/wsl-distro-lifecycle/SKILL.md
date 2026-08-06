---
name: wsl-distro-lifecycle
description: Use when WSL 内服务周期性被杀或需 7×24 常驻. 空闲回收诊断与保活方案。
tags: [wsl, keepalive, lifecycle, idle-reclamation, systemd, windows]
---

# WSL Distro Lifecycle（空闲回收 & 保活）

WSL2 发行版被 Windows 自动回收的机制、诊断方法，以及经过实证（夜王 7 天+ 稳定）的保活方案。

## 核心认知（最重要的一条）

**WSL2 回收判断依据是"是否还有 wsl.exe 会话"，不是"发行版内是否有进程在跑"。**
发行版内 systemd 服务再多，最后一个 wsl.exe 会话退出后，空闲超过 vmIdleTimeout（默认 60000ms = 60s）→ Windows 回收**整个发行版**（VM 整体销毁）→ 里面所有进程 SIGTERM。

**保活核心 = 让一个 wsl.exe 会话永不退出**，发行版永不进入空闲态。不依赖任何 .wslconfig 配置。

## 触发场景

- WSL 内服务（hermes-gateway 等）周期性被杀，用户感知"网关又断"
- 整个发行版周期性重启（`ps -p 1 -o lstart` init 时间戳在变）
- 需要让 WSL 7×24 常驻（gateway / ssh / tunnel）

## 症状识别

- init 时间戳周期性变化（整个发行版被回收，不是服务自身问题）
- dmesg 被杀前 ~2 分钟: `WSL (1) ERROR: Broken pipe @SocketChannel.h` / `Operation canceled @p9io.cpp`
- `wsl -l -v` 显示 `Stopped`
- 被杀时刻 = 每次保活会话结束 + ~60s（vmIdleTimeout 默认），分毫不差

## 诊断三板斧

1. `ps -p 1 -o lstart --no-headers` — 周期性变化 = 整发行版被回收
2. 找**外部周期事件源**并对齐时间戳：gateway 被杀日志时间戳规律 vs Windows 计划任务**实际**执行周期（计划任务"任务正在运行时跳过新触发"会让实际周期 ≠ 计划周期，sleep 900 阻塞 + 5min 触发 = 20min 被杀周期）
3. 先分清"服务挂了"和"发行版被回收"：`systemctl is-active <svc>` + init 时间戳一起看

## 保活方案（夜王实证：WSL 2.7.3.0，无 .wslconfig，7 天+ 不掉线）

### 三层任务架构（夜王实际取证 2026-08-06，全 RunAs dontang32 / Interactive）

| 层 | 任务名 | 触发 | 动作 |
|---|---|---|---|
| 1 | `WSL Auto Start` | BootTrigger | `wsl.exe -d Ubuntu -- /home/dmt/.hermes/scripts/wsl-keepalive.sh`（`exec sleep infinity` 常驻会话） |
| 2 | `Hermes WSL AutoStart` | LogonTrigger | `wscript.exe C:\Users\dontang32\scripts\start_wsl_silent.vbs` → `start_wsl.bat`（`wsl.exe -d Ubuntu -- sleep infinity`，兜底） |
| 3 | `WSL_Hermes_Keepalive` | BootTrigger（StartBoundary 2026-07-29T01:01:00 为历史遗留，非每日） | `wscript.exe C:\scripts\wsl_keepalive.vbs`（分离式播种 `while true; do sleep 3600`，自愈网） |

取证命令：`Get-ScheduledTask -TaskName 'WSL*','Hermes*' | % { $_.Actions; $_.Triggers; $_.Principal }`

### 分离式 VBS 写法（任务实例立即完成，wsl.exe 分离常驻）

```vbs
Set objShell = CreateObject("WScript.Shell")
objShell.Run "wsl.exe -d Ubuntu --exec /bin/sh -c ""while true; do sleep 3600; done""", 0, False
```

- `Run` 第 2 参数 `0` = 隐藏窗口；第 3 参数 `False` = **不等待** → 任务实例立即结束，任务永远 Ready
- 配每日触发器 = 自愈网（旧会话死掉后，新播种的 wsl.exe 会重新拉起整个发行版）
- 替代：保活脚本内容 `exec sleep infinity`，由 BootTrigger 任务调用

### 与"任务内无限循环"的对比

- 任务内 `while true` 阻塞也能保活，但任务实例永远 Running（Task Scheduler 界面误导）；wsl.exe 死后靠下次 5min 触发自愈（恢复 ≈ 5min + 60s + 启动）
- 分离式播种更干净：任务永远 Ready，每日播种天然自愈

## vmIdleTimeout 配置为何失效（排查顺序）

1. **用户 Profile 错位（最大嫌疑）**：`.wslconfig` 只对"启动 wsl.exe 的用户"的 `%UserProfile%` 生效。计划任务若以 SYSTEM 运行 → 读 `C:\Windows\System32\config\systemprofile\` → 无配置 → 永远默认 60s。查：`schtasks /query /tn <task> /v | findstr "Run As User"`
2. 改配置后没 `wsl --shutdown` — .wslconfig 只在 VM 创建时读取
3. 文件问题：`[wsl2]` 段必须小写精确、单位毫秒、UTF-8 无 BOM、路径必须是 Windows 侧 `C:\Users\<用户>\.wslconfig`（不能放 WSL 内部 Linux 路径）
4. 受控实验（决定性）：`wsl --shutdown` → `wsl.exe -d Ubuntu -e true`（立即退出）→ 观察 `wsl -l -v` 是否 ~60s 后变 Stopped
5. 结论：即使 vmIdleTimeout 失效，会话常驻方案也能长期稳定 — 配置只是锦上添花的双保险

## 附加陷阱

- **gateway 自身崩溃自愈**：VM 不再周期重启后，失去"每 20 分钟隐形重启兜底" → systemd 单元需 `Restart=always` + `RestartSec=5`，或加探活看门狗（每 5 分钟 curl /health，挂了 restart）
- **维护窗口**：`schtasks /end` 或改脚本会断保活 → 60s 后发行版被回收 → 操作前先手动起一个保活会话顶住，或接受短暂中断
- **Windows 更新重启后自启**：只有时间触发器的任务重启后要等下次计划时间 → 必须配 BootTrigger/LogonTrigger（第 1/2 层）
- 资源占用：`while true; do sleep 3600` / `sleep infinity` 开销可忽略；VM 永不休眠 = 内存常驻，内存紧张可加 `[experimental] autoMemoryReclaim=gradual` + `sparseVhd=true`（与保活机制独立）

## 案例速记

- **Forest（2026-08-06）**：`WslKeepAlive` 任务 `sleep 900` → 会话结束 + 60s → 整发行版回收 → gateway 每 20 分钟被杀。修复 = 改无限循环保活 + 建议补 BootTrigger/LogonTrigger/每日播种三层架构（照抄夜王）。同版本 WSL 2.7.3.0，夜王无 .wslconfig 实证稳定 → 配置非必需。
- 排查此类问题时，可先查夜王本机作为"成功参照"：`Get-Process wsl`（应有多个常驻进程）、`Get-ScheduledTask -TaskName WSL*`（BootTrigger/LogonTrigger/每日播种三层）。
