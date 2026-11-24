# 邦多利演出助手 · Agent 环境适配与故障自查手册

本手册面向在用户本机协助排查问题的 AI Agent（或技术支持人员），用于快速定位「无法启动、连不上设备、画面不适配、无法识别房间/曲目、打歌偏移/漏键/断连、结算卡住、联网失败」等环境差异问题。

---

## 0. Agent 执行原则（必读）

1. **先只读采集，再下结论**：优先读取版本信息、日志文件、`settings.json`、ADB 设备状态和模拟器分辨率，拿到直接证据后再给修复方案。
2. **禁止篡改程序运行库**：不要修改、替换或跨版本混拷 `versions/<版本>/_internal` 中的文件；`vendor/adb` 与 `scrcpy-server-v3.3.1` 每次启动都会按 `来源与SHA256.json` 校验哈希，任何修改都会直接触发安全拦截。
3. **禁止未经确认触发游戏内消耗操作**：排查期间不要点击游戏内的星石消费、道具使用、Live Boost 确认回复或抽卡界面；不要用 `adb shell input` 随意点击未知画面。
4. **修改配置前先备份**：若需要调整 `本地数据/settings.json`，先复制一份 `settings.json.bak`，并且确保程序已退出，避免界面写回覆盖。
5. **敏感信息脱敏**：整理排查报告发给他人时，隐藏设备序列号、个人目录用户名与局域网地址。

---

## 1. 目录结构与诊断文件地图

假设程序解压在 `<安装根目录>`（例如 `D:\BandoriHelper`）：

```text
<安装根目录>\
├── 启动邦多利.cmd                 # 标准入口（通过 launch-friend.ps1 读取 active-version.json）
├── launch-friend.ps1             # 设置 BANDORI_INSTALL_ROOT / BANDORI_DATA_ROOT 并拉起当前版本
├── active-version.json           # 当前生效版本号与 exe 相对路径
├── update-source.json            # GitHub 更新源配置
├── versions\<版本号>\
│   ├── 邦多利自用版.exe           # 主程序（PyInstaller 冻结版，无需安装 Python）
│   ├── release-manifest.json     # 版本清单
│   └── _internal\
│       ├── vendor\adb\adb.exe    # 随包官方 ADB（带 来源与SHA256.json 校验）
│       ├── vendor\scrcpy-server-v3.3.1
│       ├── vendor\minitouch\     # 各 ABI 的 minitouch 二进制
│       ├── vendor\ssm\ssm-helper.exe
│       └── vendor\node\          # Bestdori 计算资源运行环境
└── 本地数据\                      # 所有版本共享的用户数据与诊断日志
    ├── settings.json             # 轨道校准、延迟、触控后端、设备与模式配置
    ├── 运行日志\
    │   ├── Qt窗口启动-*.json      # 每次启动的就绪耗时与设备连接状态
    │   └── YYYYMMDD-HHMMSS.log   # 界面实时运行日志（设备、下载、报错汇总）
    ├── 演出记录\<时间戳>\         # 协力/车队/单曲/对邦/组曲/5v5/挖矿的单次记录
    ├── 活动演出\<时间戳>\         # 活动挑战演出记录
    ├── 连续清CP\<时间戳>\         # 连续清 CP 记录
    ├── 挑战演出\<时间戳>\         # 挑战演出记录
    ├── 连续全P排练\<时间戳>\      # 排练记录
    │   ├── ui-session.json       # 该局完整参数快照（region/mode/options/serial）
    │   ├── run.log               # 该局详细步骤日志（定位卡点核心文件）
    │   ├── summary.json          # 结束状态与异常原因
    │   ├── 结算超时画面.png       # 结算卡住时自动保存的现场截图
    │   └── 开场-<曲目ID>-<难度>\  # 首拍未触发时保存的 diagnostics.json 与 last-plane.png
    ├── 校时记录\*.json            # 逐音符视觉校时残差（诊断打歌偏早/偏晚核心文件）
    ├── 曲库\                     # Bestdori 公开曲库缓存（all.5.json 与 charts/）
    └── 识别区域\<cn|jp>\          # 可选自定义 OCR 裁剪区域配置
```

---

## 2. 一键只读体检流程（PowerShell）

请先在 `<安装根目录>` 下执行以下只读命令，收集完整基线信息。

