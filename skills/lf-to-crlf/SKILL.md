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
# PS> 单文件：读取 → 检测并去除 BOM → 替换换行符 → 写回（UTF-8 无 BOM）
$content = Get-Content <file> -Raw
# 若首字符为 UTF-8 BOM（U+FEFF）则移除；无 BOM 时跳过，不会产生异常
if ($content -and $content[0] -eq [char]0xFEFF) { $content = $content.Substring(1) }
$content = $content -replace '\r?\n', "`r`n"
# .NET WriteAllText + UTF8Encoding($false) 写入 UTF-8 无 BOM，兼容 PS 5.1 / 7+
[System.IO.File]::WriteAllText((Resolve-Path <file>).Path, $content, (New-Object System.Text.UTF8Encoding $false))
```

```bash
# $ 单文件
perl -pi -e 's/\r?\n/\r\n/' <file>
```

如果文件数量较多，循环逐个转换：

```powershell
# PS> 逐文件转换
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
@('<file1>', '<file2>', '<file3>') | ForEach-Object {
  $content = Get-Content $_ -Raw
  if ($content -and $content[0] -eq [char]0xFEFF) { $content = $content.Substring(1) }
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
# PS> 检查文件是否包含 CRLF（先检测并去除可能的 BOM 避免干扰）
$raw = Get-Content <file> -Raw
if ($raw -and $raw[0] -eq [char]0xFEFF) { $raw = $raw.Substring(1) }
$hasCRLF = $raw -match "`r`n"
Write-Host "$(if ($hasCRLF) { 'CRLF ✓' } else { 'UNKNOWN ✗' }) : <file>"
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
# PS> 检测 LF 文件：不包含 \r\n 的文件即为 LF
$lfFiles = @()
foreach ($f in $allFiles) {
  if (-not (Test-Path $f)) { continue }
  $content = Get-Content $f -Raw -ErrorAction SilentlyContinue
  if ($content -and $content -notmatch "`r`n") {
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
# PS> 逐文件读取 → 检测并去除 BOM → 替换换行符 → 写回（UTF-8 无 BOM）
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
$lfFiles | ForEach-Object {
  $content = Get-Content $_ -Raw
  if ($content -and $content[0] -eq [char]0xFEFF) { $content = $content.Substring(1) }
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
# PS> 验证
foreach ($f in $lfFiles) {
  $raw = Get-Content $f -Raw
  if ($raw -and $raw[0] -eq [char]0xFEFF) { $raw = $raw.Substring(1) }
  $hasCRLF = $raw -match "`r`n"
  Write-Host "$f : $(if ($hasCRLF) { 'CRLF' } else { 'UNKNOWN' })"
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
  - 写入使用 `[System.IO.File]::WriteAllText()` + `UTF8Encoding($false)`，确保输出 UTF-8 无 BOM
  - 读取后以 `$content[0] -eq [char]0xFEFF` 显式检测 UTF-8 BOM，存在才移除，不存在则跳过
  - `-replace '\r?\n', "\`r\`n"` 将任意换行符统一转为 CRLF
- **Linux/macOS 环境**：使用 `perl -pi -e` 原地转换
- `git` 子命令在两环境下语法一致，无需区分
