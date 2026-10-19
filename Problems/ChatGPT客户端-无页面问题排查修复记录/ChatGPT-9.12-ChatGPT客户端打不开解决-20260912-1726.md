# 9.12-ChatGPT客户端打不开解决

**User:** Anonymous  
**Created:** 9/12/2026 15:54:05  
**Updated:** 9/12/2026 16:19:55  
**Exported:** 9/12/2026 17:26:10  
**Link:** [https://chatgpt.com/g/g-p-6a9fc63d99c481919dd12e2056b7a0ef-chatgpt-ke-hu-duan-bug-xiu-fu/c/6aa5051b-ca48-83e8-9509-30ea7bf0b956](https://chatgpt.com/g/g-p-6a9fc63d99c481919dd12e2056b7a0ef-chatgpt-ke-hu-duan-bug-xiu-fu/c/6aa5051b-ca48-83e8-9509-30ea7bf0b956)  

## Prompt:
2026/9/12 15:54:01

我的ChatGPT Windows 客户端 (原 codex) 在更新之后，打不开了，点击之后， ChatGPT.exe 有进程但软件没有窗口，请带我一步步排查并解决，
这是之前的排查记录，

[Attachment: ChatGPT-ChatGPT客户端界面打不开解决-20260911(1).md]

## Response:
2026/9/12 15:54:06

我会以你昨天的排查记录为基线，不重复做 Reset、重装、清缓存这类泛化操作。昨天已经实证过一次：更新后新 `cua_node` runtime 的 staging 不完整，补齐当前 MSIX 自带 runtime 后，`renderer` 恢复、窗口重新出现。

这次先只做**只读诊断**，确认是否又是同一故障；在证据出来前，不改任何文件。

先从**第 1 步：确认这次是否仍是同一个 `cua_node` runtime staging 故障**开始。

你的上一轮记录已经建立了很清楚的基线：当时 `26.903.8094.0` 更新后，新 runtime `b58ca2eaa616c2da` 的 `.staging-*` 目录里只有 `node.exe`，缺 `node_repl.exe` 和 `manifest.json`，而安装包中的官方源文件完整；人工用 `xcopy /G` 重建后，关键文件 SHA-256 一致，随后 `renderer` 重新出现、`MainWindowHandle` 由 `0` 变为非零，GUI 恢复。`ChatGPT-ChatGPT客户端界面打不开解决-20260911.md` `ChatGPT-ChatGPT客户端界面打不开解决-20260911.md` `ChatGPT-ChatGPT客户端界面打不开解决-20260911.md`

所以这次**不要先重装、Reset、删除 runtime 或 staging**。

### 第 1 步：只读检查当前状态

保持现在这个状态：

> 已经点击过 ChatGPT，任务管理器里有 `ChatGPT.exe`，但是没有窗口。

打开**普通 PowerShell，不需要管理员权限**，一次性运行下面整段：

```powershell
$pkg = Get-AppxPackage -Name OpenAI.Codex

Write-Host "`n========== 1. PACKAGE ==========" -ForegroundColor Cyan

if ($null -eq $pkg) {
    Write-Host "OpenAI.Codex package not found." -ForegroundColor Red
} else {
    $pkg |
        Select-Object Name,
                      Version,
                      Status,
                      PackageFamilyName,
                      InstallLocation |
        Format-List
}

Write-Host "`n========== 2. CHATGPT PROCESS TREE ==========" -ForegroundColor Cyan

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

Write-Host "`n========== 3. WINDOW STATE ==========" -ForegroundColor Cyan

Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Select-Object Id,
                  Responding,
                  MainWindowHandle,
                  MainWindowTitle,
                  CPU |
    Format-Table -AutoSize

Write-Host "`n========== 4. CUA RUNTIME ==========" -ForegroundColor Cyan

$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

Write-Host "Runtime root:"
Write-Host $runtimeRoot

if (Test-Path $runtimeRoot) {

    Get-ChildItem $runtimeRoot -Force -Directory |
        Sort-Object LastWriteTime -Descending |
        Select-Object -First 20 `
            Name,
            LastWriteTime,
            @{
                Name = "node.exe"
                Expression = {
                    Test-Path (Join-Path $_.FullName "bin\node.exe")
                }
            },
            @{
                Name = "node_repl.exe"
                Expression = {
                    Test-Path (Join-Path $_.FullName "bin\node_repl.exe")
                }
            },
            @{
                Name = "manifest.json"
                Expression = {
                    Test-Path (Join-Path $_.FullName "manifest.json")
                }
            } |
        Format-Table -AutoSize

} else {
    Write-Host "Runtime root does not exist." -ForegroundColor Yellow
}

Write-Host "`n========== 5. PACKAGED CUA SOURCE ==========" -ForegroundColor Cyan

if ($null -ne $pkg) {

    $src = Join-Path $pkg.InstallLocation "app\resources\cua_node"

    Write-Host "Source:"
    Write-Host $src

    [PSCustomObject]@{
        SourceExists = Test-Path $src
        Manifest     = Test-Path (Join-Path $src "manifest.json")
        Node         = Test-Path (Join-Path $src "bin\node.exe")
        NodeRepl     = Test-Path (Join-Path $src "bin\node_repl.exe")
    } | Format-List
}
```

### 我们这一步主要看三个信号

如果再次看到：

```text
没有 ChatGPT.exe --type=renderer
```

同时最新目录类似：

```text
.staging-xxxxxxxxxxxxxxxx-xxxxxx
```

并且：

```text
node.exe       True
node_repl.exe  False
manifest.json  False
```

而第 5 部分安装包官方源是：

```text
SourceExists : True
Manifest     : True
Node         : True
NodeRepl     : True
```

那么几乎可以直接判定：**昨天修复的同一个 runtime relocation/staging bug 又因为新的客户端更新和新的 runtime ID 复发了。**

反过来，如果现在已经存在 `--type=renderer`，或者 runtime 完整，那么就**不能再机械执行 `xcopy /G`**，需要转去检查 Chromium 窗口创建、hidden HWND、GPU/renderer crash 等分支。

**现在先只运行这一段，把 `1～5` 五个区块的完整输出原样贴给我。** 我会根据你这一次实际版本和 runtime ID 给你下一步命令，不沿用昨天的旧 ID。

## Prompt:
2026/9/12 15:59:16

运行结果如下：

[Attachment: 粘贴的文本 (1)(2).txt]

## Response:
2026/9/12 15:59:21

结果已经足够明确：**这次仍然是昨天完全相同的 `cua_node` runtime staging/relocation 故障，只是客户端版本和 runtime ID 又更新了。**

证据链很完整：

- 当前客户端已经从昨天的 `26.903.8094.0` 更新到 **`26.903.9818.0`**，AppX 状态仍为 `Ok`。`粘贴的文本 (1).txt`
- `ChatGPT.exe` 主进程、GPU process、network/storage utility 都起来了，但**没有任何 `--type=renderer` 进程**。`粘贴的文本 (1).txt`
- 所有 ChatGPT 进程的 `MainWindowHandle` 都是 **0**，所以确实没有创建可见主窗口。`粘贴的文本 (1).txt`
- 新版本正在创建新的 runtime ID：**`e7fe122ad3cbcd58`**，但 staging 目录只有 `node.exe`，`node_repl.exe=False`、`manifest.json=False`。`粘贴的文本 (1).txt`
- 与此同时，`26.903.9818.0` 安装包中的官方 `cua_node` 源是完整的，三个关键文件全部存在。`粘贴的文本 (1).txt`

所以当前因果链基本已经锁死：

```text
26.903.9818.0 更新
        ↓
需要新 cua_node runtime
e7fe122ad3cbcd58
        ↓
自动 staging / relocation 再次失败
        ↓
.staging-e7fe122ad3cbcd58-JQe5Sm
node.exe       ✓
node_repl.exe  ✗
manifest.json  ✗
        ↓
renderer 未启动
        ↓
MainWindowHandle = 0
        ↓
ChatGPT 有进程，但无 GUI
```

另外日志中的这个：

```text
c:\Users\24503\.trae\extensions\openai.chatgpt-...\codex.exe
```

是 **Trae 扩展自己的 Codex 进程**，不是本次 Windows ChatGPT 客户端故障链的一部分，不要杀它，也不要修改它。`粘贴的文本 (1).txt`

## 第 2 步：停止 ChatGPT，并锁定这次的新 runtime

继续使用刚才的**普通 PowerShell**。

先执行：

```powershell
Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Stop-Process -Force

Start-Sleep -Seconds 2

Get-Process ChatGPT -ErrorAction SilentlyContinue
```

最后一条正常应当**没有任何输出**。

然后执行这一整段：

```powershell
$pkg = Get-AppxPackage -Name OpenAI.Codex

$src = Join-Path $pkg.InstallLocation "app\resources\cua_node"

$runtimeRoot =
    "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

$latestStaging =
    Get-ChildItem $runtimeRoot -Force -Directory |
    Where-Object { $_.Name -like ".staging-*" } |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

if ($latestStaging.Name -match '^\.staging-([0-9a-f]+)-') {
    $runtimeId = $Matches[1]
} else {
    throw "Unable to identify runtime ID."
}

$dst = Join-Path $runtimeRoot $runtimeId

Write-Host "`n========== TARGET ==========" -ForegroundColor Cyan

Write-Host "Package version :" $pkg.Version
Write-Host "Staging         :" $latestStaging.Name
Write-Host "Runtime ID      :" $runtimeId
Write-Host "Source          :" $src
Write-Host "Destination     :" $dst
Write-Host "Destination exists:" (Test-Path $dst)

if ([string]$pkg.Version -ne "26.903.9818.0") {
    throw "Package version changed unexpectedly. Stop here."
}

if ($runtimeId -ne "e7fe122ad3cbcd58") {
    throw "Runtime ID changed unexpectedly. Stop here."
}

[PSCustomObject]@{
    SourceManifest = Test-Path (Join-Path $src "manifest.json")
    SourceNode     = Test-Path (Join-Path $src "bin\node.exe")
    SourceNodeRepl = Test-Path (Join-Path $src "bin\node_repl.exe")
} | Format-List
```

按你刚才的诊断结果，预期应看到大致：

```text
Package version : 26.903.9818.0
Staging         : .staging-e7fe122ad3cbcd58-JQe5Sm
Runtime ID      : e7fe122ad3cbcd58
Destination     : ...\e7fe122ad3cbcd58
Destination exists: False

SourceManifest : True
SourceNode     : True
SourceNodeRepl : True
```

**这一阶段仍然没有修改任何 runtime 文件。**

如果这里没有抛出 `Package version changed unexpectedly` 或 `Runtime ID changed unexpectedly`，那么我们就可以直接执行和昨天同样、已经在你这台机器上验证成功的 **`xcopy /G` 重建**。

把这一段输出贴给我，我下一步直接给你本次 `e7fe122ad3cbcd58` 的修复命令。

## Prompt:
2026/9/12 16:00:30

运行结果如下：
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\24503> Get-Process ChatGPT -ErrorAction SilentlyContinue |
>>     Stop-Process -Force
PS C:\Users\24503>
PS C:\Users\24503> Start-Sleep -Seconds 2
PS C:\Users\24503>
PS C:\Users\24503> Get-Process ChatGPT -ErrorAction SilentlyContinue
PS C:\Users\24503> $pkg = Get-AppxPackage -Name OpenAI.Codex
PS C:\Users\24503>
PS C:\Users\24503> $src = Join-Path $pkg.InstallLocation "app\resources\cua_node"
PS C:\Users\24503>
PS C:\Users\24503> $runtimeRoot =
>>     "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"
PS C:\Users\24503>
PS C:\Users\24503> $latestStaging =
>>     Get-ChildItem $runtimeRoot -Force -Directory |
>>     Where-Object { $_.Name -like ".staging-*" } |
>>     Sort-Object LastWriteTime -Descending |
>>     Select-Object -First 1
PS C:\Users\24503>
PS C:\Users\24503> if ($latestStaging.Name -match '^\.staging-([0-9a-f]+)-') {
>>     $runtimeId = $Matches[1]
>> } else {
>>     throw "Unable to identify runtime ID."
>> }
PS C:\Users\24503>
PS C:\Users\24503> $dst = Join-Path $runtimeRoot $runtimeId
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== TARGET ==========" -ForegroundColor Cyan

========== TARGET ==========
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Package version :" $pkg.Version
Package version : 26.903.9818.0
PS C:\Users\24503> Write-Host "Staging         :" $latestStaging.Name
Staging         : .staging-e7fe122ad3cbcd58-JQe5Sm
PS C:\Users\24503> Write-Host "Runtime ID      :" $runtimeId
Runtime ID      : e7fe122ad3cbcd58
PS C:\Users\24503> Write-Host "Source          :" $src
Source          : C:\Program Files\WindowsApps\OpenAI.Codex_26.903.9818.0_x64__2p2nqsd0c76g0\app\resources\cua_node
PS C:\Users\24503> Write-Host "Destination     :" $dst
Destination     : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\e7fe122ad3cbcd58
PS C:\Users\24503> Write-Host "Destination exists:" (Test-Path $dst)
Destination exists: False
PS C:\Users\24503>
PS C:\Users\24503> if ([string]$pkg.Version -ne "26.903.9818.0") {
>>     throw "Package version changed unexpectedly. Stop here."
>> }
PS C:\Users\24503>
PS C:\Users\24503> if ($runtimeId -ne "e7fe122ad3cbcd58") {
>>     throw "Runtime ID changed unexpectedly. Stop here."
>> }
PS C:\Users\24503>
PS C:\Users\24503> [PSCustomObject]@{
>>     SourceManifest = Test-Path (Join-Path $src "manifest.json")
>>     SourceNode     = Test-Path (Join-Path $src "bin\node.exe")
>>     SourceNodeRepl = Test-Path (Join-Path $src "bin\node_repl.exe")
>> } | Format-List


SourceManifest : True
SourceNode     : True
SourceNodeRepl : True



PS C:\Users\24503>

## Response:
2026/9/12 16:00:40

这一步完全符合预期，已经可以安全进入修复阶段：

- ChatGPT 进程已经全部停止；
- 当前版本仍是 `26.903.9818.0`；
- 新 runtime ID 确认为 `e7fe122ad3cbcd58`；
- 正式目标目录目前不存在；
- 当前 MSIX 安装包中的 `manifest.json`、`node.exe`、`node_repl.exe` 三个关键源文件全部完整。

因此现在执行与昨天相同、且已经在你这台机器上验证有效的 **`xcopy /G` 重建 runtime**。

## 第 3 步：重建 `e7fe122ad3cbcd58`

继续在**当前这个 PowerShell 窗口**执行：

```powershell
Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

if (Test-Path $dst) {

    $backup =
        "$dst.broken-" +
        (Get-Date -Format "yyyyMMdd-HHmmss")

    Move-Item `
        -LiteralPath $dst `
        -Destination $backup

    Write-Host "Existing runtime moved to:"
    Write-Host $backup
}

New-Item `
    -ItemType Directory `
    -Path $dst `
    -Force |
    Out-Null

& "$env:SystemRoot\System32\xcopy.exe" `
    "$src\*" `
    "$dst\" `
    /E /I /H /Y /G /Q

$copyExit = $LASTEXITCODE

Write-Host "`nxcopy exit code:"
Write-Host $copyExit

if ($copyExit -gt 1) {
    throw "xcopy failed with exit code $copyExit."
}
```

注意：**现在不要删除**

```text
.staging-e7fe122ad3cbcd58-JQe5Sm
```

我们暂时保留失败现场。

正常情况下应该看到类似：

```text
复制了 xxxx 个文件

xcopy exit code:
0
```

文件数量不要求和昨天一致，因为 `26.903.9818.0` 的 bundled runtime 内容可能已经变化。

## 第 4 步：验证重建结果

如果上一步 `exit code = 0`，紧接着执行：

```powershell
Write-Host "`n========== RUNTIME VERIFY ==========" -ForegroundColor Cyan

[PSCustomObject]@{
    PackageVersion = $pkg.Version
    RuntimeID      = $runtimeId
    Manifest       = Test-Path (Join-Path $dst "manifest.json")
    Node           = Test-Path (Join-Path $dst "bin\node.exe")
    NodeRepl       = Test-Path (Join-Path $dst "bin\node_repl.exe")
} | Format-List

Write-Host "`n========== HASH VERIFY ==========" -ForegroundColor Cyan

$checkFiles = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($file in $checkFiles) {

    $srcHash =
        (Get-FileHash `
            (Join-Path $src $file) `
            -Algorithm SHA256).Hash

    $dstHash =
        (Get-FileHash `
            (Join-Path $dst $file) `
            -Algorithm SHA256).Hash

    [PSCustomObject]@{
        File    = $file
        Match   = ($srcHash -eq $dstHash)
        SrcHash = $srcHash
        DstHash = $dstHash
    }
}
```

我们要求看到：

```text
PackageVersion : 26.903.9818.0
RuntimeID      : e7fe122ad3cbcd58
Manifest       : True
Node           : True
NodeRepl       : True
```

以及三个文件全部：

```text
Match = True
```

如果这些全部通过，**不要做其他修改，也不要清理 `.staging-*`**。直接：

**Windows 开始菜单 → ChatGPT**

正常启动一次。

如果窗口恢复，我们再做最后一步进程树和 `MainWindowHandle` 验证，把这一次故障完整闭环。

现在先执行上面的**第 3、4 步**，把 `xcopy exit code`、`RUNTIME VERIFY` 和 `HASH VERIFY` 的结果贴给我。

## Prompt:
2026/9/12 16:01:29

运行第一份代码，出现如下问题：
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\24503> Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

========== REBUILD RUNTIME ==========
PS C:\Users\24503>
PS C:\Users\24503> if (Test-Path $dst) {
>>
>>     $backup =
>>         "$dst.broken-" +
>>         (Get-Date -Format "yyyyMMdd-HHmmss")
>>
>>     Move-Item `
>>         -LiteralPath $dst `
>>         -Destination $backup
>>
>>     Write-Host "Existing runtime moved to:"
>>     Write-Host $backup
>> }
Test-Path : 无法将参数绑定到参数“Path”，因为该参数是空值。
所在位置 行:1 字符: 15
+ if (Test-Path $dst) {
+               ~~~~
    + CategoryInfo          : InvalidData: (:) [Test-Path]，ParameterBindingValidationException
    + FullyQualifiedErrorId : ParameterArgumentValidationErrorNullNotAllowed,Microsoft.PowerShell.Commands.TestPathCom
   mand

PS C:\Users\24503>
PS C:\Users\24503> New-Item `
>>     -ItemType Directory `
>>     -Path $dst `
>>     -Force |
>>     Out-Null
New-Item : 无法将参数绑定到参数“Path”，因为该参数是空值。
所在位置 行:3 字符: 11
+     -Path $dst `
+           ~~~~
    + CategoryInfo          : InvalidData: (:) [New-Item]，ParameterBindingValidationException
    + FullyQualifiedErrorId : ParameterArgumentValidationErrorNullNotAllowed,Microsoft.PowerShell.Commands.NewItemComm
   and

PS C:\Users\24503>
PS C:\Users\24503> & "$env:SystemRoot\System32\xcopy.exe" `
>>     "$src\*" `
>>     "$dst\" `
>>     /E /I /H /Y /G /Q
不能执行循环复制
复制了 0 个文件
PS C:\Users\24503>
PS C:\Users\24503> $copyExit = $LASTEXITCODE
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nxcopy exit code:"

xcopy exit code:
PS C:\Users\24503> Write-Host $copyExit
4
PS C:\Users\24503>
PS C:\Users\24503> if ($copyExit -gt 1) {
>>     throw "xcopy failed with exit code $copyExit."
>> }
xcopy failed with exit code 4.
所在位置 行:2 字符: 5
+     throw "xcopy failed with exit code $copyExit."
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (xcopy failed with exit code 4.:String) [], RuntimeException
    + FullyQualifiedErrorId : xcopy failed with exit code 4.

PS C:\Users\24503>

## Response:
2026/9/12 16:01:41

这个报错不是新的客户端故障，原因已经很明确：**当前 PowerShell 会话中的 `$dst`、很可能连 `$src` 等变量都已经丢失了。**

从报错可以直接看出：

```text
Test-Path $dst
→ Path 参数是空值

New-Item -Path $dst
→ Path 参数是空值
```

随后 `xcopy` 实际拿到的是错误的空路径组合，所以返回：

```text
不能执行循环复制
复制了 0 个文件
exit code = 4
```

**这次失败没有复制任何文件，也没有破坏 runtime。** 先不要运行后面的 HASH VERIFY。

最可能的原因是：你执行上一轮诊断后，重新打开了一个 PowerShell 窗口。PowerShell 的 `$pkg`、`$src`、`$runtimeId`、`$dst` 等变量只存在于创建它们的那个会话里，新窗口不会继承。

这次不再依赖前一个 PowerShell 会话。我们用一段**从零重新获取所有变量 + 严格检查 + 执行复制**的完整命令。

### 第 3 步重新执行：重新初始化并重建 runtime

保持 ChatGPT 关闭，在当前普通 PowerShell 中**完整执行下面一整段**：

```powershell
$ErrorActionPreference = "Stop"

$expectedVersion   = "26.903.9818.0"
$expectedRuntimeId = "e7fe122ad3cbcd58"

Write-Host "`n========== INITIALIZE ==========" -ForegroundColor Cyan

# 1. 获取当前安装包
$pkg = Get-AppxPackage -Name OpenAI.Codex

if ($null -eq $pkg) {
    throw "OpenAI.Codex package not found."
}

if ([string]$pkg.Version -ne $expectedVersion) {
    throw "Package version changed. Expected $expectedVersion, found $($pkg.Version). Stop here."
}

# 2. 重新定义源目录和 runtime 根目录
$src = Join-Path $pkg.InstallLocation "app\resources\cua_node"
$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

if (-not (Test-Path -LiteralPath $src)) {
    throw "Packaged cua_node source does not exist: $src"
}

if (-not (Test-Path -LiteralPath $runtimeRoot)) {
    throw "Runtime root does not exist: $runtimeRoot"
}

# 3. 重新定位失败的 staging
$latestStaging =
    Get-ChildItem -LiteralPath $runtimeRoot -Force -Directory |
    Where-Object { $_.Name -like ".staging-$expectedRuntimeId-*" } |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

if ($null -eq $latestStaging) {
    throw "Expected staging directory for $expectedRuntimeId was not found."
}

if ($latestStaging.Name -notmatch '^\.staging-([0-9a-f]+)-') {
    throw "Unable to parse runtime ID from staging directory."
}

$runtimeId = $Matches[1]

if ($runtimeId -ne $expectedRuntimeId) {
    throw "Runtime ID mismatch. Expected $expectedRuntimeId, found $runtimeId."
}

# 4. 重新定义正式目标目录
$dst = Join-Path $runtimeRoot $runtimeId

Write-Host "Package version :" $pkg.Version
Write-Host "Runtime ID      :" $runtimeId
Write-Host "Staging         :" $latestStaging.FullName
Write-Host "Source          :" $src
Write-Host "Destination     :" $dst
Write-Host "Destination exists:" (Test-Path -LiteralPath $dst)

# 5. 在复制前再次验证安装包源文件完整
$requiredFiles = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($file in $requiredFiles) {
    $path = Join-Path $src $file

    if (-not (Test-Path -LiteralPath $path)) {
        throw "Required source file is missing: $path"
    }
}

Write-Host "`nSource validation: PASS" -ForegroundColor Green

Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

# 理论上现在目标目录不存在。
# 如果意外存在，不覆盖，先备份。
if (Test-Path -LiteralPath $dst) {

    $backup =
        "$dst.broken-" +
        (Get-Date -Format "yyyyMMdd-HHmmss")

    Move-Item `
        -LiteralPath $dst `
        -Destination $backup

    Write-Host "Existing destination backed up to:"
    Write-Host $backup
}

New-Item `
    -ItemType Directory `
    -Path $dst `
    -Force |
    Out-Null

# 防止任何异常情况下源、目标路径解析错误
$resolvedSrc = (Resolve-Path -LiteralPath $src).Path
$resolvedDst = (Resolve-Path -LiteralPath $dst).Path

if ($resolvedSrc -eq $resolvedDst) {
    throw "Source and destination unexpectedly resolve to the same directory."
}

Write-Host "Resolved source      :" $resolvedSrc
Write-Host "Resolved destination :" $resolvedDst

& "$env:SystemRoot\System32\xcopy.exe" `
    "$resolvedSrc\*" `
    "$resolvedDst\" `
    /E /I /H /Y /G /Q

$copyExit = $LASTEXITCODE

Write-Host "`nxcopy exit code:"
Write-Host $copyExit

if ($copyExit -gt 1) {
    throw "xcopy failed with exit code $copyExit."
}

Write-Host "`nRuntime copy completed successfully." -ForegroundColor Green
```

这一次我把脚本改得更保守了：它会重新建立全部变量，而且在真正执行 `xcopy` 之前会锁定：

```text
客户端版本 = 26.903.9818.0
runtime ID = e7fe122ad3cbcd58
```

还会确认三个官方源文件确实存在，并用 `Resolve-Path` 确认源、目标不是同一个目录。

### 正常结果

前半部分应看到：

```text
Package version : 26.903.9818.0
Runtime ID      : e7fe122ad3cbcd58
Staging         : ...\.staging-e7fe122ad3cbcd58-JQe5Sm
Source          : ...\app\resources\cua_node
Destination     : ...\e7fe122ad3cbcd58

Source validation: PASS
```

然后：

```text
复制了 xxxx 个文件

xcopy exit code:
0

Runtime copy completed successfully.
```

**如果中间出现任何 `throw`，就停在那里，不要继续执行其他命令。**

先执行这一整段，把从 `INITIALIZE` 到 `xcopy exit code` 的输出发给我。复制成功以后，我们再单独做 SHA-256 校验和启动验证。

## Prompt:
2026/9/12 16:06:36

运行结果如下：
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\24503> Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

========== REBUILD RUNTIME ==========
PS C:\Users\24503>
PS C:\Users\24503> if (Test-Path $dst) {
>>
>>     $backup =
>>         "$dst.broken-" +
>>         (Get-Date -Format "yyyyMMdd-HHmmss")
>>
>>     Move-Item `
>>         -LiteralPath $dst `
>>         -Destination $backup
>>
>>     Write-Host "Existing runtime moved to:"
>>     Write-Host $backup
>> }
Test-Path : 无法将参数绑定到参数“Path”，因为该参数是空值。
所在位置 行:1 字符: 15
+ if (Test-Path $dst) {
+               ~~~~
    + CategoryInfo          : InvalidData: (:) [Test-Path]，ParameterBindingValidationException
    + FullyQualifiedErrorId : ParameterArgumentValidationErrorNullNotAllowed,Microsoft.PowerShell.Commands.TestPathCom
   mand

PS C:\Users\24503>
PS C:\Users\24503> New-Item `
>>     -ItemType Directory `
>>     -Path $dst `
>>     -Force |
>>     Out-Null
New-Item : 无法将参数绑定到参数“Path”，因为该参数是空值。
所在位置 行:3 字符: 11
+     -Path $dst `
+           ~~~~
    + CategoryInfo          : InvalidData: (:) [New-Item]，ParameterBindingValidationException
    + FullyQualifiedErrorId : ParameterArgumentValidationErrorNullNotAllowed,Microsoft.PowerShell.Commands.NewItemComm
   and

PS C:\Users\24503>
PS C:\Users\24503> & "$env:SystemRoot\System32\xcopy.exe" `
>>     "$src\*" `
>>     "$dst\" `
>>     /E /I /H /Y /G /Q
不能执行循环复制
复制了 0 个文件
PS C:\Users\24503>
PS C:\Users\24503> $copyExit = $LASTEXITCODE
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nxcopy exit code:"

xcopy exit code:
PS C:\Users\24503> Write-Host $copyExit
4
PS C:\Users\24503>
PS C:\Users\24503> if ($copyExit -gt 1) {
>>     throw "xcopy failed with exit code $copyExit."
>> }
xcopy failed with exit code 4.
所在位置 行:2 字符: 5
+     throw "xcopy failed with exit code $copyExit."
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (xcopy failed with exit code 4.:String) [], RuntimeException
    + FullyQualifiedErrorId : xcopy failed with exit code 4.

PS C:\Users\24503> $ErrorActionPreference = "Stop"
PS C:\Users\24503>
PS C:\Users\24503> $expectedVersion   = "26.903.9818.0"
PS C:\Users\24503> $expectedRuntimeId = "e7fe122ad3cbcd58"
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== INITIALIZE ==========" -ForegroundColor Cyan

========== INITIALIZE ==========
PS C:\Users\24503>
PS C:\Users\24503> # 1. 获取当前安装包
PS C:\Users\24503> $pkg = Get-AppxPackage -Name OpenAI.Codex
PS C:\Users\24503>
PS C:\Users\24503> if ($null -eq $pkg) {
>>     throw "OpenAI.Codex package not found."
>> }
PS C:\Users\24503>
PS C:\Users\24503> if ([string]$pkg.Version -ne $expectedVersion) {
>>     throw "Package version changed. Expected $expectedVersion, found $($pkg.Version). Stop here."
>> }
PS C:\Users\24503>
PS C:\Users\24503> # 2. 重新定义源目录和 runtime 根目录
PS C:\Users\24503> $src = Join-Path $pkg.InstallLocation "app\resources\cua_node"
PS C:\Users\24503> $runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"
PS C:\Users\24503>
PS C:\Users\24503> if (-not (Test-Path -LiteralPath $src)) {
>>     throw "Packaged cua_node source does not exist: $src"
>> }
PS C:\Users\24503>
PS C:\Users\24503> if (-not (Test-Path -LiteralPath $runtimeRoot)) {
>>     throw "Runtime root does not exist: $runtimeRoot"
>> }
PS C:\Users\24503>
PS C:\Users\24503> # 3. 重新定位失败的 staging
PS C:\Users\24503> $latestStaging =
>>     Get-ChildItem -LiteralPath $runtimeRoot -Force -Directory |
>>     Where-Object { $_.Name -like ".staging-$expectedRuntimeId-*" } |
>>     Sort-Object LastWriteTime -Descending |
>>     Select-Object -First 1
PS C:\Users\24503>
PS C:\Users\24503> if ($null -eq $latestStaging) {
>>     throw "Expected staging directory for $expectedRuntimeId was not found."
>> }
PS C:\Users\24503>
PS C:\Users\24503> if ($latestStaging.Name -notmatch '^\.staging-([0-9a-f]+)-') {
>>     throw "Unable to parse runtime ID from staging directory."
>> }
PS C:\Users\24503>
PS C:\Users\24503> $runtimeId = $Matches[1]
PS C:\Users\24503>
PS C:\Users\24503> if ($runtimeId -ne $expectedRuntimeId) {
>>     throw "Runtime ID mismatch. Expected $expectedRuntimeId, found $runtimeId."
>> }
PS C:\Users\24503>
PS C:\Users\24503> # 4. 重新定义正式目标目录
PS C:\Users\24503> $dst = Join-Path $runtimeRoot $runtimeId
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Package version :" $pkg.Version
Package version : 26.903.9818.0
PS C:\Users\24503> Write-Host "Runtime ID      :" $runtimeId
Runtime ID      : e7fe122ad3cbcd58
PS C:\Users\24503> Write-Host "Staging         :" $latestStaging.FullName
Staging         : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\.staging-e7fe122ad3cbcd58-JQe5Sm
PS C:\Users\24503> Write-Host "Source          :" $src
Source          : C:\Program Files\WindowsApps\OpenAI.Codex_26.903.9818.0_x64__2p2nqsd0c76g0\app\resources\cua_node
PS C:\Users\24503> Write-Host "Destination     :" $dst
Destination     : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\e7fe122ad3cbcd58
PS C:\Users\24503> Write-Host "Destination exists:" (Test-Path -LiteralPath $dst)
Destination exists: False
PS C:\Users\24503>
PS C:\Users\24503> # 5. 在复制前再次验证安装包源文件完整
PS C:\Users\24503> $requiredFiles = @(
>>     "manifest.json",
>>     "bin\node.exe",
>>     "bin\node_repl.exe"
>> )
PS C:\Users\24503>
PS C:\Users\24503> foreach ($file in $requiredFiles) {
>>     $path = Join-Path $src $file
>>
>>     if (-not (Test-Path -LiteralPath $path)) {
>>         throw "Required source file is missing: $path"
>>     }
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nSource validation: PASS" -ForegroundColor Green

Source validation: PASS
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

========== REBUILD RUNTIME ==========
PS C:\Users\24503>
PS C:\Users\24503> # 理论上现在目标目录不存在。
PS C:\Users\24503> # 如果意外存在，不覆盖，先备份。
PS C:\Users\24503> if (Test-Path -LiteralPath $dst) {
>>
>>     $backup =
>>         "$dst.broken-" +
>>         (Get-Date -Format "yyyyMMdd-HHmmss")
>>
>>     Move-Item `
>>         -LiteralPath $dst `
>>         -Destination $backup
>>
>>     Write-Host "Existing destination backed up to:"
>>     Write-Host $backup
>> }
PS C:\Users\24503>
PS C:\Users\24503> New-Item `
>>     -ItemType Directory `
>>     -Path $dst `
>>     -Force |
>>     Out-Null
PS C:\Users\24503>
PS C:\Users\24503> # 防止任何异常情况下源、目标路径解析错误
PS C:\Users\24503> $resolvedSrc = (Resolve-Path -LiteralPath $src).Path
PS C:\Users\24503> $resolvedDst = (Resolve-Path -LiteralPath $dst).Path
PS C:\Users\24503>
PS C:\Users\24503> if ($resolvedSrc -eq $resolvedDst) {
>>     throw "Source and destination unexpectedly resolve to the same directory."
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Resolved source      :" $resolvedSrc
Resolved source      : C:\Program Files\WindowsApps\OpenAI.Codex_26.903.9818.0_x64__2p2nqsd0c76g0\app\resources\cua_node
PS C:\Users\24503> Write-Host "Resolved destination :" $resolvedDst
Resolved destination : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\e7fe122ad3cbcd58
PS C:\Users\24503>
PS C:\Users\24503> & "$env:SystemRoot\System32\xcopy.exe" `
>>     "$resolvedSrc\*" `
>>     "$resolvedDst\" `
>>     /E /I /H /Y /G /Q
复制了 4051 个文件
PS C:\Users\24503>
PS C:\Users\24503> $copyExit = $LASTEXITCODE
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nxcopy exit code:"

xcopy exit code:
PS C:\Users\24503> Write-Host $copyExit
0
PS C:\Users\24503>
PS C:\Users\24503> if ($copyExit -gt 1) {
>>     throw "xcopy failed with exit code $copyExit."
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nRuntime copy completed successfully." -ForegroundColor Green

Runtime copy completed successfully.
PS C:\Users\24503>

## Response:
2026/9/12 16:06:49

这一轮已经执行成功。关键结果是：

```text
Source validation: PASS
Resolved source      : ...\26.903.9818.0...\cua_node
Resolved destination : ...\e7fe122ad3cbcd58
复制了 4051 个文件
xcopy exit code: 0
Runtime copy completed successfully.
```

因此，新的正式 runtime：

```text
C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\e7fe122ad3cbcd58
```

已经建立完成。前面那次 `$dst = null` 导致的失败复制了 `0` 个文件，没有造成破坏，可以忽略。

现在**先不要启动 ChatGPT，也不要删除 `.staging-*`**。先验证复制结果。

### 第 4 步：关键文件与 SHA-256 校验

继续在**当前 PowerShell 窗口**运行：

```powershell
Write-Host "`n========== RUNTIME VERIFY ==========" -ForegroundColor Cyan

[PSCustomObject]@{
    PackageVersion = $pkg.Version
    RuntimeID      = $runtimeId
    Destination    = $dst
    Manifest       = Test-Path -LiteralPath (Join-Path $dst "manifest.json")
    Node           = Test-Path -LiteralPath (Join-Path $dst "bin\node.exe")
    NodeRepl       = Test-Path -LiteralPath (Join-Path $dst "bin\node_repl.exe")
} | Format-List

Write-Host "`n========== HASH VERIFY ==========" -ForegroundColor Cyan

$checkFiles = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($file in $checkFiles) {

    $srcFile = Join-Path $src $file
    $dstFile = Join-Path $dst $file

    if (-not (Test-Path -LiteralPath $srcFile)) {
        throw "Source file missing: $srcFile"
    }

    if (-not (Test-Path -LiteralPath $dstFile)) {
        throw "Destination file missing: $dstFile"
    }

    $srcHash = (Get-FileHash -LiteralPath $srcFile -Algorithm SHA256).Hash
    $dstHash = (Get-FileHash -LiteralPath $dstFile -Algorithm SHA256).Hash

    [PSCustomObject]@{
        File  = $file
        Match = ($srcHash -eq $dstHash)
    }
}
```

预期必须是：

```text
PackageVersion : 26.903.9818.0
RuntimeID      : e7fe122ad3cbcd58
Manifest       : True
Node           : True
NodeRepl       : True
```

然后：

```text
File                Match
----                -----
manifest.json       True
bin\node.exe        True
bin\node_repl.exe   True
```

### 如果全部通过：第 5 步直接测试 GUI

不要清理任何目录。

直接通过：

**Windows 开始菜单 → ChatGPT**

正常启动。

如果窗口出现，随即在 PowerShell 执行最终验证：

```powershell
Write-Host "`n========== CHATGPT AFTER REPAIR ==========" -ForegroundColor Cyan

Get-CimInstance Win32_Process |
    Where-Object {
        $_.Name -eq "ChatGPT.exe"
    } |
    Select-Object ProcessId,
                  ParentProcessId,
                  CommandLine |
    Format-List

Write-Host "`n========== WINDOW STATE ==========" -ForegroundColor Cyan

Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Select-Object Id,
                  Responding,
                  MainWindowHandle,
                  MainWindowTitle |
    Format-Table -AutoSize
```

我们要闭合两个关键证据：

1. 进程树中重新出现：

```text
--type=renderer
```

2. 至少一个 ChatGPT 进程：

```text
MainWindowHandle
```

从之前的 `0` 变成**非零值**，并通常看到：

```text
MainWindowTitle = ChatGPT
```

先执行 **第 4 步校验**。如果三个 `Match` 都是 `True`，就直接启动 ChatGPT；然后把校验结果以及“窗口是否恢复”告诉我。

## Prompt:
2026/9/12 16:10:15

运行结果如下：

[Attachment: 粘贴的文本 (1)(3).txt]

## Response:
2026/9/12 16:10:23

这次结果里有一个需要区分的点：

**客户端本身已经修复成功，但你前半段的 SHA-256 校验实际上无效。**

前半段再次出现了 `$src`、`$dst` 为空：`Join-Path $dst` 和 `Join-Path $src` 都明确报了空值错误。`粘贴的文本 (1).txt` `粘贴的文本 (1).txt`  
所以后面显示的：

```text
manifest.json      True
bin\node.exe       True
bin\node_repl.exe  True
```

不能当成真正的哈希匹配。这里实际上是 `$srcHash` 和 `$dstHash` 都没有成功赋值，随后两个空值比较产生了误导性的 `True`。`粘贴的文本 (1).txt`

不过，**修复是否成功已经可以从后半段独立确认，而且证据非常明确。**

新的启动状态里已经重新出现多个：

```text
ChatGPT.exe --type=renderer
```

例如 PID `93496`、`153516`、`159412`、`38700`。`粘贴的文本 (1).txt`

更关键的是主进程 PID `12052` 现在已经有：

```text
MainWindowHandle = 41292366
MainWindowTitle  = ChatGPT
Responding       = True
```

`粘贴的文本 (1).txt`

因此这一次故障链已经实质闭环：

```text
26.903.9818.0 更新
        ↓
新 runtime e7fe122ad3cbcd58
        ↓
自动 staging 失败
        ↓
node_repl.exe / manifest.json 未完成
        ↓
renderer 不启动
MainWindowHandle = 0
        ↓
人工 xcopy /G 重建 runtime
4051 files / exit code 0
        ↓
renderer 恢复
        ↓
MainWindowHandle = 41292366
        ↓
GUI 恢复
```

### 再做一次真正独立的最终校验

为了把文件层面的证据也闭合，这次不要依赖任何旧 PowerShell 变量。可以新开一个普通 PowerShell，整段运行：

```powershell
$ErrorActionPreference = "Stop"

$runtimeId = "e7fe122ad3cbcd58"

$pkg = Get-AppxPackage -Name OpenAI.Codex

$src = Join-Path `
    $pkg.InstallLocation `
    "app\resources\cua_node"

$runtimeRoot =
    "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

$dst = Join-Path $runtimeRoot $runtimeId

Write-Host "`n========== FINAL RUNTIME VERIFY ==========" `
    -ForegroundColor Cyan

[PSCustomObject]@{
    PackageVersion = $pkg.Version
    RuntimeID      = $runtimeId
    SourceExists   = Test-Path -LiteralPath $src
    RuntimeExists  = Test-Path -LiteralPath $dst
    Manifest       = Test-Path -LiteralPath `
        (Join-Path $dst "manifest.json")
    Node           = Test-Path -LiteralPath `
        (Join-Path $dst "bin\node.exe")
    NodeRepl       = Test-Path -LiteralPath `
        (Join-Path $dst "bin\node_repl.exe")
} | Format-List

Write-Host "`n========== FINAL HASH VERIFY ==========" `
    -ForegroundColor Cyan

$checkFiles = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($file in $checkFiles) {

    $srcFile = Join-Path $src $file
    $dstFile = Join-Path $dst $file

    if (-not (Test-Path -LiteralPath $srcFile)) {
        throw "Source file missing: $srcFile"
    }

    if (-not (Test-Path -LiteralPath $dstFile)) {
        throw "Runtime file missing: $dstFile"
    }

    $srcHash =
        (Get-FileHash `
            -LiteralPath $srcFile `
            -Algorithm SHA256).Hash

    $dstHash =
        (Get-FileHash `
            -LiteralPath $dstFile `
            -Algorithm SHA256).Hash

    [PSCustomObject]@{
        File  = $file
        Match = ($srcHash -eq $dstHash)
    }
}
```

正常结果应该是：

```text
PackageVersion : 26.903.9818.0
RuntimeID      : e7fe122ad3cbcd58
SourceExists   : True
RuntimeExists  : True
Manifest       : True
Node           : True
NodeRepl       : True
```

以及：

```text
manifest.json       True
bin\node.exe        True
bin\node_repl.exe   True
```

如果这次三项 `Match=True`，就可以正式认定 **26.903.9818.0 本轮故障已经完全修复**。

另外，这已经是你连续多次在**客户端自动更新后，以新的 runtime ID 重复触发完全相同故障**。等这次最终校验完成后，我建议下一步不再重复人工操作，而是把目前已经验证过的流程整理成一个**自检 + 安全修复的一键 PowerShell 脚本**：自动识别版本、staging runtime ID、检测缺失文件，只在故障特征完全匹配时执行 `xcopy /G`，之后自动进行 SHA-256 和启动状态验证。这样以后再更新复发，可以直接处理。

## Prompt:
2026/9/12 16:11:18

我希望，暂停一段时间的 ChatGPT 客户端的更新，如何做

## Response:
2026/9/12 16:11:27

可以，但需要区分“**官方可控的暂停方式**”和“**阻断更新机制的非官方手段**”。我先核对当前 ChatGPT Windows 客户端的更新渠道和 Windows/MSIX 的可控项，再给你一个尽量不影响现有安装、并且能随时恢复更新的方案。

可以。结合你目前这个 **`26.903.9818.0` 更新后反复触发 `cua_node` staging 故障**的情况，我建议暂时把版本冻结一段时间，而不是继续让客户端自动追新版本。

我刚核对了当前 Windows 版的更新机制。需要注意：**ChatGPT/Codex Windows 客户端自身有更新器，同时 Microsoft Store 也可能独立更新 MSIX 包**。Microsoft 官方也明确说明，Store 可以自动更新应用，而部分应用还会自行管理更新。([微软支持](https://support.microsoft.com/en-us/windows/apps/turn-on-automatic-app-updates?utm_source=chatgpt.com)) 当前 Codex/ChatGPT 的公开问题中，也确实记录了 Windows 客户端通过自身的 Store/MSIX updater 检测和部署更新。([GitHub](https://github.com/openai/codex/issues/44601?utm_source=chatgpt.com))

因此，如果你的目标是**这几周先稳定使用当前已经修好的 `26.903.9818.0`，尽量确保它不再自动升级**，建议同时关闭两层更新。

### 第一层：关闭 ChatGPT 自身的 updater

这是目前最有针对性的办法。公开的 `openai/codex` issue 已经验证：

```powershell
CODEX_SPARKLE_ENABLED=false
```

会使 Windows 客户端日志中的 `enableUpdater` 变成 `false`，从而不调用 native Windows updater。([GitHub](https://github.com/openai/codex/issues/33320?utm_source=chatgpt.com))

先**正常退出 ChatGPT**，然后打开普通 PowerShell，不需要管理员权限，运行：

```powershell
[Environment]::SetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    "false",
    "User"
)
```

然后验证：

```powershell
[Environment]::GetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    "User"
)
```

应输出：

```text
false
```

注意，这个变量是**用户级永久环境变量**，不是只对当前 PowerShell 窗口有效的 `$env:...` 临时变量。

设置完成后，我建议你**注销 Windows 后重新登录一次，或者直接重启电脑一次**，这样从开始菜单启动的 ChatGPT 可以确定获得新的用户环境。

这个方法目前属于 **OpenAI GitHub issue 中验证有效的 workaround，不是 OpenAI 正式文档中提供的“暂停更新”设置**。这一点需要明确。不过从现有证据看，它比修改 MSIX 权限、阻断服务、改注册表等方法干净得多，而且完全可逆。([GitHub](https://github.com/openai/codex/issues/41073))

### 第二层：关闭 Microsoft Store 的自动应用更新

单独做第一层还不能百分之百保证版本被冻结，因为 **Microsoft Store 本身仍然可能更新 `OpenAI.Codex`**。

打开：

**Microsoft Store → 右上角头像 → 设置 / Store Settings → App updates（应用更新）**

把：

```text
App updates
```

暂时关闭。

这是 Microsoft 官方提供的 Store 自动更新开关。([微软支持](https://support.microsoft.com/en-us/windows/apps/turn-on-automatic-app-updates?utm_source=chatgpt.com))

但这里有一个代价：**它是 Store 的全局设置，不是只针对 ChatGPT。** 暂停期间，其他 Microsoft Store 应用也不会自动更新。

所以我建议你的组合是：

```text
ChatGPT 自身 updater
    ↓
CODEX_SPARKLE_ENABLED=false
    ↓
关闭

Microsoft Store 自动 App updates
    ↓
关闭

最终结果
    ↓
尽量冻结 OpenAI.Codex 26.903.9818.0
```

这比禁用 `InstallService`、`AppXSvc`、`BITS`、Windows Update 服务，或者改 WindowsApps ACL 要安全得多。**这些系统级服务不要动。**

### 设置完以后，再确认当前版本

重启 Windows 后，可以运行：

```powershell
Get-AppxPackage -Name OpenAI.Codex |
    Select-Object Name, Version, Status
```

目前应保持：

```text
Name    : OpenAI.Codex
Version : 26.903.9818.0
Status  : Ok
```

你现在刚刚修好的：

```text
%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\e7fe122ad3cbcd58
```

也不要再修改。

### 将来想恢复更新

不需要把变量设成 `"true"`；更干净的做法是**直接删除这个用户环境变量**：

```powershell
[Environment]::SetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    $null,
    "User"
)
```

验证：

```powershell
[Environment]::GetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    "User"
)
```

此时应该没有输出。

然后再：

**Microsoft Store → Settings → App updates → On**

最后注销/重新登录或重启 Windows。

---

考虑到你已经连续遇到**更新 → 新 runtime ID → staging 不完整 → `renderer` 不启动 → GUI 消失**，我认为目前最合理的是：**先把 `26.903.9818.0` 固定使用一段时间，等对应 GitHub 问题明确修复后再恢复更新。**

你现在可以先只做第一步：

```powershell
[Environment]::SetEnvironmentVariable("CODEX_SPARKLE_ENABLED","false","User")

[Environment]::GetEnvironmentVariable("CODEX_SPARKLE_ENABLED","User")
```

把输出告诉我；我可以继续带你确认 **ChatGPT 内置 updater 是否确实已经进入 disabled 状态**，然后再关闭 Store 自动更新。

---

**Sources:**

- [Turn on automatic app updates | Microsoft Support](https://support.microsoft.com/en-us/windows/apps/turn-on-automatic-app-updates?utm_source=chatgpt.com)
- [[Windows] In-app updater enqueues a paused Store item it cannot commit: "Preparing the update" hangs forever and the Microsoft Store Update button stops working · Issue #44601 · openai/codex](https://github.com/openai/codex/issues/44601?utm_source=chatgpt.com)

## Prompt:
2026/9/12 16:17:12

![tmpD063.png](https://chatgpt.com/backend-api/estuary/content?id=file_000000003394822fab76718eefe2898d&fn=tmpD063.png&cd=attachment&ts=497001&p=fs&cid=1&sig=3e31da43134cb50ae2744ddb2a689600d63e8863bc355ffe3fb670cc4033a6a7&v=0)

第一步我已操作完成，第二步在Microsoft Store  中的操作无法完成，显示如下，
请问该怎么办

## Response:
2026/9/12 16:17:23

你这个截图与近期 Microsoft Store 的一个已知问题高度一致：**即使当前 Windows 账户本身是管理员，Store 仍可能错误提示“需要管理员权限才能暂停更新”**。2026 年 8 月的 Microsoft Q&A 已出现与截图几乎完全相同的报告，而且发生在管理员账户上；目前更像是 Store 端/UI 权限判断的问题，而不是你操作错误。([Microsoft Learn](https://learn.microsoft.com/en-nz/answers/questions/5972319/microsoft-store-says-you-need-administrator-privil?utm_source=chatgpt.com))

因此，不建议继续折腾这个开关，也**不要**去禁用 Windows Update、BITS、AppXSvc，更不要启用 Windows 隐藏的 Administrator 账户。对你现在的目标，直接使用 Microsoft 官方提供的 **Store 自动更新策略**更稳妥。

### 推荐做法：用管理员 PowerShell 暂时关闭 Store 应用自动更新

Microsoft 官方文档明确给出了对应策略：

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\WindowsStore
AutoDownload = 2
```

其中 `2` 表示**关闭 Store 应用更新的自动下载和安装**；`4` 表示开启自动更新。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com))

先右键 Windows 开始按钮，打开：

**终端（管理员）**  
或  
**Windows PowerShell（管理员）**

确认标题栏中有“管理员”。

然后先执行一个只读检查：

```powershell
Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
    -Name AutoDownload `
    -ErrorAction SilentlyContinue
```

如果没有输出，也正常，说明目前没有这个策略。

然后执行：

```powershell
$storePolicy = "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore"

New-Item `
    -Path $storePolicy `
    -Force |
    Out-Null

New-ItemProperty `
    -Path $storePolicy `
    -Name "AutoDownload" `
    -PropertyType DWord `
    -Value 2 `
    -Force |
    Out-Null

Get-ItemProperty `
    -Path $storePolicy `
    -Name AutoDownload
```

最后应该看到：

```text
AutoDownload : 2
```

再执行：

```powershell
gpupdate /force
```

之后**重启 Windows 一次**。不需要再尝试截图中那个 Store 开关。

### 这样设置后的状态

你现在实际上会形成两层保护：

```text
ChatGPT 自身更新机制
    ↓
CODEX_SPARKLE_ENABLED=false
    ↓
暂停

Microsoft Store 自动应用更新
    ↓
AutoDownload = 2
    ↓
暂停
```

第二层是 **Microsoft 官方的设备级策略**，因此它比当前有 bug 的 Store 图形界面开关更可靠。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com))

但必须注意一个副作用：

> **`AutoDownload=2` 会暂停 Microsoft Store 中所有应用的自动更新，而不仅仅是 ChatGPT。**

所以这是适合你现在“先稳定使用几周”的方案，但不建议永久保持。

### 将来恢复更新

到时候不要把它改成 `4`。为了恢复 Windows 原本由用户/Store 控制的默认行为，更干净的办法是**删除我们新增的策略值**：

以管理员 PowerShell 执行：

```powershell
Remove-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
    -Name "AutoDownload" `
    -ErrorAction SilentlyContinue

gpupdate /force
```

然后恢复你之前设置的 ChatGPT 环境变量：

```powershell
[Environment]::SetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    $null,
    "User"
)
```

再重启 Windows。

官方文档也说明，当该策略处于“未配置”状态时，Store 的更新行为重新交由用户的 Store 设置控制。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com))

**现在建议你先只执行上面的管理员 PowerShell 设置 `AutoDownload=2`，把最后 `Get-ItemProperty` 的输出发给我。** 我们再确认第二层更新冻结是否真正生效。

---

**Sources:**

- [Microsoft Store says "You need administrator privileges to pause updates" even though I am an administrator - Microsoft Q&A](https://learn.microsoft.com/en-nz/answers/questions/5972319/microsoft-store-says-you-need-administrator-privil?utm_source=chatgpt.com)



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)