### 2.1 检查系统环境、目录权限、当前版本与关键文件

```powershell
$Root = (Get-Location).Path
$Active = Get-Content "$Root\active-version.json" -Raw -Encoding UTF8 | ConvertFrom-Json
$Exe = Join-Path $Root $Active.entrypoint
$VerDir = Split-Path $Exe -Parent
$Internal = Join-Path $VerDir '_internal'

[PSCustomObject]@{
    OS            = (Get-CimInstance Win32_OperatingSystem).Caption
    Arch          = $env:PROCESSOR_ARCHITECTURE
    InstallRoot   = $Root
    ActiveVersion = $Active.version
    ExeExists     = Test-Path $Exe
    AdbExists     = Test-Path "$Internal\vendor\adb\adb.exe"
    ScrcpyExists  = Test-Path "$Internal\vendor\scrcpy-server-v3.3.1"
    SsmExists     = Test-Path "$Internal\vendor\ssm\ssm-helper.exe"
    VendorManifest= Test-Path "$Internal\vendor\来源与SHA256.json"
    DataWritable  = try {
        $probe = Join-Path $Root "本地数据\.write_probe_$PID"
        New-Item -ItemType File -Path $probe -Force | Out-Null
        Remove-Item $probe -Force
        $true
    } catch { $false }
} | Format-List
```

### 2.2 离线运行内置依赖自检（不弹窗、不连设备）

程序内置 `BANDORI_PORTABLE_SELFTEST` 钩子，可直接验证冻结后的 PySide6 / RapidOCR / 日文识别模型 / PyAV (FFmpeg) / `ssm-helper.exe` / Node.js 是否在本机正常加载：

```powershell
$Report = Join-Path $env:TEMP "bandori-selftest-$PID.json"
$env:BANDORI_PORTABLE_SELFTEST = $Report
$proc = Start-Process -FilePath $Exe -WorkingDirectory $VerDir -PassThru -Wait
Remove-Item Env:BANDORI_PORTABLE_SELFTEST -ErrorAction SilentlyContinue
Write-Host "ExitCode=$($proc.ExitCode)"
if (Test-Path $Report) { Get-Content $Report -Raw -Encoding UTF8 }
```

- 预期：`ExitCode=0` 且 JSON 中 `"passed": true`。
- 若失败，直接查看 JSON 中的 `error` 字段，通常是缺系统运行库、杀软拦截 DLL/exe、或解压不完整。

### 2.3 检查模拟器进程、监听端口、ADB 与画面分辨率

```powershell
$Adb = Join-Path $Internal 'vendor\adb\adb.exe'

Write-Host '=== 1. MuMu 相关进程与本机监听端口 ==='
$owners = @(Get-Process -ErrorAction SilentlyContinue | Where-Object {
    $_.ProcessName -match '^(MuMu|Nemu|MuMuVMM)'
})
$owners | Select-Object ProcessName, Id | Format-Table -AutoSize
if ($owners.Count -gt 0) {
    Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
        Where-Object { $_.OwningProcess -in $owners.Id -and $_.LocalAddress -in @('127.0.0.1','0.0.0.0','::','::1') } |
        Select-Object LocalAddress, LocalPort, OwningProcess | Sort-Object LocalPort -Unique | Format-Table -AutoSize
}

Write-Host '=== 2. ADB 已连接设备 ==='
& $Adb devices -l

Write-Host '=== 3. 选中设备的分辨率 / DPI / ABI / 前台应用（将 $Serial 替换为实际序列号）==='
# 示例：$Serial = '127.0.0.1:16384'
if ($Serial) {
    Write-Host "wm size:     $(& $Adb -s $Serial shell wm size)"
    Write-Host "wm density:  $(& $Adb -s $Serial shell wm density)"
    Write-Host "abi:         $(& $Adb -s $Serial shell getprop ro.product.cpu.abi)"
    Write-Host "packages:"
    & $Adb -s $Serial shell pm list packages | Select-String -Pattern 'bilibili\.star|craftegg\.band|bushiroad|bangdream'
    Write-Host "foreground:"
    & $Adb -s $Serial shell dumpsys activity activities | Select-String -Pattern 'mResumedActivity|topResumedActivity|Display #'
}
```

### 2.4 读取当前设置与最近日志

