---
name: git-commit
description: '按项目规范创建 Git 提交。当用户说 "commit"、"提交"、"git commit" 或请求创建代码提交时必须触发此 skill。自动分析变更、生成符合规范的提交信息，经用户确认后执行提交。提交前先把全部改动备份到仓库 .git 下的临时区（补丁 + 未跟踪文件副本，校验可还原），提交成功后再删除，防止误删代码无法找回。提交信息一律写入文件（UTF-8 无 BOM + LF）后用 git commit -F 提交，避免 PowerShell / TypeScript(Node) 的命令行编码弄乱中文；消息文件与备份同放 .git 下，跨执行、跨提权都可读。多份提交涉及 GPG 签名时一次提权、多次使用。支持标准提交类型 (feat/fix/docs/style/refactor/test/chore/perf) 和中文主题。'
---

# Git 提交助手

自动分析 Git 变更、生成符合项目规范的提交信息，经确认后执行提交。提交前先把改动备份到仓库 `.git` 下的临时区，提交成功后再删除——**避免误删代码后无法找回**。

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

### 3. 提交前先备份改动（防误删，可还原）

在**任何写操作（`git add` / `git commit` / `git checkout`）之前**，先把当前**全部**改动备份到仓库 `.git` 下的临时区：一份补丁（覆盖已跟踪文件的修改与删除）+ 未跟踪待提交文件的副本。提交成功后再删除；一旦发现误删代码，用备份还原。

**为什么放在 `.git` 下**：它不在工作区，`git status` 与 `git add -A` 都看不到；路径由 git 目录推导，跨执行、跨提权都读得到（系统临时目录在普通执行与提权执行之间可能不是同一个，Node 的 `os.tmpdir()` 在受限环境还会被重映射成无效路径）。

PowerShell（PS 5.1 / 7+ 通用）：

```powershell
# 0) 备份目录：<git-dir>/git-commit-backup/<时间戳>
$gitDir = (git rev-parse --path-format=absolute --git-dir).Trim()
$stamp  = Get-Date -Format 'yyyyMMdd-HHmmss'
$bk     = Join-Path $gitDir "git-commit-backup\$stamp"
$untrackedDir = Join-Path $bk 'untracked'
New-Item -ItemType Directory -Path $untrackedDir -Force | Out-Null

# 1) 已跟踪文件的修改/删除：用 --output 直接写文件，别经过 PowerShell 管道（会写成 UTF-16 / 加 BOM）
$patch = Join-Path $bk 'changes.patch'
git diff HEAD --binary "--output=$patch"

# 2) 未跟踪待提交文件：连同目录结构复制，并逐个校验 SHA256
foreach ($f in (git ls-files --others --exclude-standard)) {
  $dest = Join-Path $untrackedDir $f
  New-Item -ItemType Directory -Path (Split-Path $dest -Parent) -Force | Out-Null
  Copy-Item -LiteralPath $f -Destination $dest -Force
  if ((Get-FileHash -Algorithm SHA256 -LiteralPath $f).Hash -ne (Get-FileHash -Algorithm SHA256 -LiteralPath $dest).Hash) {
    throw "备份校验失败：$f"
  }
}

# 3) 记录基线（当前 HEAD），供还原与比对
Set-Content -Path (Join-Path $bk 'manifest.txt') -Value ((git rev-parse HEAD).Trim())

# 4) 校验补丁可还原：能反向应用说明它与当前工作区一致
if ((Get-Item $patch).Length -gt 0) {
  git apply --check --reverse $patch
  if ($LASTEXITCODE -ne 0) { throw '补丁校验失败，停止提交' }
}
```

Bash：

```bash
gitdir="$(git rev-parse --path-format=absolute --git-dir)"
bk="$gitdir/git-commit-backup/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$bk/untracked"

# 1) 已跟踪文件的修改/删除
git diff HEAD --binary --output="$bk/changes.patch"

# 2) 未跟踪待提交文件 + 逐个校验（macOS 用 shasum -a 256）
git ls-files --others --exclude-standard | while IFS= read -r f; do
  mkdir -p "$bk/untracked/$(dirname "$f")"
  cp -p "$f" "$bk/untracked/$f"
  [ "$(sha256sum "$f" | cut -d' ' -f1)" = "$(sha256sum "$bk/untracked/$f" | cut -d' ' -f1)" ] || { echo "备份校验失败: $f"; exit 1; }
done

# 3) 记录基线
git rev-parse HEAD > "$bk/manifest.txt"

# 4) 校验补丁可还原
if [ -s "$bk/changes.patch" ]; then git apply --check --reverse "$bk/changes.patch" || exit 1; fi
```

