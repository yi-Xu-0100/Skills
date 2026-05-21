---
name: lf-to-crlf
description: '将 Git 修改文件中的 LF 换行符转换为 CRLF。当 Git 报错 "fatal: LF would be replaced by CRLF" 或类似换行符相关错误导致 add/commit/checkout 等操作失败时必须触发此 skill。也适用于手动将 Git 仓库中文件的换行符从 LF 批量转换为 CRLF 的场景（跨平台协作中 Windows 开发者处理 Unix 风格换行符）。'
---

# LF → CRLF 换行符转换

将 Git 仓库中文件的换行符从 LF 转换为 CRLF，解决 Windows 环境下 Git 因 `core.autocrlf` 配置导致的换行符冲突问题。

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

对提取出的文件执行换行符转换：

```bash
perl -pi -e 's/\r?\n/\r\n/' <lf-files>
```

如果文件数量较多，可以使用循环逐个转换或 `xargs`：

```bash
echo "<file1> <file2> ..." | xargs perl -pi -e 's/\r?\n/\r\n/'
```

### 3. 验证并重试 Git 操作

转换完成后，使用 `file` 命令确认文件已变为 CRLF 编码，然后重新执行之前被阻止的 Git 操作（`git add` 等）。

---

## 变更文件批量转换流程

### 1. 获取变更文件列表

获取已跟踪且已修改的文件（包括已暂存和未暂存）：

```bash
git diff --name-only HEAD
```

获取未跟踪的文件（排除 .gitignore 中的文件）：

```bash
git ls-files --others --exclude-standard
```

合并这两个列表，去重，作为后续步骤的输入。

### 2. 检测 LF 编码文件

对文件列表中的每个文件，使用 `file` 命令检查其换行符编码。`file` 命令输出中包含 "CRLF" 的表示已经是 CRLF 格式，不包含的则为 LF 格式。

```bash
for f in <file-list>; do
  file "$f" 2>/dev/null
done
```

只保留检测为 LF 格式的文件进入下一步。

### 3. 转换为 CRLF

对检测出的 LF 文件执行换行符转换：

```bash
perl -pi -e 's/\r?\n/\r\n/' <lf-files>
```

此命令兼容 Git Bash for Windows 环境。

### 4. 验证转换结果

确认所有被转换的文件现在显示为 CRLF 编码：

```bash
for f in <converted-files>; do
  file "$f"
done
```

打印转换结果摘要，列出已处理文件及其编码状态。

## 注意事项

- 仅作用于**当前 Git 仓库**内的文件 — 使用 `git` 命令自动限定到当前工作目录
- 跳过已经为 CRLF 编码的文件，不做重复转换
- 使用 `perl -pi -e` 进行原地转换，无需创建临时文件
- 如果文件列表为空，提前结束并提示用户
