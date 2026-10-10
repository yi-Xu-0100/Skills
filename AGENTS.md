# AGENTS.md — Skills 仓库

本仓库是个人技能（skill）集合：每个技能是 `skills/<kebab-case-name>/` 下的一个 `SKILL.md`（YAML frontmatter + Markdown 指令）。本文件只做导航与仓库约定，详细内容一律见对应文档。

## 何时读取哪个文档

| 场景 | 读取 |
|---|---|
| 了解仓库结构、命名与描述约定、如何新增 skill | [`README.md`](README.md) |
| 新增 skill 的起点模板 | [`template/SKILL.md`](template/SKILL.md) |
| 执行 Git 提交 / 处理提交信息编码 | [`skills/git-commit/SKILL.md`](skills/git-commit/SKILL.md) |
| 处理 LF/CRLF 报错或批量转换换行符 | [`skills/lf-to-crlf/SKILL.md`](skills/lf-to-crlf/SKILL.md) |

## 本仓库约定

- **本仓库唯一的 agent 入口是 `AGENTS.md`**。
- 一切提交走 `git-commit` skill；提交信息用中文，格式 `<type>(<scope>): <subject>`（完整规范见 `README.md`）。
- 写文件一律 **UTF-8 无 BOM + LF**：本仓库 `core.autocrlf=false`、`core.safecrlf=true`。
