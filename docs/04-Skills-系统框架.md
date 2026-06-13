# 模块4：Skills 系统

Skills 是 Hermes Agent 的**程序性记忆** — 将成功的任务方法封装为可复用的指令集。与通用记忆（MEMORY.md / USER.md）不同，Skills 是狭义且可操作的。

## 一、核心文件概览

| 文件 | 职责 | 行数 |
|---|---|---|
| `tools/skills_tool.py` | skills_list / skill_view 工具实现（渐进式披露架构） | 1,601 |
| `tools/skill_manager_tool.py` | skill_manage 工具实现（创建/编辑/删除 Skills） | 1,043 |
| `tools/skills_guard.py` | Skill 安全扫描器（恶意代码检测） | 1,086 |
| `tools/skills_hub.py` | Hub 安装/发布/同步（agentskills.io 生态） | 3,748 |
| `tools/skill_usage.py` | 使用追踪与遥测 | 852 |
| `tools/skill_provenance.py` | 来源标记（agent-created vs user-created） | 78 |
| `tools/skills_sync.py` | 技能同步（bundled → home 目录） | 897 |
| `tools/skills_ast_audit.py` | AST 审计工具 | 133 |
| `agent/prompt_builder.py` | 系统提示词中 Skills 索引构建 | 1,553 |
| `agent/curator.py` | 后台 Skill 维护编排器 | 1,843 |
| `agent/background_review.py` | 后台审查 Agent 执行 | 597 |

## 二、Skill 目录结构

```
~/.hermes/skills/                    # 所有 Skills 的单一来源
├── my-skill/
│   ├── SKILL.md                     # 主指令文件（必需）
│   ├── references/                  # 参考文档
│   │   ├── api.md
│   │   └── examples.md
│   ├── templates/                   # 输出模板
│   │   └── template.md
│   ├── scripts/                     # 可执行脚本
│   │   └── validate.py
│   └── assets/                      # 补充文件
└── category-name/                   # 分类目录
    └── another-skill/
        └── SKILL.md
```

### 内置 Skill 分类（19 个目录）

```
skills/
├── apple/                # Apple 生态（Notes、Reminders、FindMy、iMessage）
├── autonomous-ai-agents/ # AI 编码 Agent 委派（Claude Code、Codex、OpenCode）
├── creative/             # 创意内容（ASCII art、Excalidraw、p5.js、设计）
├── data-science/         # 数据科学（Jupyter kernel）
├── dogfood/              # 内部 QA 工具
├── email/                # 邮件（Himalaya CLI）
├── github/               # GitHub 工作流（Auth、PR、Issues、Review）
├── media/                # 媒体（YouTube、GIF、音乐生成）
├── mlops/                # ML 运维（HF Hub、评估、推理）
├── note-taking/          # 笔记（Obsidian）
├── productivity/         # 生产力（Google Workspace、Notion、Maps）
├── red-teaming/          # 红队测试（越狱工具）
├── research/             # 学术研究（arXiv、博客监控、Polymarket）
├── smart-home/           # 智能家居（Hue 灯控）
├── social-media/         # 社交媒体（X/Twitter）
└── software-development/ # 软件开发（调试、TDD、Plan、Spike）
```

## 三、SKILL.md 格式

SKILL.md 使用 YAML frontmatter + Markdown body 的格式：

```markdown
---
name: skill-name              # 必需，最长 64 字符
description: Brief description # 必需，最长 1024 字符
version: 1.0.0                # 可选
platforms: [macos]            # 可选 — 限制平台（macos/linux/windows）
prerequisites:                # 可选
  env_vars: [API_KEY]         # 环境变量需求
  commands: [curl, jq]        # 命令行工具需求
metadata:                     # 可选，任意键值对
  hermes:
    tags: [fine-tuning, llm]
---

# Skill 标题

完整指令内容...

## 触发条件
描述何时应加载此 Skill

## 步骤
1. 第一步
2. 第二步

## 常见陷阱
- 陷阱 1
```

## 四、渐进式披露架构

Hermes 的 Skills 系统采用三层渐进式披露，优化 Token 消耗：

| 层级 | 工具 | 内容 | Token 开销 |
|---|---|---|---|
| Tier 1 | `skills_list` | 仅名称 + 描述（~100 chars/Skill） | 极低 |
| Tier 2 | `skill_view` | 完整 SKILL.md 内容 | 中等 |
| Tier 3 | `skill_view(name, file_path)` | 参考文件/模板/脚本 | 按需 |

### 4.1 skills_list 实现

```python
def skills_list(category=None):
    """列出所有可用 Skills 的元数据
    - 过滤平台不兼容的 Skills
    - 过滤环境不匹配的 Skills
    - 按 category 分组
    - 返回 name + description 摘要
    """
```

### 4.2 skill_view 实现

