# WSL 7×24 长活方案（夜王实证版）

> 适用：WSL2 发行版内服务（hermes-gateway / ssh / tunnel 等）周期性被杀，或需要 WSL 7×24 常驻。
> 实证：夜王（DESKTOP-4ODUEMT，WSL 2.7.3.0，无 `.wslconfig`）init 时间戳 7 天+ 不变，4 个常驻 wsl.exe 进程，稳定不掉线。
> 取证日期：2026-08-06

---

## 1. 核心原理（最重要的一条）

**WSL2 回收判断依据是「是否还有 wsl.exe 会话」，不是「发行版内是否有进程在跑」。**

- 发行版内 systemd 服务再多，最后一个 wsl.exe 会话退出后，空闲超过 `vmIdleTimeout`（默认 60000ms = 60 秒）→ Windows 回收**整个发行版**（VM 整体销毁）→ 里面所有进程 SIGTERM。
- **保活核心 = 让一个 wsl.exe 会话永不退出**，发行版永不进入空闲态。
- 不依赖 `.wslconfig` 配置——夜王无配置也稳定 7 天+。

## 2. 症状识别

| 症状 | 说明 |
|------|------|
| init 时间戳周期性变化 | `ps -p 1 -o lstart` 时间在变 = 整个发行版被回收（不是服务自身问题） |
| dmesg 被杀前 ~2 分钟 | `WSL (1) ERROR: Broken pipe @SocketChannel.h` / `Operation canceled @p9io.cpp` |
| `wsl -l -v` 显示 Stopped | 发行版已被回收 |
| 被杀时刻有规律 | = 每次保活会话结束 + ~60s（vmIdleTimeout），分毫不差 |

## 3. 三层任务架构（本机实际状态）

| 层 | 任务名 | 触发 | 动作 |
|----|--------|------|------|
| 1 | `WSL Auto Start` | BootTrigger（开机） | `wsl.exe -d Ubuntu -- /home/dmt/.hermes/scripts/wsl-keepalive.sh`（`exec sleep infinity` 常驻会话） |
| 2 | `Hermes WSL AutoStart` | LogonTrigger（登录） | `wscript.exe start_wsl_silent.vbs` → `start_wsl.bat`（`wsl.exe -d Ubuntu -- sleep infinity`，兜底） |
| 3 | `WSL_Hermes_Keepalive` | BootTrigger（开机，StartBoundary 2026-07-29T01:01:00 为历史遗留） | `wscript.exe wsl_keepalive.vbs`（分离式播种 `while true; do sleep 3600`，自愈网） |

三个任务均 RunAs `dontang32` / Interactive（非 SYSTEM，避免 Profile 错位读不到 `.wslconfig` 的问题）。

## 4. 脚本清单（全部原文）

### 4.1 `/home/dmt/.hermes/scripts/wsl-keepalive.sh`（第 1 层调用）

```bash
#!/bin/bash
# WSL keep-alive: prevents WSL from shutting down when no interactive sessions exist
# Started by Windows scheduled task at boot
echo "[$(date)] WSL keep-alive started"
exec sleep infinity
```

### 4.2 `C:\Users\dontang32\scripts\start_wsl_silent.vbs`（第 2 层调用）

```vbs
' VBScript to run WSL silently (no visible window)
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run "cmd /c C:\Users\dontang32\scripts\start_wsl.bat", 0, False
```

### 4.3 `C:\Users\dontang32\scripts\start_wsl.bat`（第 2 层，被 VBS 调用）

```bat
@echo off
REM Start WSL in background to keep systemd alive (hermes-gateway auto-starts via systemd)
wsl.exe -d Ubuntu -- sleep infinity
```

### 4.4 `C:\scripts\wsl_keepalive.vbs`（第 3 层，分离式播种）

```vbs
Set objShell = CreateObject("WScript.Shell")
' 保持 WSL 存活 — 每小时 ping 一次
objShell.Run "wsl.exe -d Ubuntu --exec /bin/sh -c ""while true; do sleep 3600; done""", 0, False
```

