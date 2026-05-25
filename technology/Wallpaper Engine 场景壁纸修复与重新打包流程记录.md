## 任务背景

本次任务要修复一个 Wallpaper Engine 场景壁纸在笔记本屏幕上没有完全覆盖屏幕的问题。现象是图片底部出现灰色空白。

涉及路径：

- 解包后的可编辑目录：`D:\Users\贺宇\Downloads\3700944756`
- Wallpaper Engine 原始创意工坊目录：`E:\SteamLibrary\steamapps\workshop\content\431960\3700944756`
- RePKG 程序：`E:\APP\RePKG\RePKG.exe`
- Wallpaper Engine 安装目录：`E:\SteamLibrary\steamapps\common\wallpaper_engine`

最终处理结果：

- 修改了 `D:\Users\贺宇\Downloads\3700944756\scene.json`
- 生成了 `D:\Users\贺宇\Downloads\3700944756\scene_fixed.pkg`
- 生成了可导入目录 `D:\Users\贺宇\Downloads\3700944756\modified_pkg`
- 将生成好的文件覆盖到 `E:\SteamLibrary\steamapps\workshop\content\431960\3700944756`

## 使用到的工具

### 1. PowerShell

用于：

- 检查目录结构
- 读取 JSON 文件
- 复制文件
- 调用外部 exe
- 手写 PKG 打包脚本
- 验证文件和进程状态

### 2. RePKG

路径：

```powershell
E:\APP\RePKG\RePKG.exe
```

实际可用能力：

```powershell
& 'E:\APP\RePKG\RePKG.exe' --help
```

输出显示该版本只支持：

- `extract`
- `info`
- `help`
- `version`

注意：本机这份 RePKG 0.4.0 不支持 `convert` 和 `pack`。

验证命令：

```powershell
& 'E:\APP\RePKG\RePKG.exe' convert --help
& 'E:\APP\RePKG\RePKG.exe' pack --help
```

结果：

- `Verb 'convert' is not recognized.`
- `Verb 'pack' is not recognized.`

因此不能直接使用网上常见的：

```bash
repkg convert -r . -f tex -o ../new_pkg
repkg pack . -o ../modified.pkg
```

### 3. Wallpaper Engine resourcecompiler

发现路径：

```powershell
E:\SteamLibrary\steamapps\common\wallpaper_engine\bin\resourcecompiler64.exe
```

尝试过：

```powershell
& 'E:\SteamLibrary\steamapps\common\wallpaper_engine\bin\resourcecompiler64.exe' --help
```

结果：

```text
Starting compiler: unsupported mode
result: -1
```

随后通过字符串检查发现它支持 `-pkg` 模式，但直接打包失败。

尝试命令：

```powershell
& 'E:\SteamLibrary\steamapps\common\wallpaper_engine\bin\resourcecompiler64.exe' -pkg -i 'D:\Users\贺宇\Downloads\3700944756' -o 'D:\Users\贺宇\Downloads\3700944756\scene_fixed.pkg'
```

结果失败，并且路径被截断到：

```text
D:\Users\
result: -1
```

后续也尝试过相对路径：

```powershell
& 'E:\SteamLibrary\steamapps\common\wallpaper_engine\bin\resourcecompiler64.exe' -pkg -i . -o scene_fixed.pkg
```

仍失败：

```text
Starting compiler: package encoder
. - scene_fixed.pkg
result: -1
```

结论：本次不使用 resourcecompiler 完成打包。

### 4. RePKG 源码参考

为了确认 PKG 文件格式，使用 PowerShell 拉取 RePKG 源码中的关键文件：

```powershell
(Invoke-WebRequest -UseBasicParsing -Uri 'https://raw.githubusercontent.com/notscuffed/repkg/master/RePKG.Application/Package/PackageWriter.cs' -TimeoutSec 20).Content
```

源码中的 `PackageWriter` 逻辑很简单：

1. 写入字符串魔术头 `PKGV0005`
2. 写入文件条目数量
3. 写入每个条目的路径、偏移、长度
4. 顺序写入所有文件内容

对应源码参考：

- `RePKG.Application/Package/PackageWriter.cs`
- `RePKG.Application/Package/PackageReader.cs`
- `RePKG.Application/Extensions.cs`

重要细节：

- 字符串写法是 `WriteStringI32Size`
- 先写 `int32` 长度，再写 UTF-8 字节
- RePKG 源码里长度用的是 `input.Length`
- 本次包内路径全部是 ASCII 字符，所以没有多字节路径问题
- 如果以后包内资源路径包含中文或其它非 ASCII 字符，需要谨慎验证

