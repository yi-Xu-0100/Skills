---
name: git-commit
description: '按项目规范创建 Git 提交。当用户说 "commit"、"提交"、"git commit" 或请求创建代码提交时必须触发此 skill。自动分析变更、生成符合规范的提交信息，经用户确认后执行提交。支持标准提交类型 (feat/fix/docs/style/refactor/test/chore/perf) 和中文主题。'
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
- 提交命令改用 git commit -m 双参数支持多行消息
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

### 3. 用户确认后执行提交

使用多行提交信息（标题 + 正文）：

```powershell
PS> git add <files>
PS> git commit -m "<subject>" -m "<body>"
```

```bash
$ git add <files>
$ git commit -m "<subject>" -m "<body>"
```

每个 `-m` 对应一个段落：第一个是标题，第二个是正文。

正文需要多行时的换行写法（普通双引号中 `\n` 无效）：

- **Bash**：用 `$'...'` 语法，如 `$'第一行\n第二行'`
- **PowerShell**：用 `` `n ``，如 `` "第一行`n第二行" ``

### 4. 验证提交结果

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
- 针对当前仓库的文件，不做超出范围的修改
- **Windows 环境使用 PowerShell**（`PS>` 前缀），**Linux/macOS 环境使用 Bash**（`$` 前缀）