```powershell
$Data = Join-Path $Root '本地数据'

Write-Host '=== settings.json ==='
if (Test-Path "$Data\settings.json") { Get-Content "$Data\settings.json" -Raw -Encoding UTF8 }

Write-Host '=== 最新启动日志 ==='
Get-ChildItem "$Data\运行日志\Qt窗口启动-*.json" -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1 | ForEach-Object {
        Write-Host $_.FullName
        Get-Content $_.FullName -Raw -Encoding UTF8
    }

Write-Host '=== 最新界面运行日志（尾部 60 行）==='
Get-ChildItem "$Data\运行日志\*.log" -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1 | ForEach-Object {
        Write-Host $_.FullName
        Get-Content $_.FullName -Tail 60 -Encoding UTF8
    }

Write-Host '=== 最近一次演出记录 ==='
$latestRun = @('演出记录','活动演出','连续清CP','挑战演出','连续全P排练') | ForEach-Object {
    Get-ChildItem (Join-Path $Data $_) -Directory -ErrorAction SilentlyContinue
} | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestRun) {
    Write-Host "RunDir: $($latestRun.FullName)"
    Get-ChildItem $latestRun.FullName | Select-Object Name, Length | Format-Table -AutoSize
    if (Test-Path "$($latestRun.FullName)\summary.json") {
        Get-Content "$($latestRun.FullName)\summary.json" -Raw -Encoding UTF8
    }
    if (Test-Path "$($latestRun.FullName)\run.log") {
        Get-Content "$($latestRun.FullName)\run.log" -Tail 80 -Encoding UTF8
    }
}
```

---

## 3. 按症状排查决策树（现象 → 证据 → 根因 → 修复）

### A. 无法启动 / 闪退 / 权限与文件异常

| 现象 / 报错关键词 | 必查证据 | 常见根因 | 修复方式 |
| --- | --- | --- | --- |
| 双击 `启动邦多利.cmd` 报 `Active version executable is missing` 或无反应 | `active-version.json`、杀软隔离区、`BANDORI_PORTABLE_SELFTEST` | 只复制了 exe 没带 `_internal`；解压套娃；或 Windows Defender/杀软隔离了 `邦多利自用版.exe` / `ssm-helper.exe` / DLL | 退出旧进程，重新完整解压首装包到独立可写目录（如 `D:\BandoriHelper`），将该目录加入杀软白名单；切勿放进 `C:\Program Files` 等只读目录。 |
| 报 `官方工具校验失败` 或 `找不到官方 adb` | `_internal\vendor\来源与SHA256.json` 及 `vendor\adb\*`、`scrcpy-server-v3.3.1` | 杀软删除/篡改了 `adb.exe` / `AdbWinApi.dll` / `scrcpy-server-v3.3.1`，或手工覆盖了旧版 `_internal` | 重新解压完整版本包，不要从旧版目录复制 `_internal`。 |
| `BANDORI_PORTABLE_SELFTEST` 在 `onnxruntime` / `PySide6` / `av` 报 DLL 加载失败 | 自检 JSON 中的 `error` 字段、系统架构 | 系统为 32 位 Windows、或缺少 Microsoft Visual C++ 2015–2022 x64 运行库 | 确认系统为 Windows 10/11 64 位（x64），安装官方 VC++ 2015–2022 Redistributable (x64) 后重测自检。 |
| 界面文字极小、错位或按钮被裁切 | Windows 显示缩放比例与主屏分辨率 | 笔记本 125%–200% 缩放或极低逻辑高度屏幕导致窗口空间紧张 | 将窗口最大化或把显示器缩放设为标准比例；如仍裁切，记录屏幕物理分辨率与缩放比反馈给开发者。 |

---

### B. 搜不到模拟器 / ADB 连不上 / 触控后端报错