## 问题定位过程

### 1. 检查解包目录

命令：

```powershell
Get-ChildItem -LiteralPath 'D:\Users\贺宇\Downloads\3700944756' -Force | Format-List Name,FullName,Mode,Length,LastWriteTime
```

发现目录结构：

```text
effects
materials
models
shaders
preview.jpg
project.json
scene.json
```

说明这是 Wallpaper Engine 的 Scene 项目，不是单纯静态图片。

### 2. 读取 scene.json

命令：

```powershell
Get-Content -LiteralPath 'D:\Users\贺宇\Downloads\3700944756\scene.json' -Raw
```

关键内容：

```json
"orthogonalprojection" :
{
  "height" : 2873,
  "width" : 5120
},
...
"origin" : "2560.00000 1580.00000 0.00000",
"size" : "5120.00000 2873.00000"
```

判断逻辑：

- 画布宽高：`5120 x 2873`
- 对象尺寸：`5120 x 2873`
- X 中心点正确：`5120 / 2 = 2560`
- Y 中心点应该是：`2873 / 2 = 1436.5`
- 实际 Y 中心点是：`1580`

差值：

```text
1580 - 1436.5 = 143.5
```

这表示画面整体上移了约 `143.5px`，底部自然露出灰色背景。

## 实际修改

先备份：

```powershell
$src='D:\Users\贺宇\Downloads\3700944756\scene.json'
$bak='D:\Users\贺宇\Downloads\3700944756\scene.json.bak'
if (-not (Test-Path -LiteralPath $bak)) {
  Copy-Item -LiteralPath $src -Destination $bak
}
```

修改前：

```json
"origin" : "2560.00000 1580.00000 0.00000"
```

修改后：

```json
"origin" : "2560.00000 1436.50000 0.00000"
```

验证命令：

```powershell
Get-Content -LiteralPath 'D:\Users\贺宇\Downloads\3700944756\scene.json' -Raw |
  ConvertFrom-Json |
  Select-Object -ExpandProperty objects |
  Select-Object origin,size |
  Format-List
```

预期结果：

```text
origin : 2560.00000 1436.50000 0.00000
size   : 5120.00000 2873.00000
```

## 重新打包逻辑

因为本机 RePKG 不支持 `pack`，Wallpaper Engine 的 `resourcecompiler64.exe -pkg` 也无法正常打包，所以采用手写 PKG 的方式。

打包时不要把这些文件放进 `scene.pkg`：

- `project.json`
- `preview.jpg`
- `scene.json.bak`
- `.png`
- `.jpg`
- `.pkg`
- `.tex-json`
- 临时输出目录

需要放进 `scene.pkg` 的核心文件：

- `scene.json`
- `models/*.json`
- `materials/*.json`
- `materials/*.tex`
- `effects/**/*.json`
- `shaders/**/*.frag`
- `shaders/**/*.vert`

本次最终包内条目：

```text
scene.json
models/_cgi-bin_mmwebwx-bin_webwxgetmsgimg__&MsgID=4442508654530627173&skey=@crypt_ec627672_d751d83fbe4f7da38a3d4f6c3d8785ab&mmweb_appid=wx_webfilehelper.json
materials/_cgi-bin_mmwebwx-bin_webwxgetmsgimg__&MsgID=4442508654530627173&skey=@crypt_ec627672_d751d83fbe4f7da38a3d4f6c3d8785ab&mmweb_appid=wx_webfilehelper.json
materials/_cgi-bin_mmwebwx-bin_webwxgetmsgimg__&MsgID=4442508654530627173&skey=@crypt_ec627672_d751d83fbe4f7da38a3d4f6c3d8785ab&mmweb_appid=wx_webfilehelper.tex
materials/effects/xray.json
materials/masks/shimmer_mask_20a6665c.tex
materials/masks/xray_mask_11c58c9f.tex
effects/xray/effect.json
shaders/effects/xray.frag
shaders/effects/xray.vert
```

## 手写 PKG 打包脚本

在 PowerShell 中执行：

