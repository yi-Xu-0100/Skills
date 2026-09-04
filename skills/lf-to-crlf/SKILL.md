---
name: lf-to-crlf
description: '将 Git 修改文件中的 LF 换行符转换为 CRLF。当 Git 报错 "fatal: LF would be replaced by CRLF" 或类似换行符相关错误导致 add/commit/checkout 等操作失败时必须触发此 skill。也适用于手动将 Git 仓库中文件的换行符从 LF 批量转换为 CRLF 的场景（跨平台协作中 Windows 开发者处理 Unix 风格换行符）。'
---

# LF → CRLF 换行符转换

将 Git 仓库中文件的换行符从 LF 转换为 CRLF，解决 Windows 环境下 Git 因 `core.autocrlf` 配置导致的换行符冲突问题。

## 环境检测

执行任何命令前，先判断当前运行环境：

- **Windows (PowerShell)**：使用 PowerShell 语法执行所有命令
- **Linux / macOS (Bash)**：使用 Bash 语法执行所有命令

下文命令块中同时提供了两种环境的写法，`PS>` 前缀表示 PowerShell，`$` 前缀表示 Bash。根据检测到的环境选择对应的命令执行。

## ⚠️ 编码红线（Windows/PowerShell 必读）

本 skill 处理的多为 **UTF-8（含中文）** 文件。PowerShell 读取必须**显式按 UTF-8**：

- 用 `[System.IO.File]::ReadAllText(path, (New-Object System.Text.UTF8Encoding $false))` 读取，该方法自动识别并去除 UTF-8 BOM。
- **禁止使用 `Get-Content`（不带 `-Encoding`）读取文件**：Windows PowerShell 5.1 下其默认按 ANSI 代码页解码（中文系统 = gb2312），会把 UTF-8 文件的中文读成乱码；若再按 UTF-8 写回即造成**不可逆的双重编码损坏**。即使 PowerShell 7 默认 UTF-8，为兼容 5.1 也统一不用 `Get-Content`。
- 检测换行符可不解码文本：把字节按 ASCII 解释即可（CR/LF 均为 ASCII 字节，任何源编码下判断都准确）。
- 写入固定用 `WriteAllText + UTF8Encoding($false)`，输出 UTF-8 无 BOM。

## 触发场景

### 场景 A：Git 操作被 LF/CRLF 错误阻止

当用户执行 `git add`、`git commit`、`git checkout` 等操作时遇到如下错误：

```text
fatal: LF would be replaced by CRLF in <file-path>
```