**分离式 VBS 要点**：`Run` 第 2 参数 `0` = 隐藏窗口；第 3 参数 `False` = **不等待** → 任务实例立即完成，任务永远 Ready。配 Boot 触发器 = 自愈网（旧会话死掉后，新播种的 wsl.exe 会重新拉起整个发行版）。

## 5. 部署步骤（新机器照做）

1. **放置脚本**
   - `wsl-keepalive.sh` → WSL 内 `~/.hermes/scripts/`（`chmod +x`）
   - `start_wsl_silent.vbs` + `start_wsl.bat` → `C:\Users\<用户名>\scripts\`
   - `wsl_keepalive.vbs` → `C:\scripts\`（任意路径，但任务里要写对）

2. **创建计划任务**（管理员 PowerShell）：

```powershell
# 第 1 层：开机拉起 + 常驻
schtasks /create /tn "WSL Auto Start" /tr "wsl.exe -d Ubuntu -- /home/dmt/.hermes/scripts/wsl-keepalive.sh" /sc onstart /ru <用户名> /it /f

# 第 2 层：登录兜底
schtasks /create /tn "Hermes WSL AutoStart" /tr "wscript.exe C:\Users\<用户名>\scripts\start_wsl_silent.vbs" /sc onlogon /ru <用户名> /it /f

# 第 3 层：自愈播种
schtasks /create /tn "WSL_Hermes_Keepalive" /tr "wscript.exe C:\scripts\wsl_keepalive.vbs" /sc onstart /ru <用户名> /it /f
```

> 注意：任务必须 RunAs **登录用户**（`/it` = Interactive），不要用 SYSTEM。

3. **重启或手动触发验证**：

```powershell
schtasks /run /tn "WSL Auto Start"
schtasks /run /tn "Hermes WSL AutoStart"
schtasks /run /tn "WSL_Hermes_Keepalive"
```

## 6. 验证方法（取证三板斧）

```bash
# 1. init 时间戳 —— 隔几天再看，不变 = 发行版从未被回收
ps -p 1 -o lstart --no-headers

# 2. 常驻 wsl.exe 进程 —— 应有多个常驻进程
powershell.exe Get-Process wsl | Select Id,StartTime

# 3. 计划任务状态
powershell.exe Get-ScheduledTask -TaskName 'WSL*','Hermes*' | fl TaskName,State

# 4. 发行版状态
wsl.exe -l -v
```

夜王参照（2026-08-06 取证）：
- 4 个常驻 wsl.exe 进程（07-29/07-30 启动至今）
- init 时间戳 `Thu Jul 30 19:52:34 2026`，7 天未变
- 3 个任务全部 Ready

## 7. 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 发行版还是周期被杀 | 任务跑在 SYSTEM 下 → `.wslconfig` 读不到 | 任务 RunAs 登录用户 + `/it` |
| 改了 `.wslconfig` 无效 | 只在 VM 创建时读取 | `wsl --shutdown` 后重启 WSL |
| 任务显示 Running 不结束 | 任务里直接 `while true` 阻塞 | 改用分离式 VBS（不等待） |
| Windows 更新重启后没自启 | 只有时间触发器的任务 | 必须配 BootTrigger/LogonTrigger |
| 失去「20 分钟隐形重启」兜底 | 保活成功后 gateway 崩溃没人重启 | systemd 单元配 `Restart=always` + `RestartSec=5` |

## 8. vmIdleTimeout 配置（可选双保险）

`.wslconfig` 位置：`C:\Users\<用户>\.wslconfig`（Windows 侧，非 WSL 内路径）

```ini
[wsl2]
vmIdleTimeout=60000
[experimental]
autoMemoryReclaim=gradual
sparseVhd=true
```

排查顺序：① 任务 RunAs 用户 Profile 是否对 → ② 改后是否 `wsl --shutdown` → ③ 文件格式（UTF-8 无 BOM、`[wsl2]` 小写、单位毫秒）。
即使 vmIdleTimeout 失效，会话常驻方案也能长期稳定——配置只是锦上添花。

---

*方案来源：夜王本机实际取证（2026-08-06），非设计稿。配套技能见 `skills/wsl-distro-lifecycle/SKILL.md`。*