```powershell
$root='D:\Users\贺宇\Downloads\3700944756'
$out=Join-Path $root 'scene_fixed.pkg'
$excludeNames=@('project.json','preview.jpg','scene.json.bak')
$prefix=$root.TrimEnd('\') + '\'

$files=Get-ChildItem -LiteralPath $root -Recurse -File | Where-Object {
  $full=$_.FullName
  $rel=$full.Substring($prefix.Length)
  $rel -notmatch '(^|[\\/])(pack_out2?|modified_pkg)([\\/]|$)' -and
  $_.Extension -notin @('.png','.jpg','.pkg','.tex-json') -and
  $_.Name -notin $excludeNames
}

$entries=$files | ForEach-Object {
  $rel=$_.FullName.Substring($prefix.Length).Replace('\','/')
  $rank=if($rel -eq 'scene.json'){
    0
  }elseif($rel -like 'models/*'){
    1
  }elseif($rel -like 'materials/*'){
    2
  }elseif($rel -like 'effects/*'){
    3
  }elseif($rel -like 'shaders/*'){
    4
  }else{
    9
  }
  [pscustomobject]@{
    Path=$rel
    Rank=$rank
    FullName=$_.FullName
    Bytes=[IO.File]::ReadAllBytes($_.FullName)
  }
} | Sort-Object Rank,Path

$fs=[IO.File]::Open($out,[IO.FileMode]::Create,[IO.FileAccess]::Write,[IO.FileShare]::None)
try {
  $bw=New-Object IO.BinaryWriter($fs,[Text.Encoding]::UTF8,$true)

  function Write-I32String([IO.BinaryWriter]$bw,[string]$s){
    $bytes=[Text.Encoding]::UTF8.GetBytes($s)
    $bw.Write([int]$s.Length)
    $bw.Write($bytes)
  }

  Write-I32String $bw 'PKGV0005'
  $bw.Write([int]$entries.Count)

  $offset=0
  foreach($e in $entries){
    Write-I32String $bw $e.Path
    $bw.Write([int]$offset)
    $bw.Write([int]$e.Bytes.Length)
    $offset += $e.Bytes.Length
  }

  foreach($e in $entries){
    $bw.Write($e.Bytes)
  }

  $bw.Flush()
} finally {
  $fs.Dispose()
}
```

生成文件：

```text
D:\Users\贺宇\Downloads\3700944756\scene_fixed.pkg
```

## 构造可导入目录

Wallpaper Engine 创意工坊目录通常是：

```text
project.json
preview.jpg
scene.pkg
```

其中 `project.json` 里可能写的是：

```json
"file" : "scene.json"
```

这不矛盾，因为实际的 `scene.json` 在 `scene.pkg` 包内。

构造目录命令：

```powershell
$root='D:\Users\贺宇\Downloads\3700944756'
$dir=Join-Path $root 'modified_pkg'

if(Test-Path -LiteralPath $dir){
  Remove-Item -LiteralPath $dir -Recurse -Force
}

New-Item -ItemType Directory -Path $dir | Out-Null
Copy-Item -LiteralPath (Join-Path $root 'scene_fixed.pkg') -Destination (Join-Path $dir 'scene.pkg')
Copy-Item -LiteralPath (Join-Path $root 'project.json') -Destination (Join-Path $dir 'project.json')
Copy-Item -LiteralPath (Join-Path $root 'preview.jpg') -Destination (Join-Path $dir 'preview.jpg')
```

生成目录：

```text
D:\Users\贺宇\Downloads\3700944756\modified_pkg
```

目录内文件：

```text
preview.jpg
project.json
scene.pkg
```

## 验证 PKG 是否可读

命令：

```powershell
& 'E:\APP\RePKG\RePKG.exe' info -e 'D:\Users\贺宇\Downloads\3700944756\modified_pkg\scene.pkg'
```

预期输出中应该能看到：

```text
Package entries:
* scene.json - 2373 bytes
...
Done
```

这只能说明 PKG 格式可被 RePKG 识别，不等于 Wallpaper Engine 渲染一定正确。最终仍需要在 Wallpaper Engine 中实际加载查看。

## 覆盖 Wallpaper Engine 原目录

用户提供原始壁纸目录：

```text
E:\SteamLibrary\steamapps\workshop\content\431960\3700944756
```

先检查：

```powershell
Get-ChildItem -LiteralPath 'E:\SteamLibrary\steamapps\workshop\content\431960\3700944756' -Force |
  Select-Object Name,Length,LastWriteTime |
  Format-List
```

目标目录已有：

```text
shaders
preview.jpg
project.json
scene.pkg
```

覆盖命令：