| 现象 / 报错关键词 | 必查证据 | 常见根因 | 修复方式 |
| --- | --- | --- | --- |
| 选「MuMu 模拟器」扫描为空，或报 `此地址没有正在监听的 MuMu 实例` | 第 2.3 节进程与监听端口输出 | 1. 模拟器未启动或未开启 ADB 调试；<br>2. MuMu 端口不在 `7555` 或 `16384 + 32*n`（16384–17408）；<br>3. 使用的是非 MuMu 模拟器（雷电/蓝叠/夜神等），进程名不匹配 `^(MuMu\|Nemu\|MuMuVMM)`；<br>4. 系统策略禁用了 `Get-NetTCPConnection` / PowerShell 查询。 | 1. 确认 MuMu 已开启 ADB 调试，推荐端口 `16384`；<br>2. 非 MuMu 模拟器或真机请切换到「安卓手机」来源（注意：手机/通用来源仅支持单曲入口，自动循环模式按 MuMu 16:9 验收）；<br>3. 多开实例请先扫描并选中具体的 `127.0.0.1:<端口>`。 |
| `adb devices` 显示 `unauthorized` 或 `offline`，或反复断连 | `& $Adb devices -l`、任务管理器中的其他 `adb.exe` | 电脑上其他模拟器/手机助手自带了不同版本的 `adb.exe`，与随包 ADB 抢占 `5037` 端口；或真机未在手机端点击「允许 USB 调试」。 | 关闭冲突的手机助手/旧 ADB 进程，必要时执行 `& $Adb kill-server` 再重连；真机需解锁屏幕并勾选「始终允许」。 |
| 报 `scrcpy控制连接失败：Android调试或控制权限不足` | 手机开发者选项 | 小米/OPPO/vivo/华为等真机仅开了「USB 调试」，未开启「USB 调试（安全设置）」（允许模拟点击）。 | 在手机开发者选项中同时开启「USB 调试（安全设置）」/「禁止权限监控」，重新插拔 USB。 |
| 报 `scrcpy控制连接失败：设备视频编码不可用` 或 `等待新视频帧超时` | 模拟器显卡渲染模式、画面分辨率 | 模拟器渲染模式异常（如纯软件渲染导致 H.264 编码失败）或画面长时间完全静止/黑屏。 | 在 MuMu 设置中切换渲染模式（优先 DirectX / Vulkan），关闭异常帧率插件后重启模拟器再测。 |
| 报 `MuMu SDK未通过已审阅官方SHA256` 或 `minitouch不可用` | 界面运行日志中的 `touch_backend_reason` | 1. MuMu SDK 仅允许白名单内的官方 `external_renderer_ipc.dll` 哈希，MuMu 升级新版 DLL 后会拒绝加载；<br>2. `minitouch` 要求 `dumpsys input` 的触摸轴满足严格 AOSP 仿射映射且无多屏歧义。 | **直接保持默认 `scrcpy` 触控后端即可正常使用**；不要手工替换 DLL。若确实需要适配新版本 MuMu SDK，记录本机 `external_renderer_ipc.dll` 的路径、MuMu 版本号与 SHA256 供后续审核更新。 |

---

### C. 画面不适配 / 无法识别按钮 / 区服与包名不匹配

| 现象 / 报错关键词 | 必查证据 | 常见根因 | 修复方式 |
| --- | --- | --- | --- |
| 报 `需要已校准的 16:9 游戏画面`、`公开选房布局不是已核验的国服16:9` 或按钮完全识别不到 | `wm size` 输出、最新截图比例、`settings.json` 中的 `content_box` | 模拟器分辨率不是 16:9 横屏（例如平板 4:3、带鱼屏 21:9、竖屏锁定、或带不对称黑边/刘海适配）。 | **强烈推荐将模拟器分辨率设为 `1280×720`（横屏，16:9，DPI 240）**，关闭异形屏/黑边留白，再在「设备」页重新截图确认画面铺满无黑边。 |
| 报 `所选区服尚未安装` 或刚开演立刻因前台检测停止 | `pm list packages` 与 `dumpsys activity activities` 输出、`settings.json` 中的 `region` | 1. 助手只识别官服包名：国服 `com.bilibili.star.bili`、日服 `jp.co.craftegg.band`；**渠道服（华为/小米/OPPO/vivo/4399 等）或台服/国际服包名不同**；<br>2. 选错区服（安装了国服但助手选了 `jp`）；<br>3. 模拟器开启了分屏、小窗或悬浮窗，导致 `Display #0` 前台判定不唯一。 | 1. 核对助手区服设置与实际安装一致；<br>2. 若是渠道服/外服，属于当前未适配包名，直接向用户说明并在报告中附上实际包名；<br>3. 关闭分屏/悬浮窗，确保游戏独占前台全屏。 |
| 识别房间/曲名频繁失败、误识别或活动弹窗点不掉 | `run.log`、`结算超时画面.png`、游戏画质与系统语言 | 1. 游戏内非简体中文（国服）或画面缩放模糊/分辨率过低（低于 `1280×720`）；<br>2. 模拟器开启了异常滤镜/HDR/色彩增强；<br>3. 首次运行未联网下载 Bestdori 曲库（`本地数据/曲库/all.5.json` 为空）。 | 1. 确保分辨率为 `1280×720`（或等比更高 16:9，如 `1920×1080`），关闭画面滤镜；<br>2. 在「谱面」页点击更新曲库并下载目标谱面；<br>3. 保存 `结算超时画面.png` 以便补充新活动模板。 |