```python
def skill_view(name, file_path=None):
    """加载 Skill 的完整内容
    - name: Skill 名称
    - file_path: 可选，加载 references/templates/scripts/assets 下的文件
    - 路径安全检查（防止 .. 遍历攻击）
    """
```

## 五、系统提示词中的 Skills 索引

`agent/prompt_builder.py` 的 `build_skills_system_prompt()` 将所有 Skill 的元数据注入系统提示词：

### 5.1 两层缓存

```
Layer 1: 进程内 LRU 缓存（keyed by skills_dir, tools, toolsets）
    ↓ miss
Layer 2: 磁盘快照（.skills_prompt_snapshot.json）
    ↓ miss
Cold path: 完整文件系统扫描 → 写入快照
```

### 5.2 缓存键

```python
cache_key = (
    str(skills_dir.resolve()),        # Skills 目录路径
    tuple(external_dirs),              # 外部目录
    tuple(sorted(available_tools)),    # 当前可用工具
    tuple(sorted(available_toolsets)), # 当前可用工具集
    platform_hint,                     # 平台标识
    tuple(sorted(disabled_skills)),    # 被禁用的 Skills
)
```

### 5.3 Skill 过滤

Skills 在注入提示词前经过多层过滤：

1. **平台过滤**：`skill_matches_platform()` — 检查 `platforms` frontmatter。无 `platforms` 字段 = 全平台兼容。Termux 特殊处理：`linux` 标签的 Skill 在 Termux 上视为兼容
2. **环境过滤**：`skill_matches_environment()` — 检查运行环境（Docker/Kanban/S6）。**注意：这是 offer-time 相关性门控，不是硬兼容门控**——显式加载不受此限制
3. **禁用过滤**：检查 `skills.disabled` 配置列表
4. **条件激活过滤**：`_skill_should_show()` — 检查 Skill 的 `requires` / `fallback_for` 条件规则（详见 §五B）

### 5.4 外部 Skill 目录

`skills.external_dirs` 配置项允许挂载只读的外部 Skill 目录：

- 外部目录中的 Skill 与本地 `~/.hermes/skills/` 一起扫描
- **本地 Skill 优先**：同名冲突时本地版本覆盖外部版本
- 外部目录**不做磁盘快照**（只读且通常较小），每次直接扫描
- 新建 Skill 始终写入本地目录

## 五B、条件激活规则（Conditional Activation）

源码：`agent/skill_utils.py:extract_skill_conditions()` + `agent/prompt_builder.py:_skill_should_show()`

Skill 可通过 frontmatter 的 `metadata.hermes` 声明条件激活规则，控制何时在系统提示词中出现：

```yaml
metadata:
  hermes:
    requires_toolsets: ["browser"]        # 需要 browser 工具集才显示
    requires_tools: ["terminal"]           # 需要 terminal 工具才显示
    fallback_for_toolsets: ["web"]         # 仅当 web 工具集不可用时显示（降级方案）
    fallback_for_tools: ["web_search"]     # 仅当 web_search 工具不可用时显示
```

| 规则类型 | 语义 |
|---|---|
| `requires_toolsets` | **需要**指定工具集**可用**才显示该 Skill |
| `requires_tools` | **需要**指定工具**可用**才显示该 Skill |
| `fallback_for_toolsets` | 仅当指定工具集**不可用**时显示（降级替代方案） |
| `fallback_for_tools` | 仅当指定工具**不可用**时显示 |

当 `available_tools` 和 `available_toolsets` 均为 `None` 时，条件检查被跳过（所有 Skill 显示——向后兼容）。

## 六、skill_manage 工具

`skill_manage` 是 Agent 管理 Skills 的核心工具，支持 7 种操作：

| Action | 说明 | 必需参数 |
|---|---|---|
| `create` | 创建新 Skill | name, content（完整 SKILL.md） |
| `edit` | 全量重写 SKILL.md | name, content |
| `patch` | 定点替换（推荐用于修复） | name, old_string, new_string |
| `delete` | 删除 Skill | name, absorbed_into（可选） |
| `write_file` | 添加/覆盖辅助文件 | name, file_path, file_content |
| `remove_file` | 删除辅助文件 | name, file_path |

### 6.1 安全机制

**Pin 保护**：被 Pin 的 Skill 拒绝删除，但允许 patch/edit：

```python
def _pinned_guard(name):
    """Pin 保护 Skill 不被删除
    - 删除操作被拒绝
    - patch/edit 仍然允许
    - 需要用户手动 hermes curator unpin <name>
    """
```

**安全扫描**（可选，默认关闭）：

```python
def _security_scan_skill(skill_dir):
    """扫描 Skill 目录中的恶意代码
    - skills.guard_agent_created 配置控制
    - 默认关闭（Agent 本身可以通过 terminal() 执行代码）
    - 开启后会对 Agent 创建的 Skill 进行安全检查
    """
```

### 6.2 名称验证