```powershell
$src='D:\Users\贺宇\Downloads\3700944756\modified_pkg'
$dst='E:\SteamLibrary\steamapps\workshop\content\431960\3700944756'

foreach($name in @('scene.pkg','project.json','preview.jpg')){
  Copy-Item -LiteralPath (Join-Path $src $name) -Destination (Join-Path $dst $name) -Force
}

Get-ChildItem -LiteralPath $dst -Force |
  Select-Object Name,Length,LastWriteTime |
  Format-List
```

注意：

- 只覆盖 `scene.pkg`、`project.json`、`preview.jpg`
- 保留目标目录原有 `shaders` 文件夹不动
- 文件命名必须保持 `scene.pkg`，不要使用 `scene_fixed.pkg`

覆盖后再次验证：

```powershell
& 'E:\APP\RePKG\RePKG.exe' info -e 'E:\SteamLibrary\steamapps\workshop\content\431960\3700944756\scene.pkg'
```

## 本次踩坑记录

### 1. 不要假设 RePKG 有 pack

网上有些教程会写：

```bash
repkg pack . -o modified.pkg
```

但本机的 RePKG 0.4.0 不支持该命令。执行前必须先检查：

```powershell
& 'E:\APP\RePKG\RePKG.exe' --help
```

### 2. `repkg` 不一定在 PATH

本机执行：

```powershell
Get-Command repkg -ErrorAction SilentlyContinue
repkg --help
```

结果是命令不存在。因此要使用绝对路径：

```powershell
& 'E:\APP\RePKG\RePKG.exe'
```

### 3. resourcecompiler 对中文路径或参数格式不友好

尝试 `resourcecompiler64.exe -pkg` 时失败。尤其当路径中包含中文用户名时，它似乎会截断路径。

不要把它作为可靠打包方案，除非已经明确知道正确参数和路径兼容性。

### 4. PowerShell/.NET 版本差异

一开始尝试使用：

```powershell
[IO.Path]::GetRelativePath($root,$_.FullName)
```

但当前环境不支持该方法。

兼容写法是：

```powershell
$prefix=$root.TrimEnd('\') + '\'
$rel=$_.FullName.Substring($prefix.Length).Replace('\','/')
```

### 5. PKG 内不应包含 PNG 和 tex-json

RePKG 解包时会把 `.tex` 转成 `.png`，还会生成 `.tex-json`。如果只是改 `scene.json`，不需要重新转 TEX。

本次打包只保留原始 `.tex`，不要把 `.png` 或 `.tex-json` 放进包里。

## 下次执行类似任务的推荐流程

1. 先确认用户授权，因为会写入工作区外的 Steam/下载目录。
2. 检查解包目录结构：

```powershell
Get-ChildItem -LiteralPath '<解包目录>' -Force
```

3. 读取 `scene.json`，重点看：

```json
orthogonalprojection
objects[].origin
objects[].size
```

4. 如果是图片底部留白，优先检查：

```text
origin.y 是否等于 size.height / 2
origin.x 是否等于 size.width / 2
```

5. 修改前备份：

```powershell
Copy-Item -LiteralPath '<解包目录>\scene.json' -Destination '<解包目录>\scene.json.bak'
```

6. 修改 `scene.json`。
7. 重新生成 `scene.pkg`。
8. 生成 `modified_pkg`，确保文件名是：

```text
scene.pkg
project.json
preview.jpg
```

9. 用 RePKG 验证：

```powershell
& 'E:\APP\RePKG\RePKG.exe' info -e '<输出目录>\scene.pkg'
```

10. 覆盖 Wallpaper Engine 原目录。
11. 再次验证目标目录中的 `scene.pkg`。
12. 让用户重启 Wallpaper Engine 或重新选择该壁纸查看效果。

## 本次外部文件变更清单

修改或创建过的文件：

```text
D:\Users\贺宇\Downloads\3700944756\scene.json
D:\Users\贺宇\Downloads\3700944756\scene.json.bak
D:\Users\贺宇\Downloads\3700944756\scene_fixed.pkg
D:\Users\贺宇\Downloads\3700944756\modified_pkg\scene.pkg
D:\Users\贺宇\Downloads\3700944756\modified_pkg\project.json
D:\Users\贺宇\Downloads\3700944756\modified_pkg\preview.jpg
E:\SteamLibrary\steamapps\workshop\content\431960\3700944756\scene.pkg
E:\SteamLibrary\steamapps\workshop\content\431960\3700944756\project.json
E:\SteamLibrary\steamapps\workshop\content\431960\3700944756\preview.jpg
```

本笔记文件：

```text
D:\Obsidian\note\technology\Wallpaper Engine 场景壁纸修复与重新打包流程记录.md
```
