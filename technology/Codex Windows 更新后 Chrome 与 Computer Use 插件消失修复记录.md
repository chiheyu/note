# Codex Windows 更新后 Chrome 与 Computer Use 插件消失修复记录

记录时间：2026-06-05

## 适用场景

Windows 上 Codex Desktop 更新后，原本可用的 `Chrome`、`Computer Use`、`Browser` 等 `openai-bundled` 插件突然消失或不可用。尤其是在 Windows 设置中修改过：

`设置 -> 系统 -> 存储 -> 保存新内容的位置`

并且曾把新应用保存位置在 C 盘和 D 盘之间切换过。

本次本机现象：

- Codex Store 包安装在 `C:\Program Files\WindowsApps\OpenAI.Codex_26.602.3474.0_x64__2p2nqsd0c76g0`
- Codex App release 是 `26.602.30954`
- `config.toml` 里仍残留旧版 `computer-use\26.601.21317`
- `openai-bundled` marketplace 在 CLI 中解析为空
- `chrome-native-hosts-v2.json` 仍登记旧版 `26.601.21317`
- 日志出现 `missing-helper-path`、`EBUSY`、`os error 5`
- 包内实际仍包含 `browser`、`chrome`、`computer-use`、`latex`、`sites`

## 根因判断

这不是单纯的“插件文件缺失”，也不是功能开关未开放。

关键证据是日志里有：

```text
browser_use_availability_resolved available=true
computer-use notify config ensure finished platform=win32 reason=missing-helper-path status=skipped
computer-use native pipe startup failed errorMessage="Windows Computer Use helper paths are unavailable"
bundled_plugins_marketplace_resolve_failed errorCode=EBUSY
```

含义：

- `available=true` 说明 Codex 已经允许 Windows Browser/Computer Use 能力。
- `missing-helper-path` 说明本地配置仍指向旧 helper，或新版 helper 未安装到缓存。
- `EBUSY` / `os error 5` 说明旧的 `extension-host.exe` 正在锁住 Chrome 插件缓存，Codex 无法重建 bundled marketplace。
- WindowsApps 是受保护目录。更新、换盘安装、缓存半更新混在一起时，Codex 可能无法直接从 WindowsApps 内的 bundled 源稳定安装插件。

本次采用的修复策略：

1. 不直接修改 WindowsApps。
2. 把当前 Codex 包内的 `openai-bundled` 插件源复制到用户目录。
3. 让 Codex CLI 从用户目录注册和安装插件。
4. 修复 Chrome native messaging 的 HKCU 注册表项。
5. 验证 Computer Use native pipe 是否恢复。

参考文章：

- https://www.autoxb.com/article/112216
- https://developers.openai.com/codex/changelog

## 先做只读排查

检查 Codex Store 包位置和版本：

```powershell
Get-AppxPackage -Name "OpenAI.Codex" -ErrorAction SilentlyContinue |
  Select-Object Name, PackageFullName, InstallLocation, Version, PackageFamilyName
```

检查 Codex 相关进程：

```powershell
Get-Process -Name "Codex" -ErrorAction SilentlyContinue |
  Select-Object ProcessName, Id, Path

Get-Process -Name "extension-host" -ErrorAction SilentlyContinue |
  Select-Object ProcessName, Id, Path
```

检查本地 marketplace 和插件缓存：

```powershell
$codex = "$env:LOCALAPPDATA\OpenAI\Codex\bin\fb2111b91430cb17\codex.exe"
& $codex plugin marketplace list
& $codex plugin list --marketplace openai-bundled
& $codex mcp list
```

检查包内是否真实存在 bundled 插件：

```powershell
$package = Get-AppxPackage -Name "OpenAI.Codex" -ErrorAction Stop
$pluginRoot = Join-Path $package.InstallLocation "app\resources\plugins\openai-bundled\plugins"
Get-ChildItem -LiteralPath $pluginRoot -Directory -ErrorAction Stop |
  Select-Object Name, FullName, LastWriteTime
```

