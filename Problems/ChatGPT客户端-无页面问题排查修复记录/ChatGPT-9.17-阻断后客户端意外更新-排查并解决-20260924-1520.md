# 9.17-阻断后客户端意外更新-排查并解决

**User:** Anonymous  
**Created:** 9/17/2026 21:43:31  
**Updated:** 9/20/2026 16:31:44  
**Exported:** 9/24/2026 15:20:18  
**Link:** [https://chatgpt.com/g/g-p-6a9fc63d99c481919dd12e2056b7a0ef/c/6aabee81-f258-83e8-b8f4-c2310606bb0c](https://chatgpt.com/g/g-p-6a9fc63d99c481919dd12e2056b7a0ef/c/6aabee81-f258-83e8-b8f4-c2310606bb0c)  

## Prompt:
2026/9/17 21:44:00

我的 ChatGPT 客户端刚刚突然自动退出了，之后又打不开了，后台有进程，无界面，这是之前的记录， 
请帮我排查解决

[Attachment: ChatGPT-9.12-ChatGPT客户端打不开解决-20260912-1726(1).md]

## Response:
2026/9/17 21:44:03

我会沿用你上次已经验证过的故障链，但这次先不直接复制 runtime。先确认三个问题：**客户端版本有没有变化、`cua_node` 是否再次出现半成品 staging、以及你之前设置的两层“暂停更新”是否仍然生效**。上次的记录已经证明，这类故障会表现为“有 `ChatGPT.exe`，但没有 renderer、没有窗口”，并且人工重建 runtime 后恢复。

从你这份记录看，我会优先怀疑**同一类 runtime staging/relocation 故障再次复发**，但这次先不直接修。

有两个关键背景：

- 9 月 12 日你已经遇到过完全相同的“`ChatGPT.exe` 有进程、无窗口、无 renderer”的故障，当时确认是 `cua_node` 的 `.staging-*` 不完整，补齐正式 runtime 后 GUI 恢复。`ChatGPT-9.12-ChatGPT客户端打不开解决-20260912-1726.md`
- 你的记录只明确确认了第一层 `CODEX_SPARKLE_ENABLED=false` 已经设置；Microsoft Store 的自动更新开关当时因为权限提示没有成功关闭，后面虽然给出了 `AutoDownload=2` 的替代方案，但记录里没有看到你实际执行成功的确认。`ChatGPT-9.12-ChatGPT客户端打不开解决-20260912-1726.md` `ChatGPT-9.12-ChatGPT客户端打不开解决-20260912-1726.md`

而且这个问题目前并没有消失。**2026 年 9 月 16 日，也就是昨天**，openai/codex 又出现了一个新的 Windows issue：`26.908.9136.0` 更新后，`codex.exe` 和 `cua_node` 都出现 `.staging-* → 正式缓存目录` 的 `rename_staging / EPERM` 失败，并导致桌面端启动失败。([GitHub](https://github.com/openai/codex/issues/45858))

所以这次我们首先确认：**你是不是实际上又被 Microsoft Store 更新了。**

### 现在先不要做这些操作

不要：

- Reset ChatGPT
- 卸载/重装
- 删除 `%LOCALAPPDATA%\OpenAI\Codex`
- 删除任何 `.staging-*`
- 手工复制 `node_repl.exe`
- 修改 WindowsApps 权限

也先**不要结束当前后台的 ChatGPT 进程**。保持“有进程、无窗口”的现场。

### 第 1 步：一次性只读诊断

打开一个**普通 PowerShell**，不是管理员 PowerShell。把下面整段一次性复制进去运行：

```powershell
$ErrorActionPreference = "Continue"

Write-Host "`n========== 1. PACKAGE ==========" -ForegroundColor Cyan

$pkg = Get-AppxPackage -Name OpenAI.Codex

if ($null -eq $pkg) {
    Write-Host "OpenAI.Codex package NOT FOUND" -ForegroundColor Red
}
else {
    $pkg |
        Select-Object Name,
                      Version,
                      Status,
                      PackageFamilyName,
                      InstallLocation |
        Format-List
}

Write-Host "`n========== 2. UPDATE CONTROL ==========" -ForegroundColor Cyan

[PSCustomObject]@{
    CODEX_SPARKLE_ENABLED =
        [Environment]::GetEnvironmentVariable(
            "CODEX_SPARKLE_ENABLED",
            "User"
        )

    Store_AutoDownload =
        (
            Get-ItemProperty `
                -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
                -Name AutoDownload `
                -ErrorAction SilentlyContinue
        ).AutoDownload
} | Format-List

Write-Host "`n========== 3. CHATGPT PROCESS TREE ==========" -ForegroundColor Cyan

$chatProcesses =
    Get-CimInstance Win32_Process |
    Where-Object {
        $_.Name -in @(
            "ChatGPT.exe",
            "codex.exe",
            "node.exe",
            "node_repl.exe",
            "rg.exe"
        )
    }

$chatProcesses |
    Select-Object ProcessId,
                  ParentProcessId,
                  Name,
                  CommandLine |
    Format-List

Write-Host "`n========== 4. RENDERER SUMMARY ==========" -ForegroundColor Cyan

$renderers =
    $chatProcesses |
    Where-Object {
        $_.Name -eq "ChatGPT.exe" -and
        $_.CommandLine -match "--type=renderer"
    }

Write-Host "Renderer count:" @($renderers).Count

Write-Host "`n========== 5. WINDOW STATE ==========" -ForegroundColor Cyan

Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Select-Object Id,
                  Responding,
                  MainWindowHandle,
                  MainWindowTitle,
                  CPU,
                  StartTime |
    Format-Table -AutoSize

Write-Host "`n========== 6. CUA RUNTIME ==========" -ForegroundColor Cyan

$runtimeRoot =
    "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

Write-Host "Runtime root:"
Write-Host $runtimeRoot

if (Test-Path -LiteralPath $runtimeRoot) {

    Get-ChildItem `
        -LiteralPath $runtimeRoot `
        -Force `
        -Directory |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 20 `
        Name,
        LastWriteTime,
        @{
            Name = "node.exe"
            Expression = {
                Test-Path -LiteralPath `
                    (Join-Path $_.FullName "bin\node.exe")
            }
        },
        @{
            Name = "node_repl.exe"
            Expression = {
                Test-Path -LiteralPath `
                    (Join-Path $_.FullName "bin\node_repl.exe")
            }
        },
        @{
            Name = "manifest.json"
            Expression = {
                Test-Path -LiteralPath `
                    (Join-Path $_.FullName "manifest.json")
            }
        } |
    Format-Table -AutoSize
}
else {
    Write-Host "Runtime root does not exist." -ForegroundColor Yellow
}

Write-Host "`n========== 7. CODEX BIN CACHE ==========" -ForegroundColor Cyan

$binRoot =
    "$env:LOCALAPPDATA\OpenAI\Codex\bin"

Write-Host "Bin root:"
Write-Host $binRoot

if (Test-Path -LiteralPath $binRoot) {

    Get-ChildItem `
        -LiteralPath $binRoot `
        -Force `
        -Directory |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 20 `
        Name,
        LastWriteTime,
        @{
            Name = "codex.exe"
            Expression = {
                Test-Path -LiteralPath `
                    (Join-Path $_.FullName "codex.exe")
            }
        },
        @{
            Name = "rg.exe"
            Expression = {
                Test-Path -LiteralPath `
                    (Join-Path $_.FullName "rg.exe")
            }
        } |
    Format-Table -AutoSize
}
else {
    Write-Host "Bin root does not exist." -ForegroundColor Yellow
}

Write-Host "`n========== 8. PACKAGED CUA SOURCE ==========" -ForegroundColor Cyan

if ($null -ne $pkg) {

    $src =
        Join-Path `
            $pkg.InstallLocation `
            "app\resources\cua_node"

    Write-Host "Source:"
    Write-Host $src

    [PSCustomObject]@{
        SourceExists = Test-Path -LiteralPath $src
        Manifest     = Test-Path -LiteralPath `
            (Join-Path $src "manifest.json")
        Node         = Test-Path -LiteralPath `
            (Join-Path $src "bin\node.exe")
        NodeRepl     = Test-Path -LiteralPath `
            (Join-Path $src "bin\node_repl.exe")
    } | Format-List
}
```

把 **1～8 八个区块的完整输出**直接贴给我。

### 我现在重点判断四件事

第一，看版本。

你 9 月 12 日修好时是：

```text
26.903.9818.0
```

如果现在已经变成例如：

```text
26.908.xxxx.x
```

那基本就能确定：**虽然 ChatGPT 自身 updater 被关了，但 Microsoft Store 这一层还是更新了客户端。**

第二，看：

```text
CODEX_SPARKLE_ENABLED : false
Store_AutoDownload    :
```

如果第二项为空，说明你之前实际上**只关闭了 ChatGPT 自身 updater，Store 更新策略没有生效**。

第三，看有没有：

```text
Renderer count: 0
```

同时所有：

```text
MainWindowHandle = 0
```

这就再次符合你之前的 headless startup 特征。

第四，也是最关键的，看最新 `.staging-*`。

如果出现类似：

```text
.staging-xxxxxxxxxxxxxxxx-XXXXXX

node.exe       True
node_repl.exe  False
manifest.json  False
```

而第 8 部分是：

```text
SourceExists : True
Manifest     : True
Node         : True
NodeRepl     : True
```

那么就已经可以非常明确地确认：

> **又是 `cua_node` staging/relocation 没有完成。**

这次我还额外加入了第 7 部分 `CODEX BIN CACHE`，因为昨天的新 issue 表明现在不只是 `cua_node`，**`codex.exe` 自己的 staging/cache 也可能同时发生 `rename_staging / EPERM`**。([GitHub](https://github.com/openai/codex/issues/45858))

所以不要直接照搬 9 月 12 日的旧 `e7fe122ad3cbcd58` 修复命令。**当前版本、runtime ID 和可能受损的组件很可能已经变了。**

你把这一次的八段输出给我，我就可以根据现场直接进入下一步修复。

---

**Sources:**

- [[Windows] Startup fails after a Store update with rename_staging EPERM for CLI and Node runtime · Issue #45858 · openai/codex · GitHub](https://github.com/openai/codex/issues/45858)

## Prompt:
2026/9/17 21:45:37

这是运行后的输出

[Attachment: 粘贴的文本 (1)(5).txt]

## Response:
2026/9/17 21:45:38

结果已经足够明确，可以进入修复，不需要再做 Reset 或重装。你这次的现场与 9 月 12 日的故障特征高度一致，而且关键证据比上次更完整。

已经可以判定故障类型，**这次仍然是 `cua_node` runtime staging 未完成导致的“有进程、无界面”故障**。暂时不需要重装 ChatGPT。

### 这次证据已经闭合

首先，你当前实际运行的客户端已经变成：

```text
OpenAI.Codex_26.911.7940.0
```

主进程 PID `83456` 就来自这个版本。`粘贴的文本 (1).txt`

这说明从 9 月 12 日修复时的 `26.903.9818.0` 到现在，**客户端确实再次发生过版本升级**。

而你之前设置的两层更新控制目前都还存在：

```text
CODEX_SPARKLE_ENABLED : false
Store_AutoDownload    : 2
```

`粘贴的文本 (1).txt`

这意味着一个很重要的问题：**这两项设置并没有保证 `OpenAI.Codex` 的版本真正冻结。** 至于此次更新究竟通过哪个更新路径发生，我们等客户端恢复后再单独排查；现在先解决启动故障。

更关键的是当前 GUI 状态：

```text
Renderer count: 0
```

`粘贴的文本 (1).txt`

并且所有 ChatGPT 进程：

```text
MainWindowHandle = 0
```

主进程虽然 `Responding=True`，甚至累计 CPU 已经很高，但始终没有创建窗口。`粘贴的文本 (1).txt`

真正锁定问题的是这里：

```text
.staging-fcf5d08964516500-cP9p1r
    node.exe       True
    node_repl.exe  False
    manifest.json  False

.staging-fcf5d08964516500-vH7e0k
    node.exe       True
    node_repl.exe  False
    manifest.json  False
```

也就是说，它已经**连续两次尝试构建相同的新 runtime `fcf5d08964516500`，两次都卡在完全相同的位置**。`粘贴的文本 (1).txt`

而当前 `26.911.7940.0` 安装包自己的官方源文件却是完整的：

```text
SourceExists : True
Manifest     : True
Node         : True
NodeRepl     : True
```

`粘贴的文本 (1).txt`

所以当前故障链基本可以确定为：

```text
ChatGPT 26.911.7940.0
        ↓
需要新的 cua_node runtime
fcf5d08964516500
        ↓
自动 staging
        ↓
.staging-fcf5...-vH7e0k
.staging-fcf5...-cP9p1r
        ↓
两次都只完成 node.exe
node_repl.exe / manifest.json 缺失
        ↓
runtime 无法完成部署
        ↓
renderer = 0
        ↓
MainWindowHandle = 0
        ↓
有 ChatGPT.exe，但没有窗口
```

这与你上次的故障机制基本一致。

另外，你输出中的几个：

```text
else : 无法将“else”项识别为 cmdlet...
```

不是客户端故障，是因为 PowerShell 把 `if {...}` 和后续单独粘贴的 `else {...}` 当成了两次独立输入。它**不影响上述诊断结果**。例如 `$pkg` 实际获取成功了，否则最后也不可能得到正确的 `26.911.7940.0` WindowsApps 路径。`粘贴的文本 (1).txt`

---

## 现在直接修复 `fcf5d08964516500`

这次我不再让命令依赖当前 PowerShell 中已有的 `$src`、`$dst` 等变量，避免上次出现“换了 PowerShell 窗口导致变量为空”的情况。

**先不要手动杀进程。**

新开一个**普通 PowerShell，不需要管理员权限**，把下面**整段一次性执行**：

```powershell
$ErrorActionPreference = "Stop"

$expectedVersion   = "26.911.7940.0"
$expectedRuntimeId = "fcf5d08964516500"

Write-Host "`n========== INITIALIZE ==========" -ForegroundColor Cyan

# 获取当前 ChatGPT / Codex MSIX 包
$pkg = Get-AppxPackage -Name OpenAI.Codex

if ($null -eq $pkg) {
    throw "OpenAI.Codex package not found."
}

Write-Host "Package version :" $pkg.Version

if ([string]$pkg.Version -ne $expectedVersion) {
    throw "Package version changed. Expected $expectedVersion, found $($pkg.Version). Stop here."
}

# 定义官方源和 runtime 路径
$src = Join-Path `
    $pkg.InstallLocation `
    "app\resources\cua_node"

$runtimeRoot =
    "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

$dst =
    Join-Path `
        $runtimeRoot `
        $expectedRuntimeId

Write-Host "Source          :" $src
Write-Host "Runtime root    :" $runtimeRoot
Write-Host "Destination     :" $dst

if (-not (Test-Path -LiteralPath $src)) {
    throw "Packaged cua_node source does not exist."
}

if (-not (Test-Path -LiteralPath $runtimeRoot)) {
    throw "Runtime root does not exist."
}

# 确认当前确实存在此次失败的 staging
$staging =
    Get-ChildItem `
        -LiteralPath $runtimeRoot `
        -Force `
        -Directory |
    Where-Object {
        $_.Name -like ".staging-$expectedRuntimeId-*"
    } |
    Sort-Object LastWriteTime -Descending

if (@($staging).Count -eq 0) {
    throw "No staging directory found for runtime $expectedRuntimeId."
}

Write-Host "`nFailed staging directories:" -ForegroundColor Yellow

$staging |
    Select-Object Name, LastWriteTime |
    Format-Table -AutoSize

Write-Host "`n========== SOURCE VALIDATION ==========" -ForegroundColor Cyan

$requiredFiles = @(
    "manifest.json",
    "bin\node.exe",
    "bin\node_repl.exe"
)

foreach ($file in $requiredFiles) {

    $sourceFile =
        Join-Path `
            $src `
            $file

    if (-not (Test-Path -LiteralPath $sourceFile)) {
        throw "Required packaged file is missing: $sourceFile"
    }

    Write-Host "$file : OK"
}

Write-Host "`nSource validation: PASS" -ForegroundColor Green

Write-Host "`n========== STOP CHATGPT ==========" -ForegroundColor Cyan

Get-Process ChatGPT -ErrorAction SilentlyContinue |
    Stop-Process -Force

Start-Sleep -Seconds 2

$remaining =
    Get-Process ChatGPT -ErrorAction SilentlyContinue

if ($null -ne $remaining) {
    throw "Some ChatGPT processes are still running."
}

Write-Host "ChatGPT processes stopped." -ForegroundColor Green

Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

# 正常情况下 fcf5... 正式目录尚不存在。
# 如果意外存在，不覆盖，先保留备份。
if (Test-Path -LiteralPath $dst) {

    $backup =
        "$dst.broken-" +
        (Get-Date -Format "yyyyMMdd-HHmmss")

    Move-Item `
        -LiteralPath $dst `
        -Destination $backup

    Write-Host "Existing runtime backed up to:"
    Write-Host $backup
}

New-Item `
    -ItemType Directory `
    -Path $dst `
    -Force |
    Out-Null

$resolvedSrc =
    (Resolve-Path -LiteralPath $src).Path

$resolvedDst =
    (Resolve-Path -LiteralPath $dst).Path

Write-Host "Resolved source      :" $resolvedSrc
Write-Host "Resolved destination :" $resolvedDst

if ($resolvedSrc -eq $resolvedDst) {
    throw "Source and destination unexpectedly resolve to the same path."
}

& "$env:SystemRoot\System32\xcopy.exe" `
    "$resolvedSrc\*" `
    "$resolvedDst\" `
    /E /I /H /Y /G /Q

$copyExit = $LASTEXITCODE

Write-Host "`nxcopy exit code:" $copyExit

if ($copyExit -gt 1) {
    throw "xcopy failed with exit code $copyExit."
}

Write-Host "Runtime copy completed successfully." -ForegroundColor Green

Write-Host "`n========== FILE VERIFY ==========" -ForegroundColor Cyan

foreach ($file in $requiredFiles) {

    $sourceFile =
        Join-Path `
            $src `
            $file

    $runtimeFile =
        Join-Path `
            $dst `
            $file

    if (-not (Test-Path -LiteralPath $runtimeFile)) {
        throw "Runtime file missing after copy: $runtimeFile"
    }

    $sourceHash =
        (Get-FileHash `
            -LiteralPath $sourceFile `
            -Algorithm SHA256).Hash

    $runtimeHash =
        (Get-FileHash `
            -LiteralPath $runtimeFile `
            -Algorithm SHA256).Hash

    [PSCustomObject]@{
        File  = $file
        Match = ($sourceHash -eq $runtimeHash)
    }
}
```

### 正常情况下应该看到什么

最关键的是：

```text
Package version : 26.911.7940.0
```

然后识别到两个：

```text
.staging-fcf5d08964516500-...
```

接下来：

```text
manifest.json      : OK
bin\node.exe       : OK
bin\node_repl.exe  : OK

Source validation: PASS
```

之后应该复制数千个文件，并出现：

```text
xcopy exit code: 0
Runtime copy completed successfully.
```

最后三个文件必须全部是：

```text
File                Match
----                -----
manifest.json       True
bin\node.exe        True
bin\node_repl.exe   True
```

**暂时不要删除那两个 `.staging-fcf5d08964516500-*` 目录。** 它们是这次失败现场，而且不会妨碍我们人工建立正式的：

```text
%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\fcf5d08964516500
```

脚本成功后，直接从 **Windows 开始菜单启动 ChatGPT**。

如果窗口恢复，说明这一轮已经修复。然后我们再做一次 `renderer + MainWindowHandle` 验证，同时需要认真处理另一个问题：**为什么 `CODEX_SPARKLE_ENABLED=false` 和 `Store_AutoDownload=2` 都存在的情况下，客户端仍然从 `26.903.9818.0` 升到了 `26.911.7940.0`。** 这一次的输出已经证明，之前的“暂停更新”方案并没有真正实现版本冻结。

## Prompt:
2026/9/17 21:53:14

Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\24503> $ErrorActionPreference = "Stop"
PS C:\Users\24503>
PS C:\Users\24503> $expectedVersion   = "26.911.7940.0"
PS C:\Users\24503> $expectedRuntimeId = "fcf5d08964516500"
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== INITIALIZE ==========" -ForegroundColor Cyan

========== INITIALIZE ==========
PS C:\Users\24503>
PS C:\Users\24503> # 获取当前 ChatGPT / Codex MSIX 包
PS C:\Users\24503> $pkg = Get-AppxPackage -Name OpenAI.Codex
PS C:\Users\24503>
PS C:\Users\24503> if ($null -eq $pkg) {
>>     throw "OpenAI.Codex package not found."
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Package version :" $pkg.Version
Package version : 26.911.7940.0
PS C:\Users\24503>
PS C:\Users\24503> if ([string]$pkg.Version -ne $expectedVersion) {
>>     throw "Package version changed. Expected $expectedVersion, found $($pkg.Version). Stop here."
>> }
PS C:\Users\24503>
PS C:\Users\24503> # 定义官方源和 runtime 路径
PS C:\Users\24503> $src = Join-Path `
>>     $pkg.InstallLocation `
>>     "app\resources\cua_node"
PS C:\Users\24503>
PS C:\Users\24503> $runtimeRoot =
>>     "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"
PS C:\Users\24503>
PS C:\Users\24503> $dst =
>>     Join-Path `
>>         $runtimeRoot `
>>         $expectedRuntimeId
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Source          :" $src
Source          : C:\Program Files\WindowsApps\OpenAI.Codex_26.911.7940.0_x64__2p2nqsd0c76g0\app\resources\cua_node
PS C:\Users\24503> Write-Host "Runtime root    :" $runtimeRoot
Runtime root    : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node
PS C:\Users\24503> Write-Host "Destination     :" $dst
Destination     : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\fcf5d08964516500
PS C:\Users\24503>
PS C:\Users\24503> if (-not (Test-Path -LiteralPath $src)) {
>>     throw "Packaged cua_node source does not exist."
>> }
PS C:\Users\24503>
PS C:\Users\24503> if (-not (Test-Path -LiteralPath $runtimeRoot)) {
>>     throw "Runtime root does not exist."
>> }
PS C:\Users\24503>
PS C:\Users\24503> # 确认当前确实存在此次失败的 staging
PS C:\Users\24503> $staging =
>>     Get-ChildItem `
>>         -LiteralPath $runtimeRoot `
>>         -Force `
>>         -Directory |
>>     Where-Object {
>>         $_.Name -like ".staging-$expectedRuntimeId-*"
>>     } |
>>     Sort-Object LastWriteTime -Descending
PS C:\Users\24503>
PS C:\Users\24503> if (@($staging).Count -eq 0) {
>>     throw "No staging directory found for runtime $expectedRuntimeId."
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nFailed staging directories:" -ForegroundColor Yellow

Failed staging directories:
PS C:\Users\24503>
PS C:\Users\24503> $staging |
>>     Select-Object Name, LastWriteTime |
>>     Format-Table -AutoSize

Name                             LastWriteTime
----                             -------------
.staging-fcf5d08964516500-cP9p1r 2026/9/17 21:38:25
.staging-fcf5d08964516500-vH7e0k 2026/9/17 21:37:20


PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== SOURCE VALIDATION ==========" -ForegroundColor Cyan

========== SOURCE VALIDATION ==========
PS C:\Users\24503>
PS C:\Users\24503> $requiredFiles = @(
>>     "manifest.json",
>>     "bin\node.exe",
>>     "bin\node_repl.exe"
>> )
PS C:\Users\24503>
PS C:\Users\24503> foreach ($file in $requiredFiles) {
>>
>>     $sourceFile =
>>         Join-Path `
>>             $src `
>>             $file
>>
>>     if (-not (Test-Path -LiteralPath $sourceFile)) {
>>         throw "Required packaged file is missing: $sourceFile"
>>     }
>>
>>     Write-Host "$file : OK"
>> }
manifest.json : OK
bin\node.exe : OK
bin\node_repl.exe : OK
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nSource validation: PASS" -ForegroundColor Green

Source validation: PASS
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== STOP CHATGPT ==========" -ForegroundColor Cyan

========== STOP CHATGPT ==========
PS C:\Users\24503>
PS C:\Users\24503> Get-Process ChatGPT -ErrorAction SilentlyContinue |
>>     Stop-Process -Force
PS C:\Users\24503>
PS C:\Users\24503> Start-Sleep -Seconds 2
PS C:\Users\24503>
PS C:\Users\24503> $remaining =
>>     Get-Process ChatGPT -ErrorAction SilentlyContinue
PS C:\Users\24503>
PS C:\Users\24503> if ($null -ne $remaining) {
>>     throw "Some ChatGPT processes are still running."
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "ChatGPT processes stopped." -ForegroundColor Green
ChatGPT processes stopped.
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== REBUILD RUNTIME ==========" -ForegroundColor Cyan

========== REBUILD RUNTIME ==========
PS C:\Users\24503>
PS C:\Users\24503> # 正常情况下 fcf5... 正式目录尚不存在。
PS C:\Users\24503> # 如果意外存在，不覆盖，先保留备份。
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
>>     Write-Host "Existing runtime backed up to:"
>>     Write-Host $backup
>> }
PS C:\Users\24503>
PS C:\Users\24503> New-Item `
>>     -ItemType Directory `
>>     -Path $dst `
>>     -Force |
>>     Out-Null
PS C:\Users\24503>
PS C:\Users\24503> $resolvedSrc =
>>     (Resolve-Path -LiteralPath $src).Path
PS C:\Users\24503>
PS C:\Users\24503> $resolvedDst =
>>     (Resolve-Path -LiteralPath $dst).Path
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Resolved source      :" $resolvedSrc
Resolved source      : C:\Program Files\WindowsApps\OpenAI.Codex_26.911.7940.0_x64__2p2nqsd0c76g0\app\resources\cua_node
PS C:\Users\24503> Write-Host "Resolved destination :" $resolvedDst
Resolved destination : C:\Users\24503\AppData\Local\OpenAI\Codex\runtimes\cua_node\fcf5d08964516500
PS C:\Users\24503>
PS C:\Users\24503> if ($resolvedSrc -eq $resolvedDst) {
>>     throw "Source and destination unexpectedly resolve to the same path."
>> }
PS C:\Users\24503>
PS C:\Users\24503> & "$env:SystemRoot\System32\xcopy.exe" `
>>     "$resolvedSrc\*" `
>>     "$resolvedDst\" `
>>     /E /I /H /Y /G /Q
复制了 4123 个文件
PS C:\Users\24503>
PS C:\Users\24503> $copyExit = $LASTEXITCODE
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`nxcopy exit code:" $copyExit

xcopy exit code: 0
PS C:\Users\24503>
PS C:\Users\24503> if ($copyExit -gt 1) {
>>     throw "xcopy failed with exit code $copyExit."
>> }
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "Runtime copy completed successfully." -ForegroundColor Green
Runtime copy completed successfully.
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== FILE VERIFY ==========" -ForegroundColor Cyan

========== FILE VERIFY ==========
PS C:\Users\24503>
PS C:\Users\24503> foreach ($file in $requiredFiles) {
>>
>>     $sourceFile =
>>         Join-Path `
>>             $src `
>>             $file
>>
>>     $runtimeFile =
>>         Join-Path `
>>             $dst `
>>             $file
>>
>>     if (-not (Test-Path -LiteralPath $runtimeFile)) {
>>         throw "Runtime file missing after copy: $runtimeFile"
>>     }
>>
>>     $sourceHash =
>>         (Get-FileHash `
>>             -LiteralPath $sourceFile `
>>             -Algorithm SHA256).Hash
>>
>>     $runtimeHash =
>>         (Get-FileHash `
>>             -LiteralPath $runtimeFile `
>>             -Algorithm SHA256).Hash
>>
>>     [PSCustomObject]@{
>>         File  = $file
>>         Match = ($sourceHash -eq $runtimeHash)
>>     }
>> }

File          Match
----          -----
manifest.json  True
bin\node.exe   True
bin\node_r...  True


PS C:\Users\24503>

运行结果如上，
启动客户端也成功了

## Response:
2026/9/17 21:53:22

这次可以确认：**修复成功，而且文件层面的验证也是完整有效的。**

关键结果全部通过：

```text
Package version : 26.911.7940.0
Runtime ID      : fcf5d08964516500

复制了 4123 个文件
xcopy exit code: 0

manifest.json      True
bin\node.exe        True
bin\node_repl.exe   True
```

其中三个 `True` 是实际计算 SHA-256 后得到的匹配结果，不存在上次“变量为空导致误判”的问题。随后客户端又能够正常启动，因此这次故障链可以正式闭合：

```text
26.911.7940.0
        ↓
需要 cua_node runtime
fcf5d08964516500
        ↓
自动 staging 连续两次失败
        ↓
node.exe       ✓
node_repl.exe  ✗
manifest.json  ✗
        ↓
renderer = 0
MainWindowHandle = 0
        ↓
有进程，无界面
        ↓
从当前 MSIX 官方 cua_node 源重建 runtime
        ↓
4123 files / xcopy exit 0
        ↓
3 个关键文件 SHA-256 全部一致
        ↓
客户端正常启动
```

**现在不要再修改 `fcf5d08964516500` 正式 runtime，也暂时不要删除两个 `.staging-fcf5d08964516500-*` 目录。** 后者可以作为这次故障的现场证据保留，不影响正常运行。

另外，现在出现了一个比修复本身更值得排查的问题：你之前设置的

```text
CODEX_SPARKLE_ENABLED=false
Store AutoDownload=2
```

目前都仍然有效，但客户端还是从 `26.903.9818.0` 升到了 **`26.911.7940.0`**。这说明我们之前认为的“两层暂停更新”并没有真正实现版本冻结。

我建议下一步单独排查这个问题，重点确定 **26.911.7940.0 究竟是通过 Store、App Installer/MSIX servicing，还是 ChatGPT 自身另一条更新路径安装的**。把这条更新路径锁定后，才能制定真正可靠、且不破坏 Windows 更新体系的暂停方案。

## Prompt:
2026/9/17 21:53:57

请为我排查，为什么会自动更新

## Response:
2026/9/17 21:54:04

我会把这次排查分成两层：先确认**“是谁更新了包”**（ChatGPT 自身 updater / Microsoft Store / AppX/MSIX servicing），再确认**为什么你现有的两个禁用设置没有拦住它**。先不改任何策略，只做只读取证。

目前有一个关键点：`AutoDownload=2` 确实是微软定义的“关闭 Store 应用自动下载和安装更新”策略，而且 Windows 11 Pro 也受支持；但微软同时明确说明，**Store 应用还可以实现自己的 self-update 机制**，这条路径与 Store 全局自动更新并不是一回事。

因此现在不能只凭“两个开关都显示关闭”就判断是哪一层失效。最有价值的是查 **9 月 12 日之后 OpenAI.Codex 的 AppX 部署事件和 ChatGPT updater 日志**，它们通常能直接暴露更新发起者和 MSIX 来源。

可以继续排，而且现在已经有一个比较明确的判断框架：

**`AutoDownload=2` 本身没有写错。** 微软当前文档明确说明，它对应“关闭 Microsoft Store 应用更新的自动下载和安装”，Windows 11 Pro 也支持。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com))  
但微软同时明确说明：**MSIX/Store 应用可以实现自己的 self-update，直接调用 Windows 的包更新 API。** 这条路径不是普通的 Store 后台自动更新。([Microsoft Learn](https://learn.microsoft.com/en-us/windows/iot/iot-enterprise/device-management/app-updates?utm_source=chatgpt.com))

而 ChatGPT/Codex Windows 客户端确实存在这样的 updater。近期公开 issue 已经捕捉到它会通过 `windows-store-updater` / `windows-msix-updater` 主动查询、下载甚至 staging 新 MSIX；有案例的 AppX 事件直接把安装来源指向 ChatGPT 自己缓存的：

```text
...\windows-msix-updater\...\ChatGPT-x64.msix
```

([GitHub](https://github.com/openai/codex/issues/35366?utm_source=chatgpt.com))

所以目前有两个主要候选：

1. **Microsoft Store 后台自己更新了它** -- 那么 `AutoDownload=2` 没有在更新发生时生效，或者更新任务已经在设置策略前排队。
2. **ChatGPT 自己的 Windows updater 更新了它** -- 那么更值得关注的是 `CODEX_SPARKLE_ENABLED=false` 为什么没有真正禁用当前客户端的 updater，或者这个变量只控制其中一层 updater。

现在不要猜。Windows 本机日志一般可以直接把这两种情况区分出来。

## 第一步：取证这次 `26.911.7940.0` 是谁安装的

请打开**普通 PowerShell**。如果下面脚本提示访问 `WindowsApps` 被拒绝，再换成管理员 PowerShell；除此之外先不需要管理员。

整段运行：

```powershell
$ErrorActionPreference = "Continue"

$start = Get-Date "2026-09-12 15:00:00"
$end   = Get-Date "2026-09-18 00:00:00"

$out = Join-Path `
    ([Environment]::GetFolderPath("Desktop")) `
    ("ChatGPT-update-diagnosis-" + (Get-Date -Format "yyyyMMdd-HHmmss") + ".txt")

"============================================================" |
    Out-File $out -Encoding utf8
"CHATGPT UPDATE PROVENANCE DIAGNOSIS" |
    Out-File $out -Append -Encoding utf8
"Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" |
    Out-File $out -Append -Encoding utf8
"============================================================" |
    Out-File $out -Append -Encoding utf8

"`r`n========== 1. CURRENT PACKAGE ==========" |
    Out-File $out -Append -Encoding utf8

$pkg = Get-AppxPackage -Name OpenAI.Codex -ErrorAction SilentlyContinue

$pkg |
    Select-Object Name,
                  Version,
                  Status,
                  SignatureKind,
                  PackageFamilyName,
                  InstallLocation |
    Format-List |
    Out-String -Width 300 |
    Out-File $out -Append -Encoding utf8

"`r`n========== 2. UPDATE CONTROLS ==========" |
    Out-File $out -Append -Encoding utf8

[PSCustomObject]@{
    CODEX_SPARKLE_ENABLED =
        [Environment]::GetEnvironmentVariable(
            "CODEX_SPARKLE_ENABLED",
            "User"
        )

    Store_AutoDownload =
        (
            Get-ItemProperty `
                -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
                -Name AutoDownload `
                -ErrorAction SilentlyContinue
        ).AutoDownload
} |
    Format-List |
    Out-String |
    Out-File $out -Append -Encoding utf8

"`r`n========== 3. PACKAGE DIRECTORY TIMESTAMP ==========" |
    Out-File $out -Append -Encoding utf8

if ($null -ne $pkg) {

    Get-Item `
        -LiteralPath $pkg.InstallLocation `
        -ErrorAction SilentlyContinue |
    Select-Object FullName,
                  CreationTime,
                  LastWriteTime |
    Format-List |
    Out-String -Width 300 |
    Out-File $out -Append -Encoding utf8
}

"`r`n========== 4. APPX DEPLOYMENT EVENTS ==========" |
    Out-File $out -Append -Encoding utf8

$events =
    Get-WinEvent `
        -FilterHashtable @{
            LogName   = "Microsoft-Windows-AppXDeploymentServer/Operational"
            StartTime = $start
            EndTime   = $end
        } `
        -ErrorAction SilentlyContinue |
    Where-Object {
        $_.Message -match `
            "OpenAI\.Codex|ChatGPT-x64\.msix|windows-msix-updater|26\.911\.7940"
    } |
    Sort-Object TimeCreated

$events |
    Select-Object TimeCreated,
                  Id,
                  LevelDisplayName,
                  Message |
    Format-List |
    Out-String -Width 500 |
    Out-File $out -Append -Encoding utf8

"`r`n========== 5. MICROSOFT STORE EVENTS ==========" |
    Out-File $out -Append -Encoding utf8

$storeEvents =
    Get-WinEvent `
        -FilterHashtable @{
            LogName   = "Microsoft-Windows-Store/Operational"
            StartTime = $start
            EndTime   = $end
        } `
        -ErrorAction SilentlyContinue |
    Where-Object {
        $_.Message -match `
            "OpenAI\.Codex|9PLM9XGG6VKS|ChatGPT|26\.911\.7940"
    } |
    Sort-Object TimeCreated

$storeEvents |
    Select-Object TimeCreated,
                  Id,
                  LevelDisplayName,
                  Message |
    Format-List |
    Out-String -Width 500 |
    Out-File $out -Append -Encoding utf8

"`r`n========== 6. CHATGPT DESKTOP LOGS ==========" |
    Out-File $out -Append -Encoding utf8

$logRoots = @(
    "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs",
    "$env:APPDATA\Codex\Logs"
)

foreach ($root in $logRoots) {

    "LOG ROOT: $root" |
        Out-File $out -Append -Encoding utf8

    if (Test-Path -LiteralPath $root) {

        Get-ChildItem `
            -LiteralPath $root `
            -File `
            -Recurse `
            -ErrorAction SilentlyContinue |
        Where-Object {
            $_.LastWriteTime -ge $start -and
            $_.LastWriteTime -le $end
        } |
        Sort-Object LastWriteTime |
        Select-Object FullName,
                      Length,
                      CreationTime,
                      LastWriteTime |
        Format-Table -AutoSize |
        Out-String -Width 400 |
        Out-File $out -Append -Encoding utf8

        "`r`n--- updater-related log lines ---" |
            Out-File $out -Append -Encoding utf8

        Get-ChildItem `
            -LiteralPath $root `
            -File `
            -Recurse `
            -ErrorAction SilentlyContinue |
        Where-Object {
            $_.LastWriteTime -ge $start -and
            $_.LastWriteTime -le $end
        } |
        Select-String `
            -Pattern `
                "windows-store-updater",
                "windows-msix-updater",
                "sparkle",
                "hasUpdate",
                "manifestBuildVersion",
                "26.911.7940",
                "ChatGPT-x64.msix",
                "SilentDownload",
                "install prepared Windows update" `
            -ErrorAction SilentlyContinue |
        ForEach-Object {
            "{0}:{1}: {2}" -f `
                $_.Path,
                $_.LineNumber,
                $_.Line
        } |
        Out-File $out -Append -Encoding utf8
    }
}

"`r`n========== 7. GROUP POLICY RESULT ==========" |
    Out-File $out -Append -Encoding utf8

gpresult /Scope Computer /V |
    Out-String -Width 400 |
    Out-File $out -Append -Encoding utf8

"`r`n========== END ==========" |
    Out-File $out -Append -Encoding utf8

Write-Host ""
Write-Host "Diagnosis file created:" -ForegroundColor Green
Write-Host $out
```

这段**完全是只读诊断**。唯一写入的是桌面上的一个 `.txt` 报告，不会修改 ChatGPT、Store、注册表或组策略。

### 我们最想看到什么

如果 AppX 事件里出现类似：

```text
windows-msix-updater
...\ChatGPT-x64.msix
```

那么几乎可以直接确定：

> **不是 Microsoft Store 的普通后台自动更新，而是 ChatGPT 自己的 MSIX updater 发起了更新。**

已有 Codex Windows 故障报告就是通过 `AppXDeploymentServer` 的事件定位到这种来源的。([GitHub](https://github.com/openai/codex/issues/35366?utm_source=chatgpt.com))

如果 Store 日志里反而出现：

```text
productId = 9PLM9XGG6VKS
ClientAppId = ...
BeginDownload
EndDownload
```

那么就需要进一步看 `ClientAppId`。已有近期案例表明，ChatGPT 自己也可以作为客户端调用 Microsoft Store 更新接口，日志中会直接出现 `clientAppId = ChatGPT`。([GitHub](https://github.com/openai/codex/issues/44601?utm_source=chatgpt.com))

这点非常重要：

**“使用 Microsoft Store API 下载更新” ≠ “Microsoft Store 后台自动更新”。**

即使最终 MSIX 是通过 Store 基础设施下载的，也可能是 **ChatGPT 主动发起**。

如果日志里出现：

```text
[windows-store-updater]
buildVersion=...
manifestBuildVersion=...
```

也能证明客户端运行时主动执行了更新检查。近期多个公开问题都捕捉到了这一模块。([GitHub](https://github.com/openai/codex/issues/24010?utm_source=chatgpt.com))

### 我目前更怀疑哪一种

基于现有证据，我现在**略偏向 ChatGPT 自身 updater 路径，而不是 Store 普通后台自动更新**。

原因是：

- `AutoDownload=2` 当前确实存在，而且微软文档确认它就是关闭 Store 自动更新的正确设备策略。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com))
- ChatGPT Windows 客户端已经被公开证实带有独立的 `windows-store-updater` 和 `windows-msix-updater`，并能自己调用 Store/MSIX 安装链。([GitHub](https://github.com/openai/codex/issues/35366?utm_source=chatgpt.com))
- 微软明确说明 Store 应用可以实现 self-update，因此 Store 全局 auto-update policy 并不能从概念上等同于“禁止应用自己主动更新”。([Microsoft Learn](https://learn.microsoft.com/en-us/windows/iot/iot-enterprise/device-management/app-updates?utm_source=chatgpt.com))

但这里还有一个尚未解释的问题：

```text
CODEX_SPARKLE_ENABLED=false
```

理论上应该抑制 ChatGPT updater。公开故障报告中确实观察到这个变量会改变 updater 初始化行为。([GitHub](https://github.com/openai/codex/issues/41073?utm_source=chatgpt.com))

所以**如果日志最终证明是 ChatGPT 自己更新的**，下一步我们就要继续查：

> 为什么这个用户级环境变量虽然在注册表/PowerShell 中是 `false`，但 ChatGPT 进程实际上仍启动了 updater？

这可能涉及 MSIX 激活时继承的环境、变量生效时机，或者当前版本已经存在不受这个变量控制的另一条 updater 路径。

先运行上面的脚本，把生成的 **`ChatGPT-update-diagnosis-*.txt` 上传给我**。有这份日志后，我们大概率能把“谁更新的、什么时候更新的、通过哪条路径”精确定位出来。

---

**Sources:**

- [组策略如何与打包的应用配合使用 - MSIX | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com)
- [[Windows] MSIX update leaves Codex unregistered · Issue #35366 · openai/codex · GitHub](https://github.com/openai/codex/issues/35366?utm_source=chatgpt.com)

## Prompt:
2026/9/17 21:56:36

Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\24503> $ErrorActionPreference = "Continue"
PS C:\Users\24503>
PS C:\Users\24503> $start = Get-Date "2026-09-12 15:00:00"
PS C:\Users\24503> $end   = Get-Date "2026-09-18 00:00:00"
PS C:\Users\24503>
PS C:\Users\24503> $out = Join-Path `
>>     ([Environment]::GetFolderPath("Desktop")) `
>>     ("ChatGPT-update-diagnosis-" + (Get-Date -Format "yyyyMMdd-HHmmss") + ".txt")
PS C:\Users\24503>
PS C:\Users\24503> "============================================================" |
>>     Out-File $out -Encoding utf8
PS C:\Users\24503> "CHATGPT UPDATE PROVENANCE DIAGNOSIS" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503> "Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503> "============================================================" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 1. CURRENT PACKAGE ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> $pkg = Get-AppxPackage -Name OpenAI.Codex -ErrorAction SilentlyContinue
PS C:\Users\24503>
PS C:\Users\24503> $pkg |
>>     Select-Object Name,
>>                   Version,
>>                   Status,
>>                   SignatureKind,
>>                   PackageFamilyName,
>>                   InstallLocation |
>>     Format-List |
>>     Out-String -Width 300 |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 2. UPDATE CONTROLS ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> [PSCustomObject]@{
>>     CODEX_SPARKLE_ENABLED =
>>         [Environment]::GetEnvironmentVariable(
>>             "CODEX_SPARKLE_ENABLED",
>>             "User"
>>         )
>>
>>     Store_AutoDownload =
>>         (
>>             Get-ItemProperty `
>>                 -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
>>                 -Name AutoDownload `
>>                 -ErrorAction SilentlyContinue
>>         ).AutoDownload
>> } |
>>     Format-List |
>>     Out-String |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 3. PACKAGE DIRECTORY TIMESTAMP ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> if ($null -ne $pkg) {
>>
>>     Get-Item `
>>         -LiteralPath $pkg.InstallLocation `
>>         -ErrorAction SilentlyContinue |
>>     Select-Object FullName,
>>                   CreationTime,
>>                   LastWriteTime |
>>     Format-List |
>>     Out-String -Width 300 |
>>     Out-File $out -Append -Encoding utf8
>> }
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 4. APPX DEPLOYMENT EVENTS ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> $events =
>>     Get-WinEvent `
>>         -FilterHashtable @{
>>             LogName   = "Microsoft-Windows-AppXDeploymentServer/Operational"
>>             StartTime = $start
>>             EndTime   = $end
>>         } `
>>         -ErrorAction SilentlyContinue |
>>     Where-Object {
>>         $_.Message -match `
>>             "OpenAI\.Codex|ChatGPT-x64\.msix|windows-msix-updater|26\.911\.7940"
>>     } |
>>     Sort-Object TimeCreated
PS C:\Users\24503>
PS C:\Users\24503> $events |
>>     Select-Object TimeCreated,
>>                   Id,
>>                   LevelDisplayName,
>>                   Message |
>>     Format-List |
>>     Out-String -Width 500 |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 5. MICROSOFT STORE EVENTS ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> $storeEvents =
>>     Get-WinEvent `
>>         -FilterHashtable @{
>>             LogName   = "Microsoft-Windows-Store/Operational"
>>             StartTime = $start
>>             EndTime   = $end
>>         } `
>>         -ErrorAction SilentlyContinue |
>>     Where-Object {
>>         $_.Message -match `
>>             "OpenAI\.Codex|9PLM9XGG6VKS|ChatGPT|26\.911\.7940"
>>     } |
>>     Sort-Object TimeCreated
PS C:\Users\24503>
PS C:\Users\24503> $storeEvents |
>>     Select-Object TimeCreated,
>>                   Id,
>>                   LevelDisplayName,
>>                   Message |
>>     Format-List |
>>     Out-String -Width 500 |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 6. CHATGPT DESKTOP LOGS ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> $logRoots = @(
>>     "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs",
>>     "$env:APPDATA\Codex\Logs"
>> )
PS C:\Users\24503>
PS C:\Users\24503> foreach ($root in $logRoots) {
>>
>>     "LOG ROOT: $root" |
>>         Out-File $out -Append -Encoding utf8
>>
>>     if (Test-Path -LiteralPath $root) {
>>
>>         Get-ChildItem `
>>             -LiteralPath $root `
>>             -File `
>>             -Recurse `
>>             -ErrorAction SilentlyContinue |
>>         Where-Object {
>>             $_.LastWriteTime -ge $start -and
>>             $_.LastWriteTime -le $end
>>         } |
>>         Sort-Object LastWriteTime |
>>         Select-Object FullName,
>>                       Length,
>>                       CreationTime,
>>                       LastWriteTime |
>>         Format-Table -AutoSize |
>>         Out-String -Width 400 |
>>         Out-File $out -Append -Encoding utf8
>>
>>         "`r`n--- updater-related log lines ---" |
>>             Out-File $out -Append -Encoding utf8
>>
>>         Get-ChildItem `
>>             -LiteralPath $root `
>>             -File `
>>             -Recurse `
>>             -ErrorAction SilentlyContinue |
>>         Where-Object {
>>             $_.LastWriteTime -ge $start -and
>>             $_.LastWriteTime -le $end
>>         } |
>>         Select-String `
>>             -Pattern `
>>                 "windows-store-updater",
>>                 "windows-msix-updater",
>>                 "sparkle",
>>                 "hasUpdate",
>>                 "manifestBuildVersion",
>>                 "26.911.7940",
>>                 "ChatGPT-x64.msix",
>>                 "SilentDownload",
>>                 "install prepared Windows update" `
>>             -ErrorAction SilentlyContinue |
>>         ForEach-Object {
>>             "{0}:{1}: {2}" -f `
>>                 $_.Path,
>>                 $_.LineNumber,
>>                 $_.Line
>>         } |
>>         Out-File $out -Append -Encoding utf8
>>     }
>> }
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== 7. GROUP POLICY RESULT ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> gpresult /Scope Computer /V |
>>     Out-String -Width 400 |
>>     Out-File $out -Append -Encoding utf8
错误: 拒绝访问。
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> "`r`n========== END ==========" |
>>     Out-File $out -Append -Encoding utf8
PS C:\Users\24503>
PS C:\Users\24503> Write-Host ""

PS C:\Users\24503> Write-Host "Diagnosis file created:" -ForegroundColor Green
Diagnosis file created:
PS C:\Users\24503> Write-Host $out
D:\桌面\ChatGPT-update-diagnosis-20260917-215535.txt
PS C:\Users\24503>
PS C:\Users\24503>

运行结果如上

## Response:
2026/9/17 21:56:44

这一步执行基本成功。唯一异常是：

```text
gpresult /Scope Computer /V
错误: 拒绝访问。
```

这是因为当前 PowerShell 不是管理员权限，**不影响前面最关键的 AppX 部署事件、Microsoft Store 事件和 ChatGPT updater 日志采集**。

现在真正用于判断“是谁自动更新了 ChatGPT”的证据已经写入：

```text
D:\桌面\ChatGPT-update-diagnosis-20260917-215535.txt
```

请直接把这个 **`ChatGPT-update-diagnosis-20260917-215535.txt` 文件上传给我**。我会重点交叉核对三组证据：

1. `APPX DEPLOYMENT EVENTS`
   - 查 `26.911.7940.0` 的实际部署时间；
   - 查 MSIX 的来源路径；
   - 如果出现 `windows-msix-updater`、`ChatGPT-x64.msix` 等，就能直接追到 ChatGPT 自身 updater。

2. `MICROSOFT STORE EVENTS`
   - 判断是不是 Store 后台自动更新；
   - 还是 ChatGPT 作为客户端主动调用 Store 更新接口；
   - 如果有 `ClientAppId` 等字段，可以进一步确定更新发起者。

3. `CHATGPT DESKTOP LOGS`
   - 查 `windows-store-updater`、`windows-msix-updater`、`hasUpdate`、`manifestBuildVersion` 等；
   - 确认 `CODEX_SPARKLE_ENABLED=false` 的情况下 updater 是否仍然被初始化。

暂时**不用重新以管理员身份执行整套脚本**。`gpresult` 只是辅助证据；先分析已经生成的 txt，很可能已经足以定位原因。

如果之后确实需要组策略的详细结果，我们再单独用管理员 PowerShell 补这一项即可。

## Prompt:
2026/9/17 21:58:39

文件如上

[Attachment: ChatGPT-update-diagnosis-20260917-215535.txt]

## Response:
2026/9/17 21:58:43

日志已经把“谁发起更新”指向得很清楚了：这次不是 ChatGPT 自己的 updater，而是 **Windows Update Orchestrator / MoUsoCoreWorker** 通过 Store/WU 通道拉起了 `OpenAI.Codex` 更新。接下来我核对微软当前策略语义，解释为什么 `AutoDownload=2` 仍没有挡住这条路径。

结论已经比较明确：**这次把 ChatGPT 从旧版本更新到 `26.911.7940.0` 的，不是 ChatGPT 自身 updater，而是 Windows 的 Update Orchestrator（`MoUsoCoreWorker`）经由 Microsoft Store / Windows Update 的安装通道触发的。**

你设置的第一层：

```text
CODEX_SPARKLE_ENABLED=false
```

实际上是有效的。9 月 12 日设置之后，ChatGPT 日志已经从 `enableUpdater=true` 变为 `enableUpdater=false`；9 月 13、14、15 日的后续启动也持续显示 `enableUpdater=false`。换言之，**ChatGPT 自己的 in-app updater 确实已经被关掉了**。`ChatGPT-update-diagnosis-20260917-215535.txt`

但系统层面仍然存在另一条完全独立的更新链。

你的 9 月 17 日日志显示，20:28 左右 Windows 开始扫描 `OpenAI.Codex`，明确标记：

```text
FulfillmentPluginId = WU
ClientAppId = Update;MoUpdateOrchestratorDeviceScan-...
```

随后又转入：

```text
ClientAppId =
Update;MoUpdateOrchestratorUserScan-
MoUsoCoreWorker-
SearchForAllUpdatesWithUpdateOptionsAsync
```

这不是 `ChatGPT.exe`，也不是 OpenAI 自己的 updater，而是 **Windows Update Orchestrator / MoUsoCoreWorker**。`ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt`

之后证据链非常完整。20:34:06，Windows Store Install Service 开始处理 `OpenAI.Codex_26.911.7940.0`：

```text
IsUpdate       = true
IsInteractive  = false
IsMandatory    = false
UserAttemptNumber = 0
ClientAppId    = Update;MoUpdateOrchestratorUserScan-...
```

并开始下载。`ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt`

它最终实际下载了：

```text
634,231,285 bytes
```

而且日志仍明确标记更新客户端为：

```text
MoUpdateOrchestratorUserScan-MoUsoCoreWorker
```

随后进入 `BeginInstall`，20:38:55 `EndInstall` 返回 `HResult=0`，即成功。`ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt`

与此同时 AppXDeploymentServer 在 20:34:58 开始 Stage `26.911.7940.0`，20:38:49 Stage 成功，随后明确记录：

```text
OpenAI.Codex_26.908.4834.0
is updating to
OpenAI.Codex_26.911.7940.0
```

所以新版本不是“客户端自己悄悄下载覆盖”的，而是进入了 Windows 标准 Store/MSIX 包部署体系。`ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt`

### 为什么 `AutoDownload=2` 还没挡住它？

这里需要把结论分成“已经确定”和“还差一步才能确定”。

你当前确实存在：

```text
CODEX_SPARKLE_ENABLED : false
Store_AutoDownload    : 2
```

`ChatGPT-update-diagnosis-20260917-215535.txt`

而微软现在的正式文档也确认：

```text
HKLM\SOFTWARE\Policies\Microsoft\WindowsStore
AutoDownload = 2
```

对应的就是 **Turn off Automatic Download and Install of updates**，用于关闭 Microsoft Store 应用的自动下载/安装更新；这一策略适用于 Windows Pro、Enterprise、Education 等版本。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com))

因此，我们之前设置 `AutoDownload=2` **在语义上没有设置错**。

问题在于，这次日志显示更新请求的客户端不是常规的 Microsoft Store UI，而是：

```text
MoUpdateOrchestratorDeviceScan
MoUpdateOrchestratorUserScan
MoUsoCoreWorker
FulfillmentPluginId = WU
```

即 Windows Update Orchestrator 发起扫描，再交给 Store Install Service 下载和部署。

所以现在最合理的解释是：

```text
ChatGPT 自带 updater
    ↓
CODEX_SPARKLE_ENABLED=false
    ↓
确实被关闭

Microsoft Store 常规自动更新
    ↓
AutoDownload=2
    ↓
理论上被关闭

但是

Windows Update / Update Orchestrator
    ↓
MoUsoCoreWorker
    ↓
扫描 Store/WU catalog
    ↓
发现 OpenAI.Codex 更新
    ↓
Store Install Service
    ↓
下载 + Stage + Register
    ↓
26.911.7940.0
```

微软也明确区分了应用本身通过 Store API 获取更新、Store 的自动更新行为以及 Windows 的包部署基础设施；Store 应用更新可以由系统的 Store 服务进行下载和安装。([Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/apps/package-and-deploy/package-updates-from-store?utm_source=chatgpt.com))

不过，**我暂时不会把它直接定性成“Windows 的 bug 绕过了 AutoDownload=2”**。目前还存在一个需要排除的可能性：你的注册表虽然现在有 `AutoDownload=2`，但我们上次执行 `gpresult` 时因为不是管理员而失败了，因此还没有证明该策略在 9 月 17 日更新发生时确实作为“有效计算机策略”被 Windows Policy Manager 接受，也没有排除 MDM/CSP 或其他策略覆盖它。

另外有一个细节值得注意：9 月 12 日 `26.908.4834.0` 在 **16:08** 就已经进入 Stage，而你是在随后才开始处理暂停更新问题，所以那一次不能拿来证明 `AutoDownload=2` 失效。真正值得分析的是 9 月 15 日和 9 月 17 日后续的更新，它们已经明确走的是 `MoUsoCoreWorker` 路径。`ChatGPT-update-diagnosis-20260917-215535.txt`

### 下一步：确认 `AutoDownload=2` 到底有没有成为“有效策略”

这一步只需要做一次，而且是只读检查。请打开 **PowerShell（管理员）**，不要用普通 PowerShell，执行：

```powershell
Write-Host "`n========== WINDOWS VERSION ==========" -ForegroundColor Cyan

Get-ComputerInfo |
    Select-Object WindowsProductName,
                  WindowsVersion,
                  OsBuildNumber,
                  OsArchitecture |
    Format-List

Write-Host "`n========== STORE GPO REGISTRY ==========" -ForegroundColor Cyan

Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
    -ErrorAction SilentlyContinue |
    Select-Object AutoDownload,
                  RemoveWindowsStore |
    Format-List

Write-Host "`n========== POLICY MANAGER ==========" -ForegroundColor Cyan

$policyPaths = @(
    "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\ApplicationManagement",
    "HKLM:\SOFTWARE\Microsoft\PolicyManager\default\ApplicationManagement\AllowAppStoreAutoUpdate"
)

foreach ($path in $policyPaths) {

    Write-Host "`n$path"

    Get-ItemProperty `
        -Path $path `
        -ErrorAction SilentlyContinue |
        Format-List
}

Write-Host "`n========== GPRESULT STORE POLICY ==========" -ForegroundColor Cyan

$gp = gpresult /Scope Computer /V

$gp |
    Select-String `
        -Pattern "Store",
                 "Automatic Download",
                 "AutoDownload",
                 "Microsoft Store" `
        -Context 3,5

Write-Host "`n========== WINDOWS UPDATE POLICY ==========" -ForegroundColor Cyan

Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -ErrorAction SilentlyContinue |
    Format-List

Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -ErrorAction SilentlyContinue |
    Format-List
```

**这一步不会修改任何设置。**

拿到这份结果后，我们就能把最后两种情况区分开：

1. **如果 GPResult / PolicyManager 明确确认“关闭 Store 自动更新”已经生效**，但 `MoUsoCoreWorker` 仍在 9 月 15、17 日下载 Codex，那么基本可以判断：你当前 Windows 上存在一条 **Windows Update Orchestrator 不受该 Store auto-update 策略约束的更新路径**，或者存在 Windows 当前版本的策略执行异常。届时要换更强的冻结办法。

2. **如果注册表有 `2`，但有效策略里实际上没有生效，或者 PolicyManager 显示 Store auto-update 仍被允许**，那原因就更简单：之前只是写入了 registry value，但 Windows 的实际策略状态没有按预期落地或被其他策略覆盖。我们就修正这一层即可。

所以目前“为什么会更新”已经定位了大约 90%：**OpenAI updater 已经关掉；真正更新你的是 Windows Update Orchestrator → Microsoft Store Install Service 这条系统级链路。** 剩下只需要确定为什么 `AutoDownload=2` 没有约束住它。

---

**Sources:**

- [组策略如何与打包的应用配合使用 - MSIX | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/msix/group-policy-msix?utm_source=chatgpt.com)

## Prompt:
2026/9/17 22:04:05

Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\24503> Write-Host "`n========== WINDOWS VERSION ==========" -ForegroundColor Cyan

========== WINDOWS VERSION ==========
PS C:\Users\24503>
PS C:\Users\24503> Get-ComputerInfo |
>>     Select-Object WindowsProductName,
>>                   WindowsVersion,
>>                   OsBuildNumber,
>>                   OsArchitecture |
>>     Format-List


WindowsProductName : Windows 10 Home China
WindowsVersion     : 2009
OsBuildNumber      : 26200
OsArchitecture     : 64 位



PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== STORE GPO REGISTRY ==========" -ForegroundColor Cyan

========== STORE GPO REGISTRY ==========
PS C:\Users\24503>
PS C:\Users\24503> Get-ItemProperty `
>>     -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" `
>>     -ErrorAction SilentlyContinue |
>>     Select-Object AutoDownload,
>>                   RemoveWindowsStore |
>>     Format-List


AutoDownload       : 2
RemoveWindowsStore :



PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== POLICY MANAGER ==========" -ForegroundColor Cyan

========== POLICY MANAGER ==========
PS C:\Users\24503>
PS C:\Users\24503> $policyPaths = @(
>>     "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\ApplicationManagement",
>>     "HKLM:\SOFTWARE\Microsoft\PolicyManager\default\ApplicationManagement\AllowAppStoreAutoUpdate"
>> )
PS C:\Users\24503>
PS C:\Users\24503> foreach ($path in $policyPaths) {
>>
>>     Write-Host "`n$path"
>>
>>     Get-ItemProperty `
>>         -Path $path `
>>         -ErrorAction SilentlyContinue |
>>         Format-List
>> }

HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\ApplicationManagement

HKLM:\SOFTWARE\Microsoft\PolicyManager\default\ApplicationManagement\AllowAppStoreAutoUpdate


Behavior               : 73760
GPBlockingRegKeyPath   : Software\Policies\Microsoft\WindowsStore
GPBlockingRegValueName : AutoDownload
highrange              : 2
lowrange               : 0
mergealgorithm         : 1
policytype             : 4
value                  : 2
PSPath                 : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PolicyManager\defaul
                         t\ApplicationManagement\AllowAppStoreAutoUpdate
PSParentPath           : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PolicyManager\defaul
                         t\ApplicationManagement
PSChildName            : AllowAppStoreAutoUpdate
PSDrive                : HKLM
PSProvider             : Microsoft.PowerShell.Core\Registry



PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== GPRESULT STORE POLICY ==========" -ForegroundColor Cyan

========== GPRESULT STORE POLICY ==========
PS C:\Users\24503>
PS C:\Users\24503> $gp = gpresult /Scope Computer /V
PS C:\Users\24503>
PS C:\Users\24503> $gp |
>>     Select-String `
>>         -Pattern "Store",
>>                  "Automatic Download",
>>                  "AutoDownload",
>>                  "Microsoft Store" `
>>         -Context 3,5
PS C:\Users\24503>
PS C:\Users\24503>
PS C:\Users\24503> Write-Host "`n========== WINDOWS UPDATE POLICY ==========" -ForegroundColor Cyan

========== WINDOWS UPDATE POLICY ==========
PS C:\Users\24503>
PS C:\Users\24503> Get-ItemProperty `
>>     -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
>>     -ErrorAction SilentlyContinue |
>>     Format-List
PS C:\Users\24503>
PS C:\Users\24503> Get-ItemProperty `
>>     -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
>>     -ErrorAction SilentlyContinue |
>>     Format-List
PS C:\Users\24503>

运行结果如上

## Response:
2026/9/17 22:04:18

这份结果已经把关键问题缩小到一个很具体的点：**注册表里虽然有 `AutoDownload=2`，但系统并没有把它显示为“当前生效的设备策略”**。我先核对微软当前文档对 Home 版、Policy CSP 和 Store 自动更新策略的适用范围，再判断这是“策略根本未生效”还是“Windows Update 通道绕过 Store 策略”。

这次结果已经把原因基本确定了。

**自动更新的根因不是 ChatGPT 自带 updater 失效，而是你当前 Windows SKU 不支持我们之前设置的 Store 自动更新策略；因此 `AutoDownload=2` 虽然写进了注册表，却没有成为有效策略。随后 Windows Update Orchestrator（`MoUsoCoreWorker`）正常扫描并安装了新版 ChatGPT。**

### 1. `CODEX_SPARKLE_ENABLED=false` 确实生效了

这一点已经由 ChatGPT 自身日志证明。9 月 12 日设置以后，后续启动均出现：

```text
enableSparkle=false
enableUpdater=false
```

所以 OpenAI 客户端自己的更新器已经关闭。`ChatGPT-update-diagnosis-20260917-215535.txt`

因此，不需要再调整：

```text
CODEX_SPARKLE_ENABLED=false
```

它可以继续保留。

### 2. 真正的问题在 `AutoDownload=2`

你的注册表现在确实是：

```text
HKLM\SOFTWARE\Policies\Microsoft\WindowsStore

AutoDownload = 2
```

但这次检查出现了两个非常关键的现象。

首先：

```text
HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\ApplicationManagement
```

**没有任何实际的 Store auto-update policy 输出。**

其次：

```text
gpresult /Scope Computer /V
```

里面也**没有 Store / AutoDownload 对应的有效计算机策略**。

反而你看到的：

```text
HKLM:\SOFTWARE\Microsoft\PolicyManager\default\
ApplicationManagement\AllowAppStoreAutoUpdate

value : 2
```

这里的 `default` 很容易误读。

它不是说“当前策略值等于 2，所以已经关闭更新”。

微软对 `AllowAppStoreAutoUpdate` 的定义是：

- `0` = 不允许自动更新
- `1` = 允许
- `2` = **Not configured，未配置**

而且这个 CSP 的 `Default Value` 本来就是 `2`。([Microsoft Learn](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-applicationmanagement?utm_source=chatgpt.com))

所以你刚才看到的：

```text
default\...\AllowAppStoreAutoUpdate
value = 2
```

实际上只是 Windows 内置的**策略定义默认值**，不是当前设备已经应用了这个策略。

### 3. 为什么策略没生效：你的系统是 Home SKU

你的结果显示：

```text
WindowsProductName : Windows 10 Home China
OsBuildNumber      : 26200
```

这里还有一个 Windows 自身的兼容性显示问题。

Build `26200` 实际属于 **Windows 11 version 25H2**；微软当前官方发布记录明确把 26200 系列列为 Windows 11 25H2。([Microsoft Learn](https://learn.microsoft.com/en-us/windows-insider/release-notes/release-preview-24h2-25h2/build-26100-8942-26200-8942?utm_source=chatgpt.com))

所以这里更准确地理解应该是：

```text
Windows 11 25H2
Home / China-specific Home SKU
Build 26200
```

`Get-ComputerInfo` 返回的 `"Windows 10 Home China"` 产品名称属于这类系统上可能出现的旧兼容性标签，不应根据它判断内核版本。

而真正关键的是 **Home**。

微软当前对：

```text
ApplicationManagement/AllowAppStoreAutoUpdate
```

列出的支持版本只有：

- Pro
- Enterprise
- Education
- IoT Enterprise / LTSC

**没有 Home。** ([Microsoft Learn](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-applicationmanagement?utm_source=chatgpt.com))

对应的 ADMX Store 自动更新策略同样列的是 Pro / Enterprise / Education / IoT Enterprise，不包含 Home。([Microsoft Learn](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-windowsstore?utm_source=chatgpt.com))

所以此前直接执行：

```powershell
New-ItemProperty `
    HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore `
    -Name AutoDownload `
    -Value 2
```

虽然成功地创建了注册表值，但在你的 Home SKU 上，**不能据此认为 Windows 必须执行该策略**。

这也恰好解释了你现在看到的现象：

```text
Registry:
AutoDownload = 2
        ✓

PolicyManager current:
没有 AllowAppStoreAutoUpdate
        ✗

gpresult:
没有 Store 自动更新策略
        ✗
```

也就是说，之前的方案存在一个重要遗漏：**没有检查 Windows Edition 的策略支持范围。**

### 4. 然后 Windows 是如何更新 ChatGPT 的

这部分日志已经非常完整。

2026 年 9 月 17 日 20:28 左右，系统发起：

```text
MoUpdateOrchestratorDeviceScan
MoUsoCoreWorker
SearchForAllUpdatesWithUpdateOptionsAsync
```

并发现：

```text
ProductId = 9PLM9XGG6VKS
PFN = OpenAI.Codex_2p2nqsd0c76g0
```

然后以：

```text
FulfillmentPluginId = WU
ClientAppId =
Update;MoUpdateOrchestratorUserScan-
MoUsoCoreWorker-
SearchForAllUpdatesWithUpdateOptionsAsync
```

进入 Store Install Service。`ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt`

随后开始获取并下载：

```text
OpenAI.Codex_26.911.7940.0
```

下载总量：

```text
634231285 bytes
```

约 605 MB。`ChatGPT-update-diagnosis-20260917-215535.txt` `ChatGPT-update-diagnosis-20260917-215535.txt`

最后 Windows AppX Deployment 明确记录：

```text
OpenAI.Codex_26.908.4834.0
is updating to
OpenAI.Codex_26.911.7940.0
```

并成功完成 Register。`ChatGPT-update-diagnosis-20260917-215535.txt`

所以这次真实路径是：

```text
ChatGPT 自带 updater
CODEX_SPARKLE_ENABLED=false
        │
        └── 已关闭，并非问题

Windows Home
        │
        ├─ HKLM\...\WindowsStore\AutoDownload=2
        │      仅有注册表值
        │
        ├─ PolicyManager current
        │      没有对应有效策略
        │
        └─ Home SKU 本身不在该策略支持范围
                ↓
       Windows Update Orchestrator
                ↓
            MoUsoCoreWorker
                ↓
       SearchForAllUpdates...
                ↓
       Microsoft Store Install Service
                ↓
     OpenAI.Codex 26.911.7940.0
                ↓
             自动安装
```

这就是为什么“两层保护看上去都存在”，实际上仍然更新了。

### 5. 现在不要继续强化 `AutoDownload=2`

在你的机器上继续围绕：

```text
AutoDownload=2
```

增加更多相似的 Store GPO 注册表键意义不大。

也不建议通过以下办法强行解决：

- 禁用 Windows Update；
- 禁用 `UsoSvc`；
- 禁用 `InstallService`；
- 禁用 `AppXSvc`；
- 禁用 BITS；
- 修改 `WindowsApps` ACL。

这些手段影响范围太大，很容易破坏系统更新、Store 应用修复、AppX 注册等机制。

### 下一步

如果你的真正目标仍然是：

> **只暂停 ChatGPT/Codex 更新几周，但 Windows 和其他 Store 应用继续正常更新**

那么现在需要换思路。

对你这台 **Windows Home** 机器，我们应该寻找的是**只针对 `OpenAI.Codex_2p2nqsd0c76g0` / ProductId `9PLM9XGG6VKS` 的定向阻断方法**，而不是 Store 全局 AutoDownload GPO。

我建议下一步就沿这个方向排查：看看 Windows Store Install Service / Windows Update 是否存在**按 PackageFamilyName、ProductId 或 app package 定向禁止升级**的受支持机制；如果没有，再比较几种可逆 workaround 的风险，选择副作用最小的一种。

---

**Sources:**

- [ApplicationManagement Policy CSP | Microsoft Learn](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-applicationmanagement?utm_source=chatgpt.com)

## Prompt:
2026/9/20 16:30:16

请将上述经历写成一份 issue 
更新在 [$$Windows$$ 当 cua_node staging 无法复制 node_repl.exe 时，26.901.1978.0 版本无法启动 UI · 问题 #42501 · openai/codex --- $$Windows$$ 26.901.1978.0 fails to launch UI when cua_node staging cannot copy node_repl.exe · Issue #42501 · openai/codex](https://github.com/openai/codex/issues/42501#issuecomment-5724437896)  上，

注意，之前我已报告多次多个版本的这个问题，
这回要注意详细新增，设置两个参数阻断更新之后，仍会出发更新，之后造成打不开，这个问题

## Response:
2026/9/20 16:30:26

我会按“追加到同一 issue 的更新评论”来写，不重复前几次已经报告过的完整故障背景，重点突出这次新增证据：**已经同时关闭 ChatGPT 自身 updater 与 Store 自动更新后，系统仍经 `MoUsoCoreWorker` / Windows Update Orchestrator 推送 `OpenAI.Codex` 新版，随后再次触发 `cua_node` staging 不完整并导致 UI 无法启动。**

我看了 #42501 当前正文。原 issue 已经明确指出：**每次更新如果引入新的 `cua_node` runtime hash，同一 staging/relocation 故障都可能再次发生**。([GitHub](https://github.com/openai/codex/issues/42501))

这次最值得新增的并不是再重复“`node_repl.exe` 缺失”，而是补上一条更完整的因果链：

**尝试禁用更新 → ChatGPT 自身 updater 确实已被禁用 → Windows Update Orchestrator 仍通过 Store/WU 通道升级 Codex → 新 runtime 再次 staging 失败 → UI 再次完全打不开。**

同时需要把 `AutoDownload=2` 的情况写严谨：它虽然存在于注册表，但你的 Home SKU 上没有显示为 effective policy，因此不能声称 OpenAI/Windows “绕过了一个已生效的策略”。下面这版已经做了这个区分。

**chat_message: **

### Update: the issue recurred again on `26.911.7940.0`, after I had already tried to disable automatic updates

I have now reproduced this same `cua_node` staging failure across multiple Codex/ChatGPT Windows versions. This time there is an additional finding that may be relevant: **I had already tried to stop automatic updates after the previous recurrence, but Windows still updated the package, and the next update triggered the same no-UI failure again.**

#### Update controls already configured

After the previous failure, I configured both of the following:

```text
CODEX_SPARKLE_ENABLED=false
```

and:

```text
HKLM\SOFTWARE\Policies\Microsoft\WindowsStore
AutoDownload = 2
```

The first setting was definitely effective for the Codex in-app updater. Subsequent Codex logs consistently showed:

```text
enableSparkle=false
enableUpdater=false
```

So the later update was **not initiated by the Codex in-app updater**.

For the Store setting, `AutoDownload=2` was still present when I checked it. However, this machine is using a Windows Home China SKU (build `26200`), and the setting did not appear as an effective policy under `PolicyManager\current` or in `gpresult`. Therefore, I cannot confirm that this Store policy was actually enforced on this Windows edition.

#### Windows still updated Codex through the Windows Update / Store path

The Windows Store / Install Service event logs show that the later updates were initiated by Windows Update Orchestrator / `MoUsoCoreWorker`, for example:

```text
FulfillmentPluginId = WU

ClientAppId =
Update;MoUpdateOrchestratorUserScan-
MoUsoCoreWorker-
SearchForAllUpdatesWithUpdateOptionsAsync
```

On September 17, Windows downloaded and installed:

```text
OpenAI.Codex_26.911.7940.0_x64__2p2nqsd0c76g0
```

The download size recorded by the Store Install Service was:

```text
634231285 bytes
```

and AppXDeploymentServer subsequently recorded:

```text
OpenAI.Codex_26.908.4834.0
is updating to
OpenAI.Codex_26.911.7940.0
```

The package update itself completed successfully.

#### Immediately afterward, the same `cua_node` failure occurred again

After `26.911.7940.0` was installed, launching ChatGPT produced background processes but no UI:

```text
Renderer count: 0
MainWindowHandle: 0
```

The new expected runtime was:

```text
fcf5d08964516500
```

Two consecutive staging attempts were created:

```text
.staging-fcf5d08964516500-cP9p1r
.staging-fcf5d08964516500-vH7e0k
```

Both failed in exactly the same state:

```text
node.exe       = present
node_repl.exe  = missing
manifest.json  = missing
```

At the same time, the files bundled inside the installed MSIX were complete:

```text
app\resources\cua_node\manifest.json       = present
app\resources\cua_node\bin\node.exe        = present
app\resources\cua_node\bin\node_repl.exe   = present
```

So this was again a failure while materializing/relocating the bundled `cua_node` runtime, not a missing file in the package itself.

#### The same manual workaround fixed it again

With all `ChatGPT.exe` processes stopped, I manually copied the packaged `cua_node` directory into:

```text
%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\fcf5d08964516500
```

using `xcopy /G`.

Result:

```text
4123 files copied
xcopy exit code: 0
```

I then verified SHA-256 equality between the package source and the rebuilt runtime for:

```text
manifest.json      -> Match = True
bin\node.exe       -> Match = True
bin\node_repl.exe  -> Match = True
```

After that, ChatGPT launched normally and the UI immediately returned.

#### Current failure chain

The sequence on this machine is now effectively:

```text
Codex in-app updater disabled
        ↓
Windows Update / MoUsoCoreWorker still discovers a newer Store package
        ↓
OpenAI.Codex is updated
        ↓
new cua_node runtime hash is required
        ↓
automatic runtime staging is incomplete
        ↓
node.exe copied
node_repl.exe / manifest.json missing
        ↓
renderer never starts
MainWindowHandle remains 0
        ↓
ChatGPT has background processes but no UI
        ↓
manual xcopy /G of the bundled runtime
        ↓
UI works again
```

This is particularly disruptive because the workaround only repairs the **current runtime hash**. A later package update introduces another runtime hash and the same failure can occur again.

It would be very helpful if the Windows runtime materialization path could be fixed so that protected/MSIX-packaged files can be copied reliably without requiring manual `xcopy /G`.

It would also be useful for the app to detect an incomplete `.staging-*` runtime and report the actual relocation/copy failure instead of remaining indefinitely as responsive background processes with no window.

Finally, since disabling the Codex in-app updater does not prevent Windows Update / Store servicing from replacing the MSIX package, a supported way to temporarily defer desktop-app updates would also be valuable for users affected by a version-dependent startup regression like this.

这一版有意没有写成“`AutoDownload=2` 已经生效但被 Windows 绕过”，因为目前证据并不支持这么强的结论；真正有直接日志证据的是：**Codex 自带 updater 已关闭，但后续更新由 `MoUsoCoreWorker` / Windows Update Orchestrator 发起，更新完成后同一 `cua_node` 故障再次发生。**

这会比单纯再次报告 `26.911.7940.0` 打不开更有信息量，也更便于维护者区分“更新来源问题”和“runtime staging 根因问题”。

---

**Sources:**

- [[Windows] 26.901.1978.0 fails to launch UI when cua_node staging cannot copy node_repl.exe · Issue #42501 · openai/codex · GitHub](https://github.com/openai/codex/issues/42501)



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)