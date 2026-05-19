---
name: lf-to-crlf
description: 将 Git 修改文件中的 LF 换行符转换为 CRLF。当需要将 Git 仓库中文件的换行符从 LF 转换为 CRLF 时使用，适用于跨平台协作中 Windows 开发者处理 Unix 风格换行符的场景。
---

# LF → CRLF 换行符转换

将当前 Git 仓库中所有已修改文件（已暂存、未暂存、未跟踪）的换行符从 LF 转换为 CRLF。仅作用于当前仓库内的变更文件。

## 步骤

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

- 仅作用于**当前 Git 仓库**内的变更文件 — 使用 `git` 命令自动限定到当前工作目录，无需手动指定仓库路径
- 跳过已经为 CRLF 编码的文件，不做重复转换
- 使用 `perl -pi -e` 进行原地转换，无需创建临时文件
- 如果文件列表为空（没有变更文件），提前结束并提示用户