```python
VALID_NAME_RE = re.compile(r'^[a-z0-9][a-z0-9._-]*$')
# 允许：小写字母、数字、连字符、点、下划线
# 最大长度：64 字符
```

### 6.3 absorbed_into 语义

删除 Skill 时可通过 `absorbed_into` 表明意图：

- `absorbed_into="umbrella-skill"` — 合并到另一个 Skill（需已存在）
- `absorbed_into=""` — 纯粹修剪，无转发目标

这帮助 Curator 区分合并与清理。

## 七、Curator（后台维护编排器）

`agent/curator.py` 是后台 Skill 维护系统，在 Agent 空闲时自动运行。

### 7.1 核心职责

- 自动转换 Skill 生命周期状态
- 生成后台审查 Agent 来 pin / archive / consolidate / patch Skills
- 持久化 curator 状态（last_run_at, paused 等）

### 7.2 触发条件

```python
DEFAULT_INTERVAL_HOURS = 24 * 7   # 7 天运行一次
DEFAULT_MIN_IDLE_HOURS = 2        # 至少空闲 2 小时
DEFAULT_STALE_AFTER_DAYS = 30     # 30 天未使用标记为过时
DEFAULT_ARCHIVE_AFTER_DAYS = 90   # 90 天未使用自动归档
```

### 7.3 安全约束

- **只处理 Agent 创建的 Skills**（不影响用户手动创建的）
- **永不自动删除** — 只归档（可恢复）
- **Pin 的 Skills 跳过所有自动操作**
- **使用辅助模型**（不污染主会话的提示缓存）

### 7.4 审查流程

```
maybe_run_curator()
    ↓ 检查空闲时间 + 运行间隔
fork AIAgent（辅助模型）
    ↓ 扫描 agent-created Skills
background_review()
    ↓ 分析使用数据、代码质量
生成建议（pin / archive / consolidate / patch）
    ↓ 通过 skill_manage 执行
更新 .curator_state
```

## 八、Skill Hub 生态

`tools/skills_hub.py`（3,748 行）实现了 agentskills.io 生态集成：

- **安装**：`hermes skills install <identifier>` — 从 Hub 下载 Skill
- **发布**：`hermes skills publish <name>` — 发布到 Hub
- **同步**：`skills_sync.py` — 将 bundled Skills 同步到 ~/.hermes/skills/
- **安全扫描**：安装前自动执行 `skills_guard.py` 扫描

## 九、Prompt 注入检测

Skills 系统内置了提示注入检测（`_INJECTION_PATTERNS`）：

```python
_INJECTION_PATTERNS = [
    "ignore previous instructions",
    "ignore all previous",
    "you are now",
    "disregard your",
    "forget your instructions",
    "new instructions:",
    "system prompt:",
    "<system>",
    "]]>",
]
```

所有 Skill 内容在加载时都会扫描这些模式。

## 十、数据流全景

```
SKILL.md 文件（磁盘）
    ↓
skills_list() → 扫描目录 → 过滤 → 返回元数据
    ↓
skill_view() → 读取文件 → 安全检查 → 返回内容
    ↓
build_skills_system_prompt() → 构建索引 → 缓存 → 注入系统提示词
    ↓
LLM 看到可用 Skills 列表 → 按需调用 skill_view 加载
    ↓
skill_manage() → 创建/编辑/删除 → 触发缓存失效 → 下次构建新索引
    ↓
curator.py → 后台审查 → 维护 Skill 健康
```

## 十一、关键设计决策

| 设计决策 | 原因 |
|---|---|
| 渐进式披露（3 层） | 减少 Token 消耗 — 只加载需要的 Skill 内容 |
| 两层缓存 + 磁盘快照 | 避免每次请求都扫描文件系统 |
| YAML frontmatter | 与 agentskills.io 标准兼容 |
| Pin 保护 | 防止重要 Skill 被意外删除 |
| 只归档不删除 | Curator 的安全底线 — 归档可恢复 |
| 路径遍历检测 | 防止 `../` 等攻击读取任意文件 |
| absorbed_into 语义 | 帮助 Curator 区分合并与清理 |
| 条件激活规则 | 降级 Skill 仅在主工具不可用时出现，避免干扰 |
| 本地优先 + 外部只读 | 支持多源 Skill 挂载而不破坏本地写权限 |
| 环境过滤是 offer-time 门控 | 显式加载可绕过——避免在受限环境中无法手动调用 |

## 十二、学习检查点

- [ ] 能否描述 Skill 从磁盘到系统提示词的完整生命周期？
- [ ] 理解渐进式披露的三层架构？
- [ ] 理解 Curator 的触发条件和安全约束？
- [ ] 能否自己创建一个完整的 Skill（含 frontmatter + 辅助文件）？
- [ ] 理解两层缓存的设计？（LRU + 磁盘快照）
- [ ] 能解释条件激活的四种规则（requires / fallback_for × tools / toolsets）？
- [ ] 理解外部 Skill 目录的本地优先策略？
