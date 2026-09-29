---
name: git-commit
description: '按项目规范创建 Git 提交。当用户说 "commit"、"提交"、"git commit" 或请求创建代码提交时必须触发此 skill。自动分析变更、生成符合规范的提交信息，经用户确认后执行提交。提交信息默认写入临时文件（UTF-8 无 BOM + LF）后用 git commit -F 提交，避免命令行编码弄乱中文；多份提交涉及 GPG 签名时一次提权、多次使用。支持标准提交类型 (feat/fix/docs/style/refactor/test/chore/perf) 和中文主题。'
---

# Git 提交助手

自动分析 Git 变更、生成符合项目规范的提交信息，经确认后执行提交。

## 提交格式

```text
<type>(<scope>): <subject>

<body>
```

### 提交格式规范说明

- type 使用以下之一：`feat` `fix` `docs` `style` `refactor` `test` `chore` `perf`
- scope 为改动范围（可选），如 `git` `build` `skill-name`
- subject 使用中文，不超过 50 字，命令语气，不加句号
- 一次提交只解决一个问题，保持粒度小且明确
- 标题与正文之间空一行。
- 正文包含多条独立信息时（≥2 条），**必须**使用 `-` 列表分条说明，每条保持简洁。仅单条信息时可写为段落。

### 提交格式示例

分条说明示例：

```text
feat(git-commit): 细化提交模板，增加正文规范

- 提交格式从仅标题扩展为标题 + 正文
- 正文说明做了什么、为什么这样做
- 提交命令改用临时文件 + git commit -F
```

单条说明示例：

```text
fix(build): 修复 PowerShell 下路径转义错误

build.ps1 中路径引号改为双引号。
```

## 执行流程

### 1. 检查当前状态

```powershell
PS> git status
```

```bash
$ git status
```

```powershell
PS> git diff
```

```bash
$ git diff
```

### 2. 分析变更并生成提交信息

根据 `git diff` 的内容分析改动性质和范围：

- **新增功能** → `feat`
- **修复问题** → `fix`
- **纯文档修改** → `docs`
- **格式/空白调整** → `style`
- **代码重构（不改变行为）** → `refactor`
- **测试相关** → `test`
- **构建/工具/依赖** → `chore`
- **性能优化** → `perf`

scope 根据改动文件路径推断，如 `lf-to-crlf` `git` `build` 等。若改动涉及多个不相关的范围，应拆分为多次提交。

按照**提交格式**生成提交信息并展示给用户确认。格式如下：

```text
建议提交信息：

  <type>(<scope>): <subject>

  <body>
```

改动文件少时可在末尾列出文件名；文件多时按功能分组说明每组改了什么：
- 不逐个罗列文件路径
- 按改动模块或目的分组，用一两句说明该组文件做了什么

### 3. 提交信息走临时文件，用 git commit -F 提交

**优先不让提交信息经过命令行**：命令行参数会受终端/shell 代码页影响，中文很容易乱码。统一按下面的流程：

1. 把消息写成**临时文件**，内容规整为 **UTF-8 无 BOM + LF**；
2. 用 `git commit -F <文件>` 提交；
3. 提交完成后**删除临时文件**。

规整要求：编码 UTF-8（无 BOM）、换行一律 LF、文件放系统临时目录（不要放进仓库，避免残留未跟踪文件）、一个提交对应一个文件。

PowerShell（PS 5.1 / 7+ 通用）：

```powershell
# 1) 写提交信息：用 [char]10（LF）连接各行，天然无 CR、无 BOM
$msgPath = Join-Path $env:TEMP 'git-commit-msg.txt'
$lines = @(
  'feat(scope): 中文主题',
  '',
  '- 第一条说明',
  '- 第二条说明'
)
$msg = [string]::Join([char]10, $lines)
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText($msgPath, $msg, $utf8NoBom)

# 2) 暂存并提交
git add <files>
git commit -F $msgPath

# 3) 删除临时文件
Remove-Item -Force $msgPath
```

Bash：

