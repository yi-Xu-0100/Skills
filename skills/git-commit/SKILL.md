---
name: git-commit
description: '按项目规范创建 Git 提交。当用户说 "commit"、"提交"、"git commit" 或请求创建代码提交时必须触发此 skill。自动分析变更、生成符合规范的提交信息，经用户确认后执行提交。支持标准提交类型 (feat/fix/docs/style/refactor/test/chore/perf) 和中文主题。'
---

# Git 提交助手

自动分析 Git 变更、生成符合项目规范的提交信息，经确认后执行提交。

## 提交格式规范

```text
<type>(<scope>): <subject>
```

- type 使用以下之一：`feat` `fix` `docs` `style` `refactor` `test` `chore` `perf`
- scope 为改动范围（可选），如 `git` `build` `skill-name`
- subject 使用中文，不超过 50 字，命令语气，不加句号
- 一次提交只解决一个问题，保持粒度小且明确

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

将生成的提交信息展示给用户确认。格式如下：

```text
建议提交信息：

  <type>(<scope>): <subject>

改动文件：
  - <file1>
  - <file2>
```

### 3. 用户确认后执行提交

```powershell
PS> git add <files>
PS> git commit -m "<message>"
```

```bash
$ git add <files>
$ git commit -m "<message>"
```

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
