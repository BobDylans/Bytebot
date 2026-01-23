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

## 6. 深入解释

### 6.1 Skills 本质上解决的问题
- **可复用性**：把“怎么做”沉淀为标准化工作流，避免每次从零开始。
- **可控性**：通过显式加载与权限限制，让 Agent 的行为更可预测。
- **可迁移性**：技能是纯文本 + 目录结构，项目间可直接拷贝复用。

### 6.2 `SKILL.md` 的作用与写法要点
- **作用**：是“行为契约”，告诉 Agent 何时触发、如何执行、该用哪些工具。
- **写法建议**：
  - 用清晰的触发条件（任务范围/关键词）避免误用。
  - 把流程写成步骤清单，减少歧义。
  - 把“禁止事项”写清楚（例如不要改哪些文件、不要联网）。
  - 如果依赖脚本或模板，明确路径与用法。

### 6.3 加载与优先级机制的意义
- **优先级覆盖**：同名技能按路径优先级覆盖，允许你在项目内“重定义”全局技能。
- **局部生效**：项目级技能更适合约束当前仓库的做法，全局技能更像个人偏好或通用流程。
- **符号链接支持**：可以把共享技能库链接到多个仓库，维护一次到处使用。

### 6.4 典型目录结构示例

```
my-skill/
  SKILL.md
  scripts/
    build-index.js
  references/
    schema.md
  assets/
    template.md
```

- `scripts/`：放自动化脚本，减少手工步骤。
- `references/`：放需要引用的规范/说明。
- `assets/`：放模板或样例输出。

### 6.5 OpenCode 与 Codex 的实践差异
- **Codex**：更偏向“自动匹配 + 自动加载”，技能描述写得越清晰越不容易误触发。
- **OpenCode**：更偏向“显式工具调用”，你可以更精确地控制调用时机。
- **结论**：同一份技能在两个平台都能用，但在 Codex 中要更重视触发条件的严谨性。

### 6.6 权限与安全控制的意义
- **最小权限原则**：只允许技能访问它需要的工具和文件范围。
- **审计与可解释**：权限配置是“行为边界”，出问题时更好追踪。
- **降低误操作风险**：敏感操作（删文件、联网、写配置）应被明确 ask 或 deny。

### 6.7 实战建议（如何写得更“稳”）
- **任务粒度不要过大**：大而全的技能会让触发条件模糊。
- **先写流程，再写模板**：流程稳定后再沉淀模板，减少重复返工。
- **约束工具**：清楚说明能用哪些工具、禁止哪些工具。
- **加自检步骤**：例如“修改后运行 rg 检查或输出变更摘要”。

### 6.8 常见坑与规避
- **坑：触发条件过宽** → 规避：加入明确关键词或路径限制。
- **坑：步骤依赖不写清楚** → 规避：在步骤里标注输入/输出。
- **坑：脚本路径不稳定** → 规避：用相对路径并说明工作目录。
- **坑：技能名与目录不一致**（尤其 OpenCode）→ 规避：严格一致。

### 6.9 一个最小可用的 `SKILL.md` 模板（示例）

```
---
name: example-skill
description: 将 CSV 文件转换为 JSON 并生成摘要。
---

# 目标
- 把指定的 CSV 转换成 JSON
- 输出字段统计摘要

# 触发条件
- 用户请求“CSV 转 JSON”或“字段统计”

# 步骤
1. 使用 `rg --files` 找到 CSV
2. 读取文件并转换
3. 输出 JSON 到 `out/`
4. 生成摘要并打印

# 禁止事项
- 不要联网
- 不要改动原 CSV
```

## 7. 参考来源

- OpenAI Codex Agent Skills 文档：
  - https://developers.openai.com/codex/skills
- OpenCode Agent Skills 文档：
  - https://opencode.ai/docs/skills
  - https://open-code.ai/docs/zh/skills