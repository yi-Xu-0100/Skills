# Personal Skills

个人技能（skill）集合。每个技能是一组可复用的任务指令，让 Claude / DSH 按既定方式完成某类专门、可重复的工作。

> Agent 入口见 [`AGENTS.md`](AGENTS.md)；本文件承载仓库的详细说明。

## 目录结构

```text
skills/
├── <skill-name>/
│   └── SKILL.md       # 指令 + YAML frontmatter（name、description）
└── ...
template/
└── SKILL.md            # 新 skill 模板
```

技能目录内可按需放置：

- `scripts/` — 可执行脚本
- `references/` — 参考文档
- `assets/` — 其它资源

## 使用方式

当某个技能的 `description` 与当前任务匹配时，宿主（Claude / DSH）会加载它。每个技能位于 `skills/` 下自己的目录中，核心是 `SKILL.md`：YAML frontmatter 声明 `name`、`description`，其后是 Markdown 指令。

## 新增技能

1. 在 `skills/<your-skill-name>/` 下新建目录（名称用 kebab-case）
2. 以 `template/SKILL.md` 为起点
3. 填写 `name` 与 `description`
4. 在 frontmatter 之后编写技能指令
5. 按需添加 `scripts/`、`references/`、`assets/` 子目录

## 约定

- 技能名使用 kebab-case，如 `lf-to-crlf`
- `description` 是主要触发机制：既要说明技能做什么，也要说明什么时候应触发
- 描述语言随目标工作流：面向中文工作流用中文，其余用英文
- 指令采用渐进披露结构（先摘要，后细节）
- 需要可执行脚本时放在技能目录的 `scripts/`，需要参考文档时放在 `references/`
- 仓库文件一律 **UTF-8 无 BOM + LF**（`core.autocrlf=false`、`core.safecrlf=true`）

## 现有技能

| 技能 | 说明 |
|---|---|
| [`lf-to-crlf`](skills/lf-to-crlf/SKILL.md) | 把 Git 修改文件中的 LF 换行符转换为 CRLF；Git 报 `fatal: LF would be replaced by CRLF` 时自动触发 |
| [`git-commit`](skills/git-commit/SKILL.md) | 按本仓库规范创建 Git 提交：提交前备份改动、生成中文提交信息、经确认后执行 |

## Git 规范

> 备份改动、`git commit -F`、GPG 一次提权等可执行细节见 [`skills/git-commit/SKILL.md`](skills/git-commit/SKILL.md)；本节只写仓库级约定。

### 安全策略

- **可直接执行**（无需确认）：`git status`、`git diff`、`git log`、`git branch`、`git show`、`git stash`
- **需确认**：`git add`（涉及大量文件时尤其要确认）、`git commit`（执行前必须征得同意）、`git checkout`（切换分支或恢复文件）
- **严格禁止**（除非用户明确书面批准）：`git push`、`git merge`、`git rebase`、`git reset --hard`、`git clean -f`、`git checkout -- <file>`、`git branch -D`

安全原则：

- 发现非本人造成的意外变更时**立即停止**当前操作并告知用户，不得擅自处理
- **永不**使用 `--no-verify`、`--no-gpg-sign` 等绕过 hook 的参数
- **永不**对 `main`/`master` 执行 force push
- **不** amend commit（用户明确要求除外）；提交失败时创建新提交而非 amend

### 提交信息格式

```text
<type>(<scope>): <subject>

<body>
```

- 语言用中文；一次提交只做一件事，不同问题拆成多个小而聚焦的提交
- `type` 取值：`feat` `fix` `docs` `style` `refactor` `test` `chore` `perf`
- `subject` 最多 50 字符，命令语气（「添加」而非「添加了」），不加句号，说清动机而非罗列改动
- `body` 可选；复杂提交用 `-` 分条说明「做了什么」「为什么」，并补充前置条件、影响范围、回滚方式

示例：

```text
feat(lf-to-crlf): 新增 Git 报错场景的自动触发

当 git add/commit 遇到 "fatal: LF would be replaced by CRLF" 报错时，
自动提取文件路径并完成转换，无需手动扫描整个仓库。
```

```text
fix(scripts): 修复 perl 在 PowerShell 下的路径转义问题
```

### 提交流程

1. 运行 `git status`、`git diff`，确认所有待提交改动
2. 按上述规范编写提交信息，展示给用户确认
3. 用户同意后执行提交（走 `git-commit` skill）
4. 变更存在时，提交前运行项目的 lint / 验证命令
