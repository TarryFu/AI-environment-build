# 排查ChatGPT打不开

**User:** Anonymous  
**Created:** 9/5/2026 15:11:59  
**Updated:** 9/5/2026 15:35:52  
**Exported:** 9/5/2026 15:40:20  
**Link:** [https://chatgpt.com/c/6a9bc0ab-b0ac-83e8-9629-dc3bdee02f5f](https://chatgpt.com/c/6a9bc0ab-b0ac-83e8-9629-dc3bdee02f5f)  

## Prompt:
2026/9/5 15:11:57

我的ChatGPT Windows 客户端 (原 codex) 在更新之后，打不开了，点击之后， ChatGPT.exe 有进程但软件没有窗口，请带我一步步排查并解决

## Response:
2026/9/5 15:12:08

你之前已经确认过两个关键点：`OpenAI.Codex` 包状态是 `Ok`，而 `ChatGPT.exe` 进程存在但 `MainWindowHandle = 0`。这说明问题更像是**程序启动后没有成功创建 UI 窗口**，而不是单纯“窗口跑到屏幕外”。我会按“尽量不破坏聊天记录/本地配置 → 再逐步加大修复力度”的顺序带你排查。

我查到一个与你现在状态几乎完全一致的最新回归：OpenAI 的 `openai/codex` 仓库在 **2026-09-04** 有用户报告，Windows 更新到 `26.901.2854.0` 后，`ChatGPT.exe` 后台进程存在、没有 renderer、没有可见窗口；重新注册 AppX、Reset、卸载重装、`--disable-gpu` 都无效。

而你之前查到的是 `26.901.4073.0`，同属 `26.901.x`，症状也完全一致。因此现在不建议先“重置/卸载”，先做**无损诊断**，避免不必要地动本地 Codex 状态。

可以。结合你前面已经得到的结果：

- 包：`OpenAI.Codex 26.901.4073.0`
- `Status = Ok`
- `ChatGPT.exe` 能启动
- `Responding = True`
- 但 `MainWindowHandle = 0`
- 点击图标后没有任何窗口

目前应优先把它视为 **26.901.x Windows 客户端更新后的启动回归**，而不是先怀疑你的 Windows 安装损坏。昨天 `openai/codex` 官方 GitHub 仓库刚出现 `26.901.2854.0` 的几乎相同报告：升级后后台进程存在，但没有 renderer 和窗口；Reset、重装、重新注册 AppX、禁用 GPU 都未解决。([GitHub](https://github.com/openai/codex/issues/42714))

另外，最近还有两类与你症状高度相关的问题：一类是 updater 初始化把程序卡在创建窗口之前，暂时禁用 updater 后即可打开；([GitHub](https://github.com/openai/codex/issues/41073)) 另一类是更新后的 CUA runtime staging 不完整，主进程存在但 renderer 无法启动。([GitHub](https://github.com/openai/codex/issues/42673))

所以我们按下面顺序诊断：

**Updater 卡死 → 隐藏 HWND → renderer/runtime → 日志 → 最后才考虑 Reset/重装。**

现在**只做第 1 步**，不要 Reset，也不要卸载。

### 第 1 步：临时禁用 updater，测试能否正常创建窗口

打开一个**普通 PowerShell 窗口，不需要管理员权限**。

先执行：

```powershell
Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Stop-Process -Force
```

然后确认所有进程已经结束：

```powershell
Get-Process ChatGPT -ErrorAction SilentlyContinue
```

正常情况下这里应该**没有任何输出**。

随后，在**同一个 PowerShell 窗口**完整执行：

```powershell
$env:CODEX_SPARKLE_ENABLED = "false"

$pkg = Get-AppxPackage -Name OpenAI.Codex

$pkg |
    Select-Object Name, Version, Status,
                  PackageFamilyName, InstallLocation

$exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"

Invoke-CommandInDesktopPackage `
    -PackageFamilyName $pkg.PackageFamilyName `
    -AppId "App" `
    -Command $exe
```

这是有针对性的诊断，不是随意修改系统。`CODEX_SPARKLE_ENABLED=false` **只对当前这个 PowerShell 会话启动出来的程序有效**；现在没有写系统环境变量、没有删除缓存、没有删除登录状态，也没有碰你的项目或聊天数据。

这种启动方法本身也是 Windows 提供的 packaged-app 调试机制；Microsoft 对 `Invoke-CommandInDesktopPackage` 的定义就是在指定 AppX/MSIX package context 中创建进程。([Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/appx/invoke-commandindesktoppackage?view=windowsserver2025-ps&utm_source=chatgpt.com)) 而最近的相同故障报告中，正是通过：

```text
CODEX_SPARKLE_ENABLED=false
```

绕过 updater 后，窗口恢复正常创建。([GitHub](https://github.com/openai/codex/issues/41073))

### 执行之后，只观察两件事

**情况 A：ChatGPT/Codex 窗口正常出现。**

那就基本锁定为：

> `26.901.x` 的 updater / startup initialization 回归

我们下一步会给你做一个**不影响聊天记录的稳定临时方案**，同时避免不必要的 Reset 或重装。

**情况 B：仍然只有后台进程，没有窗口。**

不要继续自行 Reset 或卸载。下一步我们直接检查：

- 是否已经存在 `--type=renderer`
- 是否存在隐藏的 `Chrome_WidgetWin_*` HWND
- `%LOCALAPPDATA%\OpenAI\Codex\runtimes\...` 是否处于不完整 staging 状态

这三项可以进一步精确区分当前 `26.901.4073.0` 属于哪一种已知启动故障。

**请现在先执行上面这一组命令，把 PowerShell 的完整输出贴给我，并告诉我：执行最后一条之后，窗口有没有出现。**我会从这个结果继续下一步，不会让你重复已经做过的检查。

---

**Sources:**

- [[Windows][26.820.9563.0] ChatGPT launches headless with MainWindowHandle=0; disabling updater fixes startup · Issue #41073 · openai/codex · GitHub](https://github.com/openai/codex/issues/41073)
- [Invoke-CommandInDesktopPackage (Appx) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/appx/invoke-commandindesktoppackage?view=windowsserver2025-ps&utm_source=chatgpt.com)

## Prompt:
2026/9/5 15:15:22

运行结果：

PS C:\Users\24503> Get-Process ChatGPT -ErrorAction SilentlyContinue |
>>     Stop-Process -Force
PS C:\Users\24503> Get-Process ChatGPT -ErrorAction SilentlyContinue
PS C:\Users\24503> $env:CODEX_SPARKLE_ENABLED = "false"
PS C:\Users\24503>
PS C:\Users\24503> $pkg = Get-AppxPackage -Name OpenAI.Codex
PS C:\Users\24503>
PS C:\Users\24503> $pkg |
>>     Select-Object Name, Version, Status,
>>                   PackageFamilyName, InstallLocation


Name              : OpenAI.Codex
Version           : 26.901.5280.0
Status            : Ok
PackageFamilyName : OpenAI.Codex_2p2nqsd0c76g0
InstallLocation   : C:\Program Files\WindowsApps\OpenAI.Codex_26.901.5280.0_x64__2p2nqsd0c76g0



PS C:\Users\24503>
PS C:\Users\24503> $exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"
PS C:\Users\24503>
PS C:\Users\24503> Invoke-CommandInDesktopPackage `
>>     -PackageFamilyName $pkg.PackageFamilyName `
>>     -AppId "App" `
>>     -Command $exe
PS C:\Users\24503>

仍然有进程，但无窗口

## Response:
2026/9/5 15:15:28

这个结果已经排除了一个重要分支：**不是 Sparkle updater 初始化卡死**。而且你现在的版本已经是 `26.901.5280.0`，比前面测试时的 `26.901.4073.0` 又更新了一版，但症状完全不变。

下一步我会先区分两类故障：**主进程根本没有拉起 renderer/UI 子进程**，还是 **renderer 已启动但窗口创建失败**。这一步仍然是只读诊断，不改配置、不清缓存。

目前可以把 **Sparkle updater** 这一条暂时排除。近期 `26.901.x` 的 Windows 启动故障主要有两种非常接近的表现：

- 一种是主进程启动，但 **renderer 根本没有生成**，启动流程阻塞在 Artifact Session host；公开报告里的关键错误是 `Unix-socket transport is not available on Windows`。([GitHub](https://github.com/openai/codex/issues/42669))
- 另一种是 `cua_node` runtime 更新后反复停留在 `.staging-*`，尤其可能出现 `node.exe` 已有、`node_repl.exe` 缺失，导致同样的“有进程、无窗口”。该报告最终通过恢复完整 runtime 解决。([GitHub](https://github.com/openai/codex/issues/42501))

你的症状目前两者都符合，所以**现在先不要 Reset、卸载或删缓存**。

## 第 2 步：确认 renderer 和 `cua_node` 状态

保持现在这个普通 PowerShell 窗口即可。

先彻底结束刚才的测试进程：

```powershell
Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Stop-Process -Force

Start-Sleep -Seconds 2
```

然后重新做一次受控启动：

```powershell
$env:CODEX_SPARKLE_ENABLED = "false"

$pkg = Get-AppxPackage -Name OpenAI.Codex
$exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"

Invoke-CommandInDesktopPackage `
    -PackageFamilyName $pkg.PackageFamilyName `
    -AppId "App" `
    -Command $exe `
    -Args "--enable-logging --v=1"
```

启动后不用反复点击图标。等待大约十几秒，然后执行下面整段诊断代码。

```powershell
Write-Host "`n========== 1. PROCESS TREE ==========" -ForegroundColor Cyan

Get-CimInstance Win32_Process |
    Where-Object {
        $_.Name -in @(
            "ChatGPT.exe",
            "codex.exe",
            "node.exe",
            "node_repl.exe"
        )
    } |
    Select-Object ProcessId,
                  ParentProcessId,
                  Name,
                  CommandLine |
    Format-List

Write-Host "`n========== 2. WINDOW STATE ==========" -ForegroundColor Cyan

Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Select-Object Id,
                  ProcessName,
                  Responding,
                  MainWindowHandle,
                  MainWindowTitle,
                  CPU |
    Format-Table -AutoSize

Write-Host "`n========== 3. CUA NODE RUNTIME ==========" -ForegroundColor Cyan

$runtimeCandidates = @(
    "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node",
    "$env:LOCALAPPDATA\Packages\$($pkg.PackageFamilyName)\LocalCache\Local\Codex\runtimes\cua_node",
    "$env:LOCALAPPDATA\Packages\$($pkg.PackageFamilyName)\LocalCache\Local\OpenAI\Codex\runtimes\cua_node"
) | Select-Object -Unique

$runtimeFound = $false

foreach ($rt in $runtimeCandidates) {

    if (Test-Path $rt) {

        $runtimeFound = $true

        Write-Host "`nRuntime root:"
        Write-Host $rt

        Get-ChildItem $rt -Force -Directory |
            Sort-Object LastWriteTime -Descending |
            Select-Object Name,
                          LastWriteTime,
                          @{
                              Name = "node.exe"
                              Expression = {
                                  Test-Path (
                                      Join-Path $_.FullName "bin\node.exe"
                                  )
                              }
                          },
                          @{
                              Name = "node_repl.exe"
                              Expression = {
                                  Test-Path (
                                      Join-Path $_.FullName "bin\node_repl.exe"
                                  )
                              }
                          },
                          @{
                              Name = "manifest.json"
                              Expression = {
                                  Test-Path (
                                      Join-Path $_.FullName "manifest.json"
                                  )
                              }
                          } |
            Format-Table -AutoSize
    }
}

if (-not $runtimeFound) {
    Write-Host "No cua_node runtime directory found."
}

Write-Host "`n========== 4. LATEST CODEX DESKTOP LOG ==========" -ForegroundColor Cyan

$logCandidates = @(
    "$env:LOCALAPPDATA\Packages\$($pkg.PackageFamilyName)\LocalCache\Local\Codex\Logs",
    "$env:LOCALAPPDATA\OpenAI\Codex\Logs",
    "$env:LOCALAPPDATA\Codex\Logs"
) | Select-Object -Unique

$logs = @()

foreach ($logRoot in $logCandidates) {
    if (Test-Path $logRoot) {
        $logs += Get-ChildItem `
            -Path $logRoot `
            -Filter "codex-desktop-*.log" `
            -File `
            -Recurse `
            -ErrorAction SilentlyContinue
    }
}

$latestLog = $logs |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

if ($latestLog) {

    Write-Host "Latest log:"
    Write-Host $latestLog.FullName
    Write-Host ""

    Get-Content $latestLog.FullName -Tail 100

} else {

    Write-Host "No codex-desktop log found."
}
```

### 我主要要看什么

第一部分最关键。正常 Electron 应用最终应出现类似：

```text
ChatGPT.exe ... --type=renderer ...
```

如果你的输出里只有：

```text
ChatGPT.exe
ChatGPT.exe --type=crashpad...
ChatGPT.exe --type=gpu-process...
ChatGPT.exe --type=utility...
```

却**完全没有 `--type=renderer`**，那么已经可以确认不是“窗口位置、显示器、多屏幕或最小化”的问题，而是程序停在 **BrowserWindow/renderer 创建之前**。这与目前报告的 Windows 回归特征一致。([GitHub](https://github.com/openai/codex/issues/42669))

第三部分尤其注意是否出现类似：

```text
.staging-xxxxxxxxxxxxxxxx-XXXXXX
```

以及：

```text
node.exe       True
node_repl.exe  False
```

如果是这种结果，几乎可以直接进入 **CUA runtime 修复路线**。近期公开案例就是这种状态，修复完整 runtime 后 GUI 随即恢复。([GitHub](https://github.com/openai/codex/issues/42501))

如果 runtime 全部完整，但日志出现：

```text
Failed to start the Artifact Session host
```

或者：

```text
Unix-socket transport is not available on Windows
```

那么则属于另一条 `26.901.x` 启动回归，单纯重装、Reset、清缓存通常没有意义。([GitHub](https://github.com/openai/codex/issues/42669))

**现在只做到这里。把这四个区块的完整 PowerShell 输出贴给我。** 我会根据结果直接判断下一步是修复 `cua_node`、处理 Artifact Session，还是转向 renderer/Windows 图形层，不会让你盲目试一遍各种通用方案。

---

**Sources:**

- [[Windows] Codex desktop app (26.901.2854.0): processes launch but no window ever appears - "Artifact Session host Unix-socket transport is not available on Windows" · Issue #42669 · openai/codex · GitHub](https://github.com/openai/codex/issues/42669)

## Prompt:
2026/9/5 15:21:32

完整运行结果如下：

[Attachment: 粘贴的文本 (1)(2).txt]

## Response:
2026/9/5 15:21:35

从这份完整输出里，已经能得到两个非常明确的结论，我先直接给出来：

1. **GUI renderer 确实没有启动**。你的进程树只有主进程、crashpad、GPU、network/storage utility，没有任何 `--type=renderer`；所有 `MainWindowHandle` 都是 `0`。这排除了“窗口跑到屏幕外/被最小化/多显示器位置异常”。
2. **`cua_node` 更新过程反复失败并留下 staging 目录**：连续多个 `.staging-440c4f...`，且里面 `node.exe=True`、`node_repl.exe=False`、`manifest.json=False`；与此同时旧的 `c2900163265bdaec` 目录是完整的。

这已经很像一个“新 runtime 下载/展开失败 → 启动链卡在 renderer 创建之前”的问题。我再核对一下当前官方 issue 中这两个错误是否已确认及有没有稳定修复路径，然后给你下一步。

我也核对了当前 `openai/codex` 官方 GitHub 仓库中的近期报告。你的机器现在与其中一类故障**高度吻合到目录结构级别**：后台主进程存在、无 renderer、每次启动新增 `.staging-<runtime-id>-...`、`node.exe` 已复制但 `node_repl.exe` 缺失。已有案例通过从当前 MSIX 包中用 `xcopy /G` 恢复**完整的** `cua_node` runtime 后立即恢复 GUI。

但你的控制台还同时出现了另一个独立信号：`Artifact Session host Unix-socket transport is not available on Windows.`  近期也有报告显示，这个错误本身可能继续阻塞 Windows 启动。 因此最合理的下一步不是猜，而是先把**已经证实损坏的 `cua_node` runtime 修完整**；如果修完仍无窗口，我们就能把剩余问题精确锁定到 Artifact Session 回归。

你的输出已经足够进入**实际修复阶段**，不需要再做泛化排查。

目前状态可以概括为：

> AppX/MSIX 包正常 → `ChatGPT.exe` 主进程正常启动 → GPU/utility 正常启动 → **renderer 未启动** → **当前 `cua_node` runtime `440c4f095d41ea30` 每次 staging 都失败**。

你的多个 staging 目录均表现为 `node.exe=True`、`node_repl.exe=False`、`manifest.json=False`，而旧 runtime 是完整的。`粘贴的文本 (1).txt`  
进程树中也确实没有任何 `--type=renderer`。`粘贴的文本 (1).txt`

## 第 3 步：修复当前版本的 `cua_node` runtime

这一操作**不会 Reset ChatGPT，不会卸载应用，也不会删除聊天记录、项目记录或 Codex 配置**。我们只修复：

```text
C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node
```

下面不要自行改路径。

### 3.1 先关闭所有 ChatGPT 进程

普通 PowerShell：

```powershell
Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Stop-Process -Force

Start-Sleep -Seconds 2

Get-Process ChatGPT -ErrorAction SilentlyContinue
```

最后一条应当没有输出。

---

### 3.2 检查 MSIX 包内的原始 runtime 是否完整

执行：

```powershell
$pkg = Get-AppxPackage -Name OpenAI.Codex

$src = Join-Path $pkg.InstallLocation "app\resources\cua_node"

Write-Host "Source runtime:"
Write-Host $src

Write-Host "`nKey files:"
[PSCustomObject]@{
    Manifest = Test-Path (Join-Path $src "manifest.json")
    Node     = Test-Path (Join-Path $src "bin\node.exe")
    NodeRepl = Test-Path (Join-Path $src "bin\node_repl.exe")
}

Write-Host "`nSource file count:"
(Get-ChildItem $src -Recurse -File -Force).Count
```

这里我们期望看到：

```text
Manifest : True
Node     : True
NodeRepl : True
```

这是关键前置条件。

---

### 3.3 自动识别这次更新要求的 runtime ID

继续执行：

```powershell
$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

$latestStaging = Get-ChildItem $runtimeRoot -Force -Directory |
    Where-Object { $_.Name -like ".staging-*" } |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

Write-Host "Latest staging:"
Write-Host $latestStaging.Name

if ($latestStaging.Name -match '^\.staging-([0-9a-f]+)-') {
    $runtimeId = $Matches[1]
} else {
    throw "Unable to identify runtime ID."
}

Write-Host "Required runtime ID:"
Write-Host $runtimeId
```

根据你刚才的结果，它应该输出：

```text
440c4f095d41ea30
```

这一步不要手工填写 hash，让程序自己识别。

---

## 3.4 建立正式 runtime 目录

先看看目标是否已经存在：

```powershell
$dst = Join-Path $runtimeRoot $runtimeId

Write-Host "Destination:"
Write-Host $dst

Test-Path $dst
```

根据你刚才的目录列表，这里预计是：

```text
False
```

然后创建：

```powershell
New-Item `
    -ItemType Directory `
    -Path $dst `
    -Force
```

---

## 3.5 用 `xcopy /G` 完整复制官方 runtime

这是本次修复最关键的一步。

**不要使用普通 `Copy-Item`。**

执行：

```powershell
& "$env:SystemRoot\System32\xcopy.exe" `
    "$src\*" `
    "$dst\" `
    /E /I /H /Y /G /Q
```

这里 `/G` 很重要。

近期相同故障中，MSIX/WindowsApps 内的 `node_repl.exe` 可能具有 Windows 的 Application Protected/EFS 类保护属性，普通复制路径会在 relocation 阶段失败；`xcopy /G` 允许将受保护源复制到非加密目标。官方仓库里的多个近期案例正是通过该方式恢复。([GitHub](https://github.com/openai/codex/issues/42501?utm_source=chatgpt.com))

复制完成后，应出现类似：

```text
xxxx File(s) copied
```

具体数量以你当前 `26.901.5280.0` 包为准，不要求与别人版本完全相同。

---

## 3.6 验证修复后的 runtime 完整性

执行：

```powershell
Write-Host "`n========== RUNTIME VERIFY ==========" -ForegroundColor Cyan

[PSCustomObject]@{
    RuntimeID = $runtimeId
    Manifest  = Test-Path (Join-Path $dst "manifest.json")
    Node      = Test-Path (Join-Path $dst "bin\node.exe")
    NodeRepl  = Test-Path (Join-Path $dst "bin\node_repl.exe")
    SrcFiles  = (Get-ChildItem $src -Recurse -File -Force).Count
    DstFiles  = (Get-ChildItem $dst -Recurse -File -Force).Count
}
```

理想结果应类似：

```text
RuntimeID : 440c4f095d41ea30
Manifest  : True
Node      : True
NodeRepl  : True
SrcFiles  : XXXX
DstFiles  : XXXX
```

重点是：

```text
Manifest = True
Node = True
NodeRepl = True
SrcFiles = DstFiles
```

### 再进一步核对三个关键文件

执行：

```powershell
Write-Host "`n========== HASH VERIFY ==========" -ForegroundColor Cyan

$checkFiles = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($f in $checkFiles) {

    $srcFile = Join-Path $src $f
    $dstFile = Join-Path $dst $f

    $srcHash = (Get-FileHash $srcFile -Algorithm SHA256).Hash
    $dstHash = (Get-FileHash $dstFile -Algorithm SHA256).Hash

    [PSCustomObject]@{
        File  = $f
        Match = ($srcHash -eq $dstHash)
    }
}
```

三个应全部：

```text
Match
-----
True
True
True
```

---

# 3.7 暂时不要删除 `.staging-*`

你现在已经积累：

```text
.staging-440c4f095d41ea30-B12d0k
.staging-440c4f095d41ea30-lujtS4
.staging-440c4f095d41ea30-AE8AY7
.staging-440c4f095d41ea30-2YaRTY
.staging-440c4f095d41ea30-Ad553T
```

`粘贴的文本 (1).txt`

**先留着。**

它们以后可以清理，但目前这些目录是诊断证据，而且不会妨碍我们建立正确的：

```text
cua_node\440c4f095d41ea30
```

现在没必要删除任何东西。

---

# 3.8 第一次测试启动

仍然在当前 PowerShell 中：

```powershell
$env:CODEX_SPARKLE_ENABLED = "false"

$exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"

Invoke-CommandInDesktopPackage `
    -PackageFamilyName $pkg.PackageFamilyName `
    -AppId "App" `
    -Command $exe
```

然后观察。

### 如果窗口出现

那就说明根因已经确定为：

> **MSIX 更新后的 `cua_node` runtime relocation/staging 失败。**

这个现象已经在近期多个 Windows Codex Desktop 报告中复现，而且修复完整 runtime 后 renderer 和 GUI 会恢复。([GitHub](https://github.com/openai/codex/issues/42673?utm_source=chatgpt.com))

之后我们还会做两件事：

1. 验证从 Windows 开始菜单普通启动也能打开；
2. 判断下一次自动更新是否可能再次触发，从而制定稳定方案。

---

### 如果仍然没有窗口

也不要再做别的。

这时我们已经排除了：

- AppX 注册；
- updater；
- 显卡/window placement；
- 缺失 `cua_node` runtime；

而剩下最值得怀疑的就是你已经**实际捕获到**的：

```text
Failed to start the Artifact Session host
Artifact Session host Unix-socket transport is not available on Windows.
```

`粘贴的文本 (1).txt`

这个错误与 2026 年 9 月 4 日公开的 Windows `26.901.x` 无窗口回归报告一致；该案例同样表现为主进程运行但 renderer 不创建。([GitHub](https://github.com/openai/codex/issues/42669?utm_source=chatgpt.com))

如果到了这一分支，我们下一步就不再碰 runtime，而是专门处理 **Artifact Session bootstrap regression**。

---

**现在请先执行到 3.8。**  
把 **3.2、3.3、3.5、3.6、HASH VERIFY** 的输出，以及最后“窗口是否出现”贴给我。不要 Reset 或重装应用。

---

**Sources:**

- [[Windows] 26.901.1978.0 fails to launch UI when cua_node staging cannot copy node_repl.exe · Issue #42501 · openai/codex · GitHub](https://github.com/openai/codex/issues/42501?utm_source=chatgpt.com)

## Prompt:
2026/9/5 15:32:58

执行完：
$env:CODEX_SPARKLE_ENABLED = "false"

$exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"

Invoke-CommandInDesktopPackage `
    -PackageFamilyName $pkg.PackageFamilyName `
    -AppId "App" `
    -Command $exe

之后，窗口正常出现，同时，我也测试了，从 Windows 开始菜单普通启动也能打开  

现在，需要你帮我完成：判断下一次自动更新是否可能再次触发，从而制定稳定方案。

## Response:
2026/9/5 15:33:07

现在可以确认：这次故障的直接原因已经被修复为“**当前版本所需的 `cua_node` runtime 未能完整落盘**”。接下来要判断的是：这个故障属于一次性安装异常，还是**每次应用更新、runtime hash 变化时都可能复发**。

我会重点核对两件事：近期官方 issue 是否确认复发条件，以及 Windows/MSIX 更新机制下，哪些措施能真正降低复发概率，而不是仅仅“暂时能打开”。

结论已经比较清楚：**下一次自动更新确实有复发可能，而且复发点不是当前这个已经修好的目录，而是“新版本带来新的 `cua_node` runtime ID 时，程序再次尝试从 WindowsApps 复制受保护文件”。**

官方 issue #42501 也明确指出：runtime hash 会随版本更新变化，因此旧版本已经完整的 runtime 不能保护新版本；只要 Windows 下这个 protected-file relocation 缺陷尚未修复，**任何引入新 runtime hash 的更新都可能再次触发同类故障**。

可以判断，而且现在证据已经比较充分：

**下一次自动更新仍有复发风险，而且在官方修复这一 Windows runtime relocation 问题之前，我建议把它当作“可重复出现的问题”，而不是一次偶发故障。**

你这次机器上的因果链已经很清楚：新版本 `26.901.5280.0` 需要新的 `cua_node` runtime `440c4f095d41ea30`，程序每次启动都创建新的 `.staging-440c4f...`，但 staging 中始终缺少 `node_repl.exe` 和 `manifest.json`；旧 runtime 则完整。`粘贴的文本 (1).txt` 手动用 `/G` 完整恢复新 runtime 后，**同一个 ChatGPT 安装包立即恢复正常，而且开始菜单普通启动也恢复**。所以对你这台机器而言，直接阻塞启动的就是 runtime staging/relocation。

官方近期的 Windows issue 也明确指出：**Codex 更新后 runtime hash 会变化，因此旧版本已经完整的 runtime 并不能保护新版本；如果受保护文件复制问题仍存在，每次引入新 `cua_node` runtime 的更新都有可能再次复发。** ([GitHub](https://github.com/openai/codex/issues/42501))

### 我建议采用下面这一套稳定方案

1. **暂时冻结当前已经验证正常的 `26.901.5280.0`。**  
   如果你目前非常依赖 ChatGPT/Codex Windows 客户端做科研工作，我建议现阶段关闭 Microsoft Store 的自动应用更新，改为**手动更新**。通常路径是 Microsoft Store → 头像/个人资料 → Settings → `App updates` 关闭。需要注意，这是 Store 的全局应用自动更新开关，而不只是 ChatGPT，所以其他 Store 应用也要偶尔手动更新。

   我不建议卸载 Store 版、修改 WindowsApps 权限、阻止 Microsoft Store 服务或使用第三方“版本锁定”工具，这些操作的副作用远大于收益。

2. **不要永久设置 `CODEX_SPARKLE_ENABLED=false`。**  
   你已经证明：修好 runtime 后，从 Windows 开始菜单普通启动也正常。这意味着 Sparkle 并不是你这次问题的根因。你之前设置的：
   ```powershell
   $env:CODEX_SPARKLE_ENABLED = "false"
   ```
   只存在于那个 PowerShell 会话中，本来就不是永久环境变量，因此现在**不需要做任何清理或永久配置**。同时，它也不能阻止 Microsoft Store 给 MSIX 包升级，所以不能作为版本冻结方案。

3. **以后不要第一时间安装 Codex/ChatGPT Windows 新版。**  
   当前这类故障已经跨多个 Windows build 出现：例如 `26.825.5331.0`、`26.901.1978.0`，现在你的 `26.901.5280.0` 也实际复现了同样的 staging 模式。([GitHub](https://github.com/openai/codex/issues/41540))  
   比较稳妥的工作流是：新版本发布后先观察相关 Windows issue；确认 `cua_node` staging / `node_repl.exe` relocation 已修，再手动升级。当前公开 issue 仍把这一问题作为 Windows bug 报告，而不是已经确认修复的问题。([GitHub](https://github.com/openai/codex/issues/42673))

4. **每次手动更新后，第一次启动只做一次快速健康检查。**  
   先查看版本：
   ```powershell
   Get-AppxPackage -Name OpenAI.Codex |
       Select-Object Name, Version, Status, InstallLocation
   ```
   正常启动后，再看有没有 renderer：
   ```powershell
   Get-CimInstance Win32_Process |
       Where-Object {
           $_.Name -eq "ChatGPT.exe" -and
           $_.CommandLine -match "--type=renderer"
       } |
       Select-Object ProcessId, CommandLine
   ```
   如果有输出并且 GUI 正常，则无需继续检查。

   如果再次出现“有进程但无窗口”，第一时间执行：
   ```powershell
   Get-ChildItem `
       "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node" `
       -Force -Directory |
       Sort-Object LastWriteTime -Descending |
       Select-Object Name, LastWriteTime
   ```
   如果又看到：
   ```text
   .staging-新的runtimeID-xxxxxx
   .staging-新的runtimeID-yyyyyy
   ```
   就不要再 Reset、重装或清聊天数据--基本可以直接走我们今天已经验证成功的 runtime 修复路线。

5. **建议在电脑上保存一个“一键修复脚本”，作为保险。**  
   下面这个脚本针对你今天实际遇到的故障设计。未来若升级后再次出现“后台有进程、无窗口”，它会自动识别最新 staging runtime ID，从**当前安装的官方 MSIX 包**复制对应 runtime，而不是写死今天的 `440c4f095d41ea30`。

```powershell
# ============================================================
# Codex / ChatGPT Windows - CUA Runtime Repair
# 用于：
#   ChatGPT.exe 有后台进程，但没有 GUI；
#   cua_node 下不断生成 .staging-*；
#   node_repl.exe relocation/staging 失败。
# ============================================================

$ErrorActionPreference = "Stop"

Write-Host "`n=== 1. Stop ChatGPT ===" -ForegroundColor Cyan

Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Stop-Process -Force

Start-Sleep -Seconds 2

Write-Host "`n=== 2. Locate installed package ===" -ForegroundColor Cyan

$pkg = Get-AppxPackage -Name OpenAI.Codex

if (-not $pkg) {
    throw "OpenAI.Codex package was not found."
}

Write-Host "Version:"
Write-Host $pkg.Version

$src = Join-Path $pkg.InstallLocation "app\resources\cua_node"

if (-not (Test-Path $src)) {
    throw "Bundled cua_node runtime was not found: $src"
}

Write-Host "`n=== 3. Verify packaged runtime ===" -ForegroundColor Cyan

$srcManifest = Join-Path $src "manifest.json"
$srcNode     = Join-Path $src "bin\node.exe"
$srcNodeRepl = Join-Path $src "bin\node_repl.exe"

[PSCustomObject]@{
    Manifest = Test-Path $srcManifest
    Node     = Test-Path $srcNode
    NodeRepl = Test-Path $srcNodeRepl
}

if (
    -not (Test-Path $srcManifest) -or
    -not (Test-Path $srcNode) -or
    -not (Test-Path $srcNodeRepl)
) {
    throw "The packaged cua_node runtime itself is incomplete. Stop here."
}

Write-Host "`n=== 4. Detect failed runtime ID ===" -ForegroundColor Cyan

$runtimeRoot =
    "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

if (-not (Test-Path $runtimeRoot)) {
    throw "Runtime root was not found: $runtimeRoot"
}

$latestStaging =
    Get-ChildItem $runtimeRoot -Force -Directory |
    Where-Object { $_.Name -like ".staging-*" } |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

if (-not $latestStaging) {
    throw @"
No .staging-* runtime was found.

This repair script is intended for the known cua_node staging failure.
Do not guess a runtime ID manually.
"@
}

if ($latestStaging.Name -notmatch '^\.staging-([0-9a-f]+)-') {
    throw "Unable to parse runtime ID from $($latestStaging.Name)"
}

$runtimeId = $Matches[1]
$dst = Join-Path $runtimeRoot $runtimeId

Write-Host "Detected runtime ID:"
Write-Host $runtimeId

Write-Host "Destination:"
Write-Host $dst

Write-Host "`n=== 5. Check whether runtime is already complete ===" `
    -ForegroundColor Cyan

$alreadyComplete =
    (Test-Path (Join-Path $dst "manifest.json")) -and
    (Test-Path (Join-Path $dst "bin\node.exe")) -and
    (Test-Path (Join-Path $dst "bin\node_repl.exe"))

if ($alreadyComplete) {

    Write-Host `
        "Runtime already contains all critical files. No repair performed." `
        -ForegroundColor Green

    exit 0
}

Write-Host "`n=== 6. Prepare destination ===" -ForegroundColor Cyan

if (Test-Path $dst) {

    $backup =
        "$dst.broken-" +
        (Get-Date -Format "yyyyMMdd-HHmmss")

    Rename-Item `
        -LiteralPath $dst `
        -NewName (Split-Path $backup -Leaf)

    Write-Host "Existing incomplete runtime renamed to:"
    Write-Host $backup
}

New-Item `
    -ItemType Directory `
    -Path $dst `
    -Force |
    Out-Null

Write-Host "`n=== 7. Restore runtime using xcopy /G ===" `
    -ForegroundColor Cyan

& "$env:SystemRoot\System32\xcopy.exe" `
    "$src\*" `
    "$dst\" `
    /E /I /H /Y /G /Q

if ($LASTEXITCODE -gt 1) {
    throw "xcopy failed with exit code $LASTEXITCODE."
}

Write-Host "`n=== 8. Verify repaired runtime ===" `
    -ForegroundColor Cyan

$srcCount =
    (Get-ChildItem $src -Recurse -File -Force).Count

$dstCount =
    (Get-ChildItem $dst -Recurse -File -Force).Count

$result = [PSCustomObject]@{
    PackageVersion = $pkg.Version
    RuntimeID      = $runtimeId
    Manifest       = Test-Path (
        Join-Path $dst "manifest.json"
    )
    Node           = Test-Path (
        Join-Path $dst "bin\node.exe"
    )
    NodeRepl       = Test-Path (
        Join-Path $dst "bin\node_repl.exe"
    )
    SourceFiles    = $srcCount
    DestFiles      = $dstCount
}

$result | Format-List

if (
    -not $result.Manifest -or
    -not $result.Node -or
    -not $result.NodeRepl -or
    ($result.SourceFiles -ne $result.DestFiles)
) {
    throw "Runtime verification failed."
}

Write-Host "`n=== 9. SHA-256 critical-file verification ===" `
    -ForegroundColor Cyan

$files = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($file in $files) {

    $sourceHash =
        (Get-FileHash `
            (Join-Path $src $file) `
            -Algorithm SHA256).Hash

    $destHash =
        (Get-FileHash `
            (Join-Path $dst $file) `
            -Algorithm SHA256).Hash

    [PSCustomObject]@{
        File  = $file
        Match = ($sourceHash -eq $destHash)
    }
}

Write-Host `
    "`nCUA runtime repair completed successfully." `
    -ForegroundColor Green

Write-Host `
    "Launch ChatGPT normally from the Windows Start menu."
```

这个脚本有几个刻意的安全设计：

- **不写死版本号；**
- **不写死 runtime hash；**
- 只使用当前正式安装包中的 `app\resources\cua_node`；
- 不修改 WindowsApps；
- 不删除 Codex 用户目录；
- 不动登录、聊天记录、项目、配置；
- 如果找不到 `.staging-*`，它会停止，而不是猜一个 runtime；
- 如果安装包本身不完整，也会停止；
- 修复后验证文件数量和三个核心文件 SHA-256。

因此，它适合作为当前 bug 未解决期间的应急工具。

### 关于你日志里的 `Artifact Session host` 错误

这一点现在也可以重新判断。

你之前确实捕获到了：

```text
Failed to start the Artifact Session host
Artifact Session host Unix-socket transport is not available on Windows.
```

`粘贴的文本 (1).txt`

官方另一个 issue 也把它与 Windows 无窗口问题联系起来。([GitHub](https://github.com/openai/codex/issues/42669))

但**对你的这台机器而言，它至少不是这一次故障的决定性根因**。理由很强：

> 我们没有更换 `26.901.5280.0` 安装包，也没有修改 Artifact Session 代码，仅仅把 `cua_node` runtime 完整恢复后，同一个版本立即恢复 GUI，甚至开始菜单普通启动也正常。

因此以后再出现无窗口，诊断优先级应调整为：

**先检查 `cua_node/.staging-*` → 再检查 renderer → 最后才调查 Artifact Session。**

这比我们最开始的故障树已经精确很多。

### 最终建议

对于你这种把 Codex 当作长期科研工作环境、而且旧聊天记录和项目连续性非常重要的使用场景，我建议目前采用：

> **`26.901.5280.0` 保持不动 + Microsoft Store 暂停自动更新 + 新版观察几天后再人工更新 + 本地保存上述 repair 脚本。**

这比继续让 Store 自动升级然后被动处理更稳。

等官方明确修复 `cua_node` 的 Windows protected-file relocation/staging 问题后，再恢复自动更新更合理。当前公开报告已经显示该问题跨多个版本出现，所以暂时没有依据把今天的成功修复视为“以后不会再发生”。([GitHub](https://github.com/openai/codex/issues/42501))

suggest_automation

---

**Sources:**

- [https://github.com/openai/codex/issues/42501](https://github.com/openai/codex/issues/42501)



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)