检查 `config.toml` 是否残留旧版本：

```powershell
Select-String -LiteralPath "$HOME\.codex\config.toml" `
  -Pattern 'notify|openai-bundled|browser@openai-bundled|chrome@openai-bundled|computer-use@openai-bundled|26\.601|26\.602'
```

检查 Codex Desktop 日志：

```powershell
$logRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs"
$files = Get-ChildItem -LiteralPath $logRoot -Recurse -File -Filter "*.log" -ErrorAction Stop |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 20

Select-String -LiteralPath $files.FullName `
  -Pattern 'browser_use_availability_resolved|bundled_plugins_runtime_marketplace_written|bundled_plugins_marketplace_added|bundled_plugins_marketplace_resolve_failed|missing-helper-path|native pipe startup ready|native pipe startup failed|EBUSY|os error 5|os error 6000|computer-use notify config ensure' |
  ForEach-Object { "{0}:{1}: {2}" -f $_.Path, $_.LineNumber, $_.Line }
```

## 修复前备份

固定使用工作区：

```text
D:\codex_temp_workspace
```

创建目录：

```powershell
$workspace = "D:\codex_temp_workspace"
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$tempDir = Join-Path $workspace "Temp"
$backupDir = Join-Path $workspace "Backup"
$logsDir = Join-Path $workspace "Logs"

New-Item -Path $tempDir -ItemType Directory -Force -ErrorAction Stop | Out-Null
New-Item -Path $backupDir -ItemType Directory -Force -ErrorAction Stop | Out-Null
New-Item -Path $logsDir -ItemType Directory -Force -ErrorAction Stop | Out-Null
```

备份 Codex 用户配置：

```powershell
$taskBackupDir = Join-Path $backupDir "codex_plugin_fix_$timestamp"
New-Item -Path $taskBackupDir -ItemType Directory -Force -ErrorAction Stop | Out-Null

$items = @(
  "$HOME\.codex\config.toml",
  "$HOME\.codex\.codex-global-state.json",
  "$HOME\.codex\chrome-native-hosts-v2.json",
  "$HOME\.codex\computer-use\config.json"
)

foreach ($item in $items) {
  if (Test-Path -LiteralPath $item) {
    $leaf = Split-Path -Path $item -Leaf
    Copy-Item -LiteralPath $item `
      -Destination (Join-Path $taskBackupDir "$leaf.$timestamp.bak") `
      -Force `
      -ErrorAction Stop
  }
}
```

## 解除 Chrome 插件缓存文件锁

如果旧 `extension-host.exe` 正在运行，它可能锁住：

```text
C:\Users\<user>\.codex\plugins\cache\openai-bundled\chrome\latest\extension-host\windows\x64\extension-host.exe
```

停止它：

```powershell
$processes = Get-Process -Name "extension-host" -ErrorAction SilentlyContinue
if ($processes) {
  $processes | Select-Object ProcessName, Id, Path
  foreach ($process in $processes) {
    Stop-Process -Id $process.Id -Force -ErrorAction Stop
  }
}
```

注意：这会断开当前 Chrome 插件 host，但不会关闭 Chrome 浏览器本身。

## 复制 bundled 插件源到用户目录

不要直接从 `WindowsApps` 做插件安装。先复制一份到用户目录：

```powershell
$package = Get-AppxPackage -Name "OpenAI.Codex" -ErrorAction Stop
$sourceRoot = Join-Path $package.InstallLocation "app\resources\plugins\openai-bundled"
$destinationRoot = "$HOME\.codex\plugins\sources\openai-bundled-fixed"

if (Test-Path -LiteralPath $destinationRoot) {
  $backupRoot = Join-Path "D:\codex_temp_workspace\Backup" "openai-bundled-fixed_existing_$timestamp"
  Move-Item -LiteralPath $destinationRoot -Destination $backupRoot -ErrorAction Stop
}

New-Item -Path $destinationRoot -ItemType Directory -Force -ErrorAction Stop | Out-Null