- **未跟踪文件必须单独复制**：`git diff HEAD` 只覆盖已跟踪文件，不显示未跟踪文件，漏掉就找不回。
- 备份只收 `--exclude-standard`（遵守 `.gitignore`），不会把 `node_modules` 这类忽略文件拷进来。
- **备份失败（校验不过、复制失败）时终止提交**，先查清原因；改动为空时补丁可能为 0 字节，属正常。
- 仓库**还没有任何提交（unborn HEAD）**时 `git diff HEAD` 会报错：基线改用空树 `4b825dc642cb6eb9a060e54bf8d69288fbee4904`（即 `git diff --binary "--output=$patch" 4b825dc...`），校验与还原同理。

### 4. 提交信息走文件，用 `git commit -F` 提交

**提交信息一律走文件**：命令行参数会经过终端/shell 代码页，PowerShell 与 TypeScript(Node) 都可能把中文弄乱，多行也会被拆坏。统一按下面的流程：

1. 把消息写成**文件**（建议直接放本次备份目录，如 `<备份目录>/message.txt`，提交后随备份一起删除），内容规整为 **UTF-8 无 BOM + LF**；
2. 用 `git commit -F <文件>` 提交；
3. 提交完成后删除消息文件（与备份一并清理）。

规整要求：编码 UTF-8（无 BOM）、换行一律 LF、**不要放进工作区**（避免残留未跟踪文件，放 `.git` 下最稳）、一个提交对应一个文件。

PowerShell（PS 5.1 / 7+ 通用）：

```powershell
# 1) 写提交信息：用 [char]10（LF）连接各行，天然无 CR、无 BOM
# 沿用第 3 步的 $bk；若尚未建备份，先按第 3 步建好目录再写消息
$msgPath = Join-Path $bk 'message.txt'
$lines = @(
  'feat(scope): 中文主题',
  '',
  '- 第一条说明',
  '- 第二条说明'
)
$msg = [string]::Join([char]10, $lines)
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText($msgPath, $msg, $utf8NoBom)

# 2) 暂存并提交（消息文件路径尽量为纯 ASCII；传的是路径，不是内容）
git add <files>
git commit -F $msgPath
```

TypeScript / Node（受限 DSH `run_code` 内同样适用）：

```ts
import { writeFileSync, mkdirSync } from "node:fs";
import { join } from "node:path";

// gitDir 用 `git rev-parse --path-format=absolute --git-dir` 取，别用 os.tmpdir()（受限环境会被重映射）
const gitDir = String.raw`D:\Github\Skills\.git`;
const bk = join(gitDir, "git-commit-backup", "20261011-011927");
mkdirSync(bk, { recursive: true });

const message = ["feat(scope): 中文主题", "", "- 第一条说明", "- 第二条说明"].join("\n");
writeFileSync(join(bk, "message.txt"), message, { encoding: "utf8" }); // UTF-8 无 BOM，\n 即 LF
```

Bash：

```bash
# 1) 写提交信息：heredoc 天然 LF、无 BOM
cat > "$bk/message.txt" <<'EOF'
feat(scope): 中文主题

- 第一条说明
- 第二条说明
EOF

# 2) 暂存并提交
git add <files>
git commit -F "$bk/message.txt"
```

- PowerShell **不要用 `Set-Content` 写消息文件**：PS 5.1 会写成 UTF-16 LE 或带 BOM 的 UTF-8；用 `WriteAllText` + `UTF8Encoding $false` 才稳定。
- Node **不要 `execSync('git commit -m "中文"')`**：消息会经过一层 shell，中文同样可能被弄乱；只写文件、只把文件路径传给 git。
- Bash 用 `<<'EOF'`（引号防止 `$`、反引号被展开），输出天然符合要求。
- 消息文件放 `.git` 下后跨执行/跨提权都能读到；但仍建议「写文件 → `git commit -F`」在同一次执行内完成，便于失败时立即定位。**若因故改用系统临时目录，则必须同一次执行内完成**：临时目录在普通执行与提权执行之间可能不同，后一次会报 `could not read log file ...`。
- 仅当消息确定为**纯 ASCII 且单行**、且用户明确要求时，才可直接 `git commit -m "<subject>"`；含中文或多行一律走文件。