---

### D. 打歌偏移 / 漏键 / 断连 / 首拍不触发

| 现象 / 报错关键词 | 必查证据 | 常见根因 | 修复方式 |
| --- | --- | --- | --- |
| 报 `谱面 ... 的横向手势超出屏幕`，或点偏轨道 | `settings.json` 中的 `center`、`gap`、`touch_y` | 不同机型/黑边裁剪后，默认 7 轨几何参数（`center=0.5`, `gap=0.1154`, `touch_y=0.821`）与实际轨道不重合。 | 进入曲目暂停或截图轨道画面，在「演出设置 / 轨道校准」中对照画面重新对齐 `center`（中央轨）、`gap`（轨距）、`touch_y`（判定线）与 `detect_y`（首音符检测线，略高于判定线）。 |
| 进入曲目后一直不打、最终报未检测到首音符 | `开场-<曲目ID>-<难度>\diagnostics.json` 与 `last-plane.png`、`detect_y`、`sensitivity` | 1. `detect_y` 落点偏离音符下落区，或使用了暗色/极细音符皮肤导致亮度超过 220 的像素比例低于 `sensitivity`（默认 `0.3`）；<br>2. 游戏中开了炫光背景/3D MV/技能特效遮挡检测带；<br>3. 模拟器 H.264 视频帧异常。 | 1. 游戏内建议开启 **轻量模式 / 关闭 3D 演出 / 关闭背景暗度外的多余特效**，使用高亮清晰的默认音符皮肤（如皮肤 1/2），音符大小 `100%`；<br>2. 查看 `last-plane.png` 核对 `detect_y`（默认 `0.795`）是否正好压在轨道上方，可适当微调 `sensitivity`（如 `0.2`–`0.35`）。 |
| 整首歌稳定偏早或偏晚（大量 FAST / LATE GREAT 或整体错拍） | `本地数据\校时记录\*.json`、`settings.json` 中的 `delay_ms`、游戏内「节奏图标速度」与「判定调节」 | 每台电脑的解码延迟、模拟器帧率（60Hz / 120Hz）、音频/视频同步与游戏内下落速度不同，导致固定 `delay_ms` 偏移。 | 读取最近几局 `校时记录\*.json` 的整体时间残差趋势，在「演出设置」微调 `delay_ms`（**负值提前发送、正值延后发送**，每次 ±10ms 步进实测单曲/排练，也可在演奏中用方向键 `Left`/`Right` ±10ms 实时找准后再写入设置）。 |
| 局部偶发漏键/绿条断连，日志出现 `触控发送晚于排定时刻 10 ms 的批次` | `run.log`、任务管理器 CPU/GPU 占用、电池电源计划 | 电脑 CPU 负载过高、笔记本处于省电模式、后台杀软/录屏抢占调度，或模拟器帧率剧烈波动。 | 切换 Windows 电源计划为「高性能」，关闭无关高负载后台与录屏，将模拟器分配足够 CPU 核心（建议 ≥ 4 核）与内存（≥ 4 GB）并锁定稳定帧率（60 或 120 FPS）。 |
| 蓝条/滑条/粉键偶发掉连 | `settings.json` 中的 `gesture_engine`、`tap_hold_ms`、`slide_settle_ms`、`slide_head_hold_ms`、`slide_release_ms` | 触控引擎/滑条参数未适配当前触控延迟或使用了不兼容的音符判定策略。 | 优先恢复推荐默认值（`tap_hold_ms=25`、`slide_release_ms=4`、`jitter=0`），先用「AP」策略跑单曲/排练验证硬件稳定性，再切换「SSM」或「FC · 少量 GREAT」等策略。 |