$sourceRootFull = (Get-Item -LiteralPath $sourceRoot -ErrorAction Stop).FullName
$destinationRootFull = (Get-Item -LiteralPath $destinationRoot -ErrorAction Stop).FullName

Get-ChildItem -LiteralPath $sourceRootFull -Recurse -Directory -ErrorAction Stop | ForEach-Object {
  $relativePath = $_.FullName.Substring($sourceRootFull.Length).TrimStart('\')
  $targetDirectory = Join-Path $destinationRootFull $relativePath
  New-Item -Path $targetDirectory -ItemType Directory -Force -ErrorAction Stop | Out-Null
}

Get-ChildItem -LiteralPath $sourceRootFull -Recurse -File -ErrorAction Stop | ForEach-Object {
  $relativePath = $_.FullName.Substring($sourceRootFull.Length).TrimStart('\')
  $targetFile = Join-Path $destinationRootFull $relativePath
  $targetDirectory = Split-Path -Path $targetFile -Parent
  New-Item -Path $targetDirectory -ItemType Directory -Force -ErrorAction Stop | Out-Null
  [System.IO.File]::WriteAllBytes($targetFile, [System.IO.File]::ReadAllBytes($_.FullName))
}
```

本次复制结果：

```text
954 files
66.46 MB
```

## 重新注册 marketplace 并安装插件

```powershell
$codex = "$env:LOCALAPPDATA\OpenAI\Codex\bin\fb2111b91430cb17\codex.exe"
$source = "$HOME\.codex\plugins\sources\openai-bundled-fixed"

& $codex plugin marketplace remove openai-bundled
& $codex plugin marketplace add $source
& $codex plugin marketplace list
& $codex plugin list --marketplace openai-bundled

& $codex plugin add browser@openai-bundled
& $codex plugin add chrome@openai-bundled
& $codex plugin add computer-use@openai-bundled

& $codex plugin list --marketplace openai-bundled
```

期望结果类似：

```text
browser@openai-bundled       installed, enabled  26.602.30954
chrome@openai-bundled        installed, enabled  26.602.30954
computer-use@openai-bundled  installed, enabled  26.602.30954
```

`sites` 和 `latex` 可按需安装，不是本次 Chrome/Computer Use 修复必需项。

## 修复 Computer Use helper 路径

检查新版 helper 是否存在：

```powershell
$helper = "$HOME\.codex\plugins\cache\openai-bundled\computer-use\26.602.30954\node_modules\@oai\sky\bin\windows\codex-computer-use.exe"
Test-Path -LiteralPath $helper
```

检查 `config.toml` 顶部 `notify` 是否已经指向新版：

```powershell
Select-String -LiteralPath "$HOME\.codex\config.toml" -Pattern 'notify|26\.601|26\.602'
```

期望：

```toml
notify = [ "C:\\Users\\heyu\\.codex\\plugins\\cache\\openai-bundled\\computer-use\\26.602.30954\\node_modules\\@oai\\sky\\bin\\windows\\codex-computer-use.exe", "turn-ended" ]
```

本次安装插件后，Codex CLI 自动把该行修到了 `26.602.30954`。

## 修复 Chrome native messaging

Chrome 插件需要 native messaging manifest 和 HKCU 注册表项。

先检查：

```powershell
$node = "C:\Users\heyu\AppData\Local\OpenAI\Codex\bin\5b9024f90663758b\node.exe"
$checkScript = "$HOME\.codex\plugins\cache\openai-bundled\chrome\26.602.30954\scripts\check-native-host-manifest.js"
& $node $checkScript --json
```

如果返回：

```json
{
  "correct": false,
  "problem": "Windows native host registry key does not exist..."
}
```

就需要注册 Chrome native host。

### 高风险说明

这一步会修改 HKCU 用户注册表，不涉及 HKLM。

影响范围：

```text
HKCU\Software\Google\Chrome\NativeMessagingHosts\com.openai.codexextension
C:\Users\<user>\AppData\Local\OpenAI\extension\com.openai.codexextension.json
C:\Users\<user>\.codex\plugins\cache\openai-bundled\chrome\latest
```

先备份注册表：

```powershell
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupRoot = "D:\codex_temp_workspace\Backup\codex_chrome_registry_fix_$timestamp"
New-Item -Path $backupRoot -ItemType Directory -Force -ErrorAction Stop | Out-Null

$registryRoot = "HKCU\Software\Google\Chrome\NativeMessagingHosts"
$registryBackup = Join-Path $backupRoot "registry_HKCU_Chrome_NativeMessagingHosts_$timestamp.reg"

reg query $registryRoot *> $null
if ($LASTEXITCODE -eq 0) {
  reg export $registryRoot $registryBackup /y
} else {
  Set-Content -LiteralPath (Join-Path $backupRoot "registry_HKCU_Chrome_NativeMessagingHosts_was_missing_$timestamp.txt") `
    -Value "HKCU\Software\Google\Chrome\NativeMessagingHosts did not exist before repair." `
    -Encoding UTF8
}
```

创建 `latest` junction：

```powershell
$chromeCacheRoot = "$HOME\.codex\plugins\cache\openai-bundled\chrome"
$versionRoot = Join-Path $chromeCacheRoot "26.602.30954"
$latestPath = Join-Path $chromeCacheRoot "latest"

if (Test-Path -LiteralPath $latestPath) {
  $existingLatestBackup = Join-Path $backupRoot "chrome_latest_existing_$timestamp"
  Move-Item -LiteralPath $latestPath -Destination $existingLatestBackup -ErrorAction Stop
}

New-Item -Path $latestPath -ItemType Junction -Target $versionRoot -ErrorAction Stop | Out-Null
```

运行官方安装逻辑：

```powershell
$node = "C:\Users\heyu\AppData\Local\OpenAI\Codex\bin\5b9024f90663758b\node.exe"
$nodeRepl = "C:\Users\heyu\AppData\Local\OpenAI\Codex\bin\34ab3e1324cc55b5\node_repl.exe"
$codexCli = "C:\Users\heyu\AppData\Local\OpenAI\Codex\bin\fb2111b91430cb17\codex.exe"
$installManifest = Join-Path $versionRoot "scripts\installManifest.mjs"

$env:CODEX_INSTALL_MANIFEST = $installManifest
$env:CODEX_CLI_PATH_FOR_INSTALL = $codexCli
$env:CODEX_NODE_PATH_FOR_INSTALL = $node
$env:CODEX_NODE_REPL_PATH_FOR_INSTALL = $nodeRepl

& $node -e "const { pathToFileURL } = await import('node:url'); const mod = await import(pathToFileURL(process.env.CODEX_INSTALL_MANIFEST).href); await mod.install({ appServerRuntimePaths: { codexCliPath: process.env.CODEX_CLI_PATH_FOR_INSTALL, nodePath: process.env.CODEX_NODE_PATH_FOR_INSTALL, nodeReplPath: process.env.CODEX_NODE_REPL_PATH_FOR_INSTALL } });"
```

期望写出：

```text
C:\Users\<user>\AppData\Local\OpenAI\extension\com.openai.codexextension.json
```

内容类似：

```json
{
  "allowed_origins": [
    "chrome-extension://hehggadaopoacecdllhhajmbjkdcmajg/"
  ],
  "description": "Codex chrome native messaging host",
  "name": "com.openai.codexextension",
  "path": "C:\\Users\\heyu\\.codex\\plugins\\cache\\openai-bundled\\chrome\\latest\\extension-host\\windows\\x64\\extension-host.exe",
  "type": "stdio"
}
```

注册表期望：

```powershell
reg query "HKCU\Software\Google\Chrome\NativeMessagingHosts\com.openai.codexextension" /ve
```

输出应指向：

```text
C:\Users\<user>\AppData\Local\OpenAI\extension\com.openai.codexextension.json
```

## 最终验证

Chrome native host：

```powershell
& $node "$HOME\.codex\plugins\cache\openai-bundled\chrome\26.602.30954\scripts\check-native-host-manifest.js" --json
```

期望：

```json
{
  "correct": true,
  "problem": null
}
```

插件安装：

```powershell
& $codex plugin marketplace list
& $codex plugin list --marketplace openai-bundled
& $codex mcp list
```

关键路径存在：

```powershell
$helper = "$HOME\.codex\plugins\cache\openai-bundled\computer-use\26.602.30954\node_modules\@oai\sky\bin\windows\codex-computer-use.exe"
$latestHost = "$HOME\.codex\plugins\cache\openai-bundled\chrome\latest\extension-host\windows\x64\extension-host.exe"

Test-Path -LiteralPath $helper
Test-Path -LiteralPath $latestHost
```

日志验证：

```powershell
$logRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs"
$files = Get-ChildItem -LiteralPath $logRoot -Recurse -File -Filter "*.log" -ErrorAction Stop |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 8

Select-String -LiteralPath $files.FullName `
  -Pattern 'computer-use notify config ensure|native pipe startup ready|missing-helper-path|EBUSY|os error 6000' |
  ForEach-Object { "{0}:{1}: {2}" -f $_.Path, $_.LineNumber, $_.Line }
```

本次最终成功日志：

```text
computer-use notify config ensure finished platform=win32 status=repaired
computer-use native pipe startup ready pipePath=\\.\pipe\codex-computer-use-... platform=win32
```

## 回滚方法

文件回滚：

- 从 `D:\codex_temp_workspace\Backup\codex_plugin_fix_时间戳` 复制 `.bak` 文件回原路径。
- 如果曾移动旧 `openai-bundled-fixed` 或 `chrome\latest`，从对应 Backup 子目录移回。

注册表回滚：

```powershell
reg import "D:\codex_temp_workspace\Backup\codex_chrome_registry_fix_时间戳\registry_HKCU_Chrome_NativeMessagingHosts_时间戳.reg"
```

如果修复前该注册表分支不存在，可以删除本次新增的子项：

```powershell
reg delete "HKCU\Software\Google\Chrome\NativeMessagingHosts\com.openai.codexextension" /f
```

删除 manifest：

```powershell
Remove-Item -LiteralPath "C:\Users\heyu\AppData\Local\OpenAI\extension\com.openai.codexextension.json" -ErrorAction Stop
```

执行删除前应再次确认路径，避免误删。

## 下次更新后的快速判断

先看三个点，能少走弯路：

1. `codex plugin list --marketplace openai-bundled` 是否能列出插件。
2. `config.toml` 的 `notify` 是否还指向旧版本号。
3. `check-native-host-manifest.js --json` 是否 `correct: true`。

如果这三项分别表现为：

- marketplace 空；
- `notify` 是旧版本；
- native host registry/manifest 不存在；

就基本可以沿用本文流程。

如果日志里是 `available=false` 或 `reason=statsig-disabled`，那不是本地缓存修复能解决的问题，应该先等功能开关、账号权限或官方发布状态变化。

## 本次修复留下的一个低风险残留

`C:\Users\heyu\.codex\chrome-native-hosts-v2.json` 仍可能保留旧 `26.601` 记录。

本次没有手动改它，因为真实生效路径是：

- HKCU native messaging 注册表项
- `com.openai.codexextension.json`
- `chrome\latest` junction
- 当前 `config.toml`
- Codex 日志中的 `status=repaired`

只要最终检查 `correct=true` 且 Computer Use 日志显示 `native pipe startup ready`，这个旧 JSON 记录不应作为失败判断依据。

## 操作建议

修复完成后：

1. 完全退出 Codex Desktop，包括托盘。
2. 重新打开 Codex。
3. 新建线程测试 `@chrome` 和 Computer Use。
4. 如果 Chrome 插件仍不能连接，先重新运行 native host 检查脚本，再看 Chrome 扩展是否安装并启用。