此时 Git 明确指出了问题文件路径，直接进入 [针对 Git 报错文件的转换流程](#针对-git-报错文件的转换流程)。

### 场景 B：主动批量转换仓库中的 LF 文件

用户希望手动将当前仓库中所有变更文件（已暂存、未暂存、未跟踪）的换行符从 LF 转换为 CRLF，进入 [变更文件批量转换流程](#变更文件批量转换流程)。

---

## 针对 Git 报错文件的转换流程

当 Git 报错中包含 `fatal: LF would be replaced by CRLF in <file>` 时：

### 1. 提取报错中的文件路径

从 Git 报错信息中提取文件路径。如果有多条报错，提取所有文件路径。

### 2. 转换为 CRLF

对提取出的文件执行换行符转换（PowerShell 产出 UTF-8 without BOM）：

```powershell
# PS> 单文件：显式 UTF-8 读取（自动去除 BOM）→ 替换换行符 → 写回（UTF-8 无 BOM）
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
$content = [System.IO.File]::ReadAllText((Resolve-Path <file>).Path, $utf8NoBom)
$content = $content -replace '\r?\n', "`r`n"
[System.IO.File]::WriteAllText((Resolve-Path <file>).Path, $content, $utf8NoBom)
```

```bash
# $ 单文件
perl -pi -e 's/\r?\n/\r\n/' <file>
```

如果文件数量较多，循环逐个转换：

```powershell
# PS> 逐文件转换（切勿改用 Get-Content，见「编码红线」）
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
@('<file1>', '<file2>', '<file3>') | ForEach-Object {
  $content = [System.IO.File]::ReadAllText((Resolve-Path $_).Path, $utf8NoBom)
  $content = $content -replace '\r?\n', "`r`n"
  [System.IO.File]::WriteAllText((Resolve-Path $_).Path, $content, $utf8NoBom)
}
```

```bash
# $ 逐文件转换
for f in <file1> <file2> <file3>; do perl -pi -e 's/\r?\n/\r\n/' "$f"; done
```

### 3. 验证并重试 Git 操作

转换完成后，验证换行符并重新执行之前被阻止的 Git 操作：

```powershell
# PS> 检查文件是否包含 CRLF
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
$raw = [System.IO.File]::ReadAllText((Resolve-Path <file>).Path, $utf8NoBom)
Write-Host "$(if ($raw -match "`r`n") { 'CRLF ✓' } else { 'UNKNOWN ✗' }) : <file>"
```

```bash
# $ 使用 file 命令确认
file <file>
```

验证通过后重新执行 `git add` 等之前被阻止的操作。

---

## 变更文件批量转换流程

### 1. 获取变更文件列表

获取已跟踪且已修改的文件（包括已暂存和未暂存），以及未跟踪文件（两环境通用）：

```powershell
# PS> 获取变更文件
$modified = git diff --name-only HEAD
$untracked = git ls-files --others --exclude-standard
```

```bash
# $ 获取变更文件
modified=$(git diff --name-only HEAD)
untracked=$(git ls-files --others --exclude-standard)
```

合并这两个列表，去重，作为后续步骤的输入。

### 2. 检测 LF 编码文件

对文件列表中的每个文件，检查其是否包含 CRLF 换行符。已含 `\r\n` 的为 CRLF 格式，否则为 LF 格式。

```powershell
# PS> 检测 LF 文件：按字节/ASCII 判断换行符，不解码文本，任何源编码都安全
$lfFiles = @()
foreach ($f in $allFiles) {
  if (-not (Test-Path $f)) { continue }
  $bytes = [System.IO.File]::ReadAllBytes((Resolve-Path $f).Path)
  $ascii = [System.Text.Encoding]::ASCII.GetString($bytes)
  if ($ascii -notmatch "`r`n") {
    $lfFiles += $f
  }
}
```

```bash
# $ 使用 file 命令检测
lf_files=()
for f in $all_files; do
  if file "$f" 2>/dev/null | grep -v 'CRLF' | grep -q 'text'; then
    lf_files+=("$f")
  fi
done
```

只保留检测为 LF 格式的文件进入下一步。

### 3. 转换为 CRLF

对检测出的 LF 文件执行换行符转换（PowerShell 产出 UTF-8 without BOM）：

```powershell
# PS> 逐文件显式 UTF-8 读取 → 替换换行符 → 写回（UTF-8 无 BOM）
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
$lfFiles | ForEach-Object {
  $content = [System.IO.File]::ReadAllText((Resolve-Path $_).Path, $utf8NoBom)
  $content = $content -replace '\r?\n', "`r`n"
  [System.IO.File]::WriteAllText((Resolve-Path $_).Path, $content, $utf8NoBom)
}
```

```bash
# $ 逐文件转换（依赖 perl，Linux/macOS 默认已安装）
for f in "${lf_files[@]}"; do perl -pi -e 's/\r?\n/\r\n/' "$f"; done
```

### 4. 验证转换结果

确认所有被转换的文件现在为 CRLF 编码：

```powershell
# PS> 验证（显式 UTF-8 读取）
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
foreach ($f in $lfFiles) {
  $raw = [System.IO.File]::ReadAllText((Resolve-Path $f).Path, $utf8NoBom)
  Write-Host "$f : $(if ($raw -match "`r`n") { 'CRLF' } else { 'UNKNOWN' })"
}
```

```bash
# $ 验证
for f in "${lf_files[@]}"; do
  echo "$f : $(file "$f")"
done
```

打印转换结果摘要，列出已处理文件及其编码状态。

## 注意事项

- 仅作用于**当前 Git 仓库**内的文件 — 使用 `git` 命令自动限定到当前工作目录
- 跳过已经为 CRLF 编码的文件，不做重复转换
- 如果文件列表为空，提前结束并提示用户
- **Windows 环境**：使用纯 PowerShell + .NET 方案，PS 5.1 / 7+ 均可用，零外部依赖
  - 读取：`[System.IO.File]::ReadAllText(path, (New-Object System.Text.UTF8Encoding $false))` —— 显式 UTF-8，自动识别并去除 BOM
  - 写入：`[System.IO.File]::WriteAllText(path, content, 同上的 UTF8Encoding $false)` —— 输出 UTF-8 无 BOM
  - **不要用 `Get-Content` 读取**：PS 5.1 默认按 ANSI（中文系统 gb2312）解码，会把 UTF-8 中文变成不可逆乱码（真实事故教训）
  - `-replace '\r?\n', "\`r\`n"` 将任意换行符统一转为 CRLF
- **Linux/macOS 环境**：使用 `perl -pi -e` 原地转换
- `git` 子命令在两环境下语法一致，无需区分