### 5. GPG 签名：一次提权、多次使用

`git config commit.gpgsign` 为 `true` 时每笔提交都要 GPG 签名。受限环境（沙箱）下 gpg.exe 可能因无法创建 msys 信号管道而崩溃（`gpg failed to sign the data` / `couldn't create signal pipe` / `Win32 error 5`），此时按 **gpg-sign** skill 处理：**不得用 `--no-gpg-sign` 绕过签名**，应升级权限重试。

需要签名的提交**不止一笔**时，不要每笔单独提权（每次提权都要用户授权一次）：

1. 先与用户敲定全部提交：确定每笔的 `git add` 范围与完整提交信息（消息文件先不落盘）；
2. 在**同一次提权执行**里逐笔「写消息文件 → `git commit -F`」；消息文件与备份都放 `.git` 下，跨提权也能读到；
3. 所有提交成功后再统一删除消息文件与备份。

```powershell
# 同一次提权执行内完成多笔签名提交：每笔都是「写消息文件 → git commit -F」
$utf8NoBom = New-Object System.Text.UTF8Encoding $false

$m1 = Join-Path $bk 'commit-1.txt'
$s1 = [string]::Join([char]10, @('feat(a): 主题一', '', '- 说明'))
[System.IO.File]::WriteAllText($m1, $s1, $utf8NoBom)
git commit -F $m1

$m2 = Join-Path $bk 'commit-2.txt'
$s2 = [string]::Join([char]10, @('docs(b): 主题二', '', '- 说明'))
[System.IO.File]::WriteAllText($m2, $s2, $utf8NoBom)
git commit -F $m2
```

若中途某笔失败，先查明原因再决定是否补一次提权；任何时候都不要改用 `--no-gpg-sign`（用户明确要求除外）。

### 6. 提交成功后清理备份与消息文件

提交成功后删除整个备份目录（含消息文件）：

```powershell
Remove-Item -Recurse -Force $bk
```

```bash
rm -rf "$bk"
```

- **提交失败或用户放弃提交时不要删备份**：把备份路径和还原方法告诉用户。
- 还原已跟踪改动：`git apply --binary <备份目录>/changes.patch`（在当前 HEAD 上不适用时加 `--3way`，或按补丁内容手工找回）；`manifest.txt` 记录了备份时的 HEAD。
- 还原未跟踪文件：从 `<备份目录>/untracked/` 按原路径拷回。
- 多个提交共用一次备份时，等**全部提交成功**后再统一删除。

### 7. 验证提交结果

```powershell
PS> git status
```

```bash
$ git status
```

## 注意事项

- 提交前必须展示提交信息给用户确认，**不得跳过确认直接提交**
- **提交前必须先备份全部改动**到 `.git` 下的临时区，并确认校验通过；备份是误删代码的唯一回退点
- **备份与消息文件在提交成功后才删除**；提交失败要保留并告知用户还原方法
- 当变更涉及多个不相关范围时，建议用户拆分为多次提交
- 不主动 `git add -A`，只暂存确认过的文件
- **提交信息一律走文件，不走命令行**：中文/多行消息先写文件（UTF-8 无 BOM + LF），用 `git commit -F` 提交，最后删除；纯 ASCII 单行且用户明确要求时才用 `-m`
- 消息文件与备份放仓库 `.git` 下（不污染工作区、跨执行与跨提权可读），不要放进工作区；结束后必须清理，不得留下未跟踪文件
- **写文件必须用可靠的编码方式**：PowerShell 用 `WriteAllText` + `UTF8Encoding $false`（不要 `Set-Content`），Node 用 `writeFileSync(..., "utf8")`（不要 `execSync('... -m "中文"')`），Bash 用 heredoc
- **禁止用 `git checkout -- .` / `git reset --hard` / `git clean -f` 之类命令「重来」**：这正是误删代码的常见来源；确需回退先看备份
- 多份提交需要 GPG 提权时，先整理好全部提交，再**一次提权、多次使用**，避免反复授权
- 针对当前仓库的文件，不做超出范围的修改
- **Windows 环境使用 PowerShell**（`PS>` 前缀），**Linux/macOS 环境使用 Bash**（`$` 前缀）