---

### E. 结算卡住 / 自动补火 / 联网下载失败

| 现象 / 报错关键词 | 必查证据 | 常见根因 | 修复方式 |
| --- | --- | --- | --- |
| 结算流程长时间不动或超时报错 | 对应演出目录内的 `run.log`、`结算超时画面.png` | 新活动/周年/版本更新新增了未见过的全屏活动弹窗，或网络波动弹出重试对话框。 | 查看 `结算超时画面.png` 确认画面上停留的弹窗类型；如是网络断线弹窗请先恢复网络，如是新活动弹窗请保存截图与 `run.log` 反馈以便补充规则。 |
| 自动补火未触发或提示停止 | `run.log`、`settings.json` 中的 `auto_refill` / `refill_method` / `refill_amount`、游戏区服 | 自动补火目前仅支持 **国服（cn）** 的协力/组曲/对邦/5v5；为防误花星石，若道具不足、火数已 ≥ 3、或界面校验不通过（例如检测到 50 星石异常项）会主动停机保护。 | 确认区服为国服、所需火罐/星石余额充足、且已进入对应模式；不要绕过安全校验。 |
| 更新曲库、下载谱面或检查程序更新失败 | 界面运行日志、代理设置、`curl.exe` 可用性 | 本机网络无法直连 `bestdori.com` 或 `api.github.com`；程序不支持自动跳转至未知域名，urllib 失败后会回退调用系统 `curl.exe`。 | 在 PowerShell 中只读测试连通性：<br>`curl.exe -I --max-time 10 https://bestdori.com/api/songs/all.5.json`<br>`curl.exe -I --max-time 10 https://api.github.com/repos/Houraiji9/bandori-live-assistant/releases/latest`<br>若均超时，配置本机系统代理后重试。 |

---

## 4. 已知适配边界（避免误判为故障）

排查前请先核对用户的使用场景是否在当前支持范围内：

1. **操作系统**：仅支持 **Windows 10 / 11 64 位（x64）**，不支持 macOS / Linux / 32 位 Windows。
2. **设备与模式**：
   - **MuMu 模拟器（16:9，推荐 `1280×720`）**：支持全部自动化模式。
   - **安卓真机（USB 调试）**：仅支持「单曲 / 排练」单曲入口，不支持自动循环房间流程。
   - **其他模拟器（雷电/蓝叠/夜神等）**：未做进程扫描与自动循环验收，仅可尝试在「安卓手机」来源下测单曲。
3. **游戏区服与客户端**：仅支持 **国服 B 站官服（`com.bilibili.star.bili`）** 与 **日服官服（`jp.co.craftegg.band`）**；不支持渠道服、台服、国际服、韩服。部分功能（如自动补火、协力缺人/未到 SSS 跳车、免费排练连刷）目前仅适配国服。

---

## 5. 标准诊断汇报模板（供 Agent 输出）

排查完成后，请按以下固定格式向用户输出结论（若需进一步反馈给开发者，直接复制本段即可）：

```markdown
### 环境摘要
- 系统与架构：Windows XX (AMD64) / 屏幕缩放 XXX%
- 助手版本：1.0.XX（来自 active-version.json）
- 离线依赖自检（BANDORI_PORTABLE_SELFTEST）：通过 / 失败（附报错）
- 设备来源与模拟器：MuMu XX / 安卓真机（型号）
- 分辨率与 DPI：1280x720 / density 240（是否 16:9 无黑边：是/否）
- 游戏区服与包名：cn (com.bilibili.star.bili) / jp / 其他（写明实际包名）
- 触控后端：scrcpy / minitouch / mumu_sdk

### 命中根因
- [分类 A/B/C/D/E/边界] 一句话概括直接原因

### 关键证据
- 配置文件摘要（center / gap / touch_y / detect_y / sensitivity / delay_ms / region / backend）
- 关键日志片段（来自 运行日志/*.log 或 演出记录/*/run.log，已脱敏）
- 现场附件（如有：结算超时画面.png / 开场-*/last-plane.png / 校时记录/*.json 摘要）

### 建议修复步骤
1. ...
2. ...
```
