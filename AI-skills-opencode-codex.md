# AI Skills 概念与应用（OpenCode + Codex）

> 目的：梳理“AI Agent Skills”在 OpenCode 与 OpenAI Codex 中的核心概念、文件结构与应用方式，作为可复用的速查文档。

## 1. 核心概念（共同点）

- **技能（Skill）**：把可复用的工作流/能力写进一个 `SKILL.md`，并可配套脚本、引用资料和资源，以便 Agent 可靠执行特定任务。
- **按需加载**：Agent 只在需要时读取技能全文（降低上下文负担），并通过“工具/命令”显式或隐式触发。
- **开放标准**：Codex 的技能基于 open agent skills 标准；OpenCode 也采用相同的 `SKILL.md` 组织方式。

## 2. Codex（OpenAI）里的 Skills

### 2.1 定义与组成
- `SKILL.md` 为必需文件；可选 `scripts/`、`references/`、`assets/` 目录。

### 2.2 触发方式
- **显式调用**：在对话里通过 `/skills` 或 `$` 选择技能（Web/iOS 暂不支持显式调用，但仓库内技能仍可请求使用）。
- **隐式调用**：当任务与技能描述匹配时，Codex 自动选择并读取技能全文。

### 2.3 存放位置与优先级（高 → 低）
- `$CWD/.codex/skills`
- `$CWD/../.codex/skills`
- `$REPO_ROOT/.codex/skills`
- `$CODEX_HOME/skills`（macOS/Linux 默认为 `~/.codex/skills`）
- `/etc/codex/skills`
- `SYSTEM`（Codex 内置）

> 同名技能按优先级覆盖；支持符号链接。

### 2.4 启用 / 禁用
- `~/.codex/config.toml` 通过 `[[skills.config]]` 单独禁用技能（实验特性）。

### 2.5 创建与安装
- 使用内置 `$skill-creator` 引导生成。
- 手动创建：`SKILL.md` 须含 `name` 与 `description` 的 YAML frontmatter。
- 使用 `$skill-installer` 从官方/社区仓库安装更多技能。

## 3. OpenCode 里的 Skills

### 3.1 存放位置
- 项目级：`.opencode/skills/<name>/SKILL.md`
- 全局：`~/.config/opencode/skills/<name>/SKILL.md`
- Claude 兼容：`.claude/skills/<name>/SKILL.md`
- 全局 Claude 兼容：`~/.claude/skills/<name>/SKILL.md`

### 3.2 发现与加载
- 在项目内向上遍历至 git 根目录，加载路径上的 `.opencode/skills/*/SKILL.md` 与 `.claude/skills/*/SKILL.md`。
- 同时加载全局目录下的技能。
- 通过 `skill` 工具列表查看技能，并用 `skill({ name: "..." })` 加载。

### 3.3 Frontmatter 规则
- 必填：`name`、`description`
- 可选：`license`、`compatibility`、`metadata`（字符串到字符串）
- `name` 规则：1–64 字符、小写字母数字、单连字符分隔，且与目录名一致
- `description` 长度：1–1024 字符

### 3.4 权限与控制
- `opencode.json` 中可用通配符配置 `allow/deny/ask` 权限。
- 可对特定 agent 覆盖权限；也可禁用 `skill` 工具。

## 4. 对比速览

| 维度 | Codex | OpenCode |
| --- | --- | --- |
| 核心文件 | `SKILL.md` + 可选脚本/资源 | `SKILL.md` + 可选内容 |
| 触发方式 | 显式 `/skills`/`$` 或隐式匹配 | `skill` 工具按需加载 |
| 主要位置 | `.codex/skills`、`$CODEX_HOME/skills`、`/etc/codex/skills` | `.opencode/skills`、`~/.config/opencode/skills`、`.claude/skills` |
| 覆盖/优先级 | 位置优先级覆盖 | 以发现路径为准，按位置加载 |
| 权限控制 | `config.toml` 可禁用 | `opencode.json` allow/deny/ask + 可禁用工具 |

## 5. 应用方式（最小流程）

1. 新建技能目录：`<skill-name>/SKILL.md`。
2. 写 YAML frontmatter（至少 `name` + `description`）。
3. 放入合适位置（项目级或全局）。
4. 通过工具显式调用，或让 Agent 自动匹配。
5. 需要时配置权限/启用状态。

## 6. 参考来源

- OpenAI Codex Agent Skills 文档：
  - https://developers.openai.com/codex/skills
- OpenCode Agent Skills 文档：
  - https://opencode.ai/docs/skills
  - https://open-code.ai/docs/zh/skills