```bash
# 1) 写提交信息：heredoc 天然 LF、无 BOM
msgFile="$(mktemp)"
cat > "$msgFile" <<'EOF'
feat(scope): 中文主题

- 第一条说明
- 第二条说明
EOF

# 2) 暂存并提交
git add <files>
git commit -F "$msgFile"

# 3) 删除临时文件
rm -f "$msgFile"
```

- PowerShell **不要用 `Set-Content` 写消息文件**：PS 5.1 会写成 UTF-16 LE 或带 BOM 的 UTF-8；用 `WriteAllText` + `UTF8Encoding $false` 才稳定。
- Bash 用 `<<'EOF'`（引号防止 `$`、反引号被展开），输出天然符合要求。
- **写消息文件、`git commit -F`、删除临时文件要在同一次执行（同一个 shell 会话）里完成**：受限环境可能按次隔离系统临时目录，普通执行与提权执行看到的临时目录并不相同，跨执行写入的文件在后一次里读不到，`git commit` 会报 `could not read log file ...`。
- 仅当消息确定为**纯 ASCII 且单行**时，才可直接 `git commit -m "<subject>"`；含中文或多行一律走文件。

### 4. GPG 签名：一次提权、多次使用

`git config commit.gpgsign` 为 `true` 时每笔提交都要 GPG 签名。受限环境（沙箱）下 gpg.exe 可能因无法创建 msys 信号管道而崩溃（`gpg failed to sign the data` / `couldn't create signal pipe` / `Win32 error 5`），此时按 **gpg-sign** skill 处理：**不得用 `--no-gpg-sign` 绕过签名**，应升级权限重试。

需要签名的提交**不止一笔**时，不要每笔单独提权（每次提权都要用户授权一次）：

1. 先与用户敲定全部提交：确定每笔的 `git add` 范围与完整提交信息（消息文件先不落盘）；
2. 在**同一次提权执行**里逐笔「写消息文件 → `git commit -F`」；写文件与提交必须在同一次执行内完成——否则提权执行读不到此前写的文件，这样才能一次授权、多次使用；
3. 所有提交成功后再统一删除全部临时文件。

```powershell
# 同一次提权执行内完成多笔签名提交：每笔都是「写消息文件 → git commit -F」
$utf8NoBom = New-Object System.Text.UTF8Encoding $false

$m1 = Join-Path $env:TEMP 'commit-1.txt'
$s1 = [string]::Join([char]10, @('feat(a): 主题一', '', '- 说明'))
[System.IO.File]::WriteAllText($m1, $s1, $utf8NoBom)
git commit -F $m1

$m2 = Join-Path $env:TEMP 'commit-2.txt'
$s2 = [string]::Join([char]10, @('docs(b): 主题二', '', '- 说明'))
[System.IO.File]::WriteAllText($m2, $s2, $utf8NoBom)
git commit -F $m2

Remove-Item -Force $m1, $m2
```

若中途某笔失败，先查明原因再决定是否补一次提权；任何时候都不要改用 `--no-gpg-sign`（用户明确要求除外）。

### 5. 验证提交结果

```powershell
PS> git status
```

```bash
$ git status
```

## 注意事项

- 提交前必须展示提交信息给用户确认，**不得跳过确认直接提交**
- 当变更涉及多个不相关范围时，建议用户拆分为多次提交
- 不主动 `git add -A`，只暂存确认过的文件
- **提交信息默认走文件，不走命令行**：中文/多行消息先写临时文件（UTF-8 无 BOM + LF），用 `git commit -F` 提交，最后删除临时文件
- 临时文件放系统临时目录，不要放进仓库；提交结束后必须清理，不得留下未跟踪文件
- **写消息文件与 `git commit -F` 必须在同一次执行内完成**：临时目录可能按次隔离，跨执行（尤其跨提权）写入的文件后一次读不到
- 多份提交需要 GPG 提权时，先整理好全部提交，再**一次提权、多次使用**，避免反复授权
- 针对当前仓库的文件，不做超出范围的修改
- **Windows 环境使用 PowerShell**（`PS>` 前缀），**Linux/macOS 环境使用 Bash**（`$` 前缀）
