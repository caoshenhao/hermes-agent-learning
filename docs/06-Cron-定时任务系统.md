# 模块6：Cron 定时任务系统

Cron 是 Hermes Agent 的**无人值守执行引擎** — 让 Agent 在没有用户在场的情况下按计划自动运行任务。

## 一、架构概览

```
Gateway（主进程）
    ↓ 每 60 秒
tick() — 检查到期任务
    ↓
┌─────────────────────────────────────┐
│  Parallel Pool（线程池）              │
│  ├── 普通任务 → 并行执行             │
│  └── workdir/profile 任务 → 串行执行  │
└─────────────────────────────────────┘
    ↓
run_job() → 创建 AIAgent → 执行 → 投递结果
    ↓
Output → 文件存储 + 消息投递
```

## 二、核心文件

| 文件 | 职责 | 行数 |
|---|---|---|
| `cron/scheduler.py` | 调度核心：tick()、run_job()、投递逻辑 | 2,268 |
| `cron/jobs.py` | 任务存储与管理：CRUD、调度时间计算 | 1,256 |
| `tools/cronjob_tool.py` | `cronjob` 工具实现（create/update/list/remove/run） | ~800 |

## 三、任务存储

### 3.1 文件结构

```
~/.hermes/cron/
├── jobs.json                    # 所有任务定义
├── .tick.lock                   # 文件锁（防止多进程重叠执行）
└── output/
    ├── <job_id>/
    │   ├── 20260601_090000.md   # 每次执行的结果
    │   └── 20260602_090000.md
    └── <another_job_id>/
        └── ...
```

### 3.2 任务定义（jobs.json）

```json
{
  "id": "uuid-string",
  "name": "Daily News Summary",
  "schedule": "0 9 * * *",
  "prompt": "Summarize today's top tech news...",
  "skills": ["web-search", "news-summary"],
  "enabled_toolsets": ["web", "terminal"],
  "model": {"provider": "openrouter", "model": "anthropic/claude-sonnet-4"},
  "workdir": "/path/to/project",
  "profile": "work",
  "deliver": "feishu",
  "no_agent": false,
  "script": "scripts/fetch_data.py",
  "context_from": ["upstream_job_id"],
  "repeat": null,
  "enabled": true,
  "last_run": "2026-06-01T09:00:00Z",
  "next_run": "2026-06-02T09:00:00Z"
}
```

### 3.3 不可变字段

```python
_IMMUTABLE_JOB_FIELDS = frozenset({"id"})
# job_id 被用作文件系统路径组件，创建后不可更改
# 防止路径逃逸攻击（../ 或绝对路径）
```

## 四、调度机制

### 4.1 tick() 循环

```
Gateway 后台线程
    ↓ 每 60 秒触发
tick()
    ├── 获取文件锁（.tick.lock）— 防止多进程重叠
    ├── 加载 jobs.json
    ├── 检查每个任务的 next_run ≤ now
    ├── 对到期任务：
    │   ├── 普通任务 → 提交到 Parallel Pool
    │   └── workdir/profile 任务 → 提交到 Sequential Pool
    └── 释放锁
```

### 4.2 双线程池设计

```python
# 并行池 — 普通任务可以同时运行
_parallel_pool = ThreadPoolExecutor(max_workers=N, thread_name_prefix="cron-parallel")

# 串行池 — workdir/profile 任务必须排队
_sequential_pool = ThreadPoolExecutor(max_workers=1, thread_name_prefix="cron-seq")
```

**为什么需要串行池？** workdir 和 profile 任务会修改进程全局状态（`os.environ["TERMINAL_CWD"]`、环境变量、profile 切换），并行执行会导致状态污染。

### 4.3 调度时间计算

使用 `croniter` 库解析 cron 表达式：

```python
# 支持的 schedule 格式：
# - "30m"         → 每 30 分钟
# - "every 2h"    → 每 2 小时
# - "0 9 * * *"   → 每天 9 点（标准 cron）
# - ISO timestamp → 一次性任务
```

## 五、任务执行流程

### 5.1 run_job() 全流程

```
run_job(job)
    ↓
1. 检查脚本模式（no_agent=True）
    ├── 有脚本 → 执行脚本 → 返回 stdout
    └── 无脚本 → 继续 AI 模式
    ↓
2. 执行前置脚本（prerun_script）
    └── script 字段指定的脚本，stdout 注入 prompt
    ↓
3. 构建提示词 _build_job_prompt()
    ├── 加载 Skills 内容
    ├── 注入 context_from（上游任务输出）
    └── 安全扫描（注入检测）
    ↓
4. 创建 AIAgent
    ├── 读取最新 config.yaml 和 .env
    ├── 配置模型、工具集、reasoning
    └── 设置 cron 会话标记
    ↓
5. 执行 agent.run_conversation()
    ↓
6. 处理结果
    ├── 保存到 output/<job_id>/<timestamp>.md
    ├── 检查 [SILENT] 标记
    └── 投递到目标平台
```

### 5.2 两种执行模式

| 模式 | 配置 | 特点 |
|---|---|---|
| **AI 模式**（默认） | `no_agent=False` | 创建 AIAgent，消耗 Token |
| **脚本模式** | `no_agent=True` | 直接运行脚本，零 Token |

脚本模式的规则：
- 空 stdout → 静默（不发送任何消息）
- 非 空 stdout → 原文发送
- 非零退出码 → 发送错误警报

## 六、安全机制

### 6.1 禁用工具集

Cron 任务**永远禁用**以下工具集：

```python
disabled = ["cronjob", "messaging", "clarify"]
```

- `cronjob` — 防止 cron 任务递归创建更多 cron 任务
- `messaging` — 需要活跃的 gateway 会话
- `clarify` — 需要用户输入（cron 环境无用户在场）

### 6.2 提示注入扫描

```python
class CronPromptInjectionBlocked(Exception):
    """组装好的 prompt（包括加载的 Skill 内容）
    触发注入扫描器时抛出。
    
    填补了 #3968 的漏洞：创建时只扫描用户提供的 prompt，
    运行时加载的 Skill 内容从未被扫描。
    """
```

### 6.3 投递平台验证

```python
_KNOWN_DELIVERY_PLATFORMS = frozenset({
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant", "dingtalk", "feishu",
    "wecom", "wecom_callback", "weixin", "sms", "email", "webhook",
    "bluebubbles", "qqbot", "yuanbao",
})
# 防止通过伪造平台名枚举环境变量
```

### 6.4 路径安全

```python
def _job_output_dir(job_id: str) -> Path:
    """验证 job_id 是安全的文件路径组件。
    拒绝：空值、.、..、含 / 或 \\ 的值、绝对路径。
    """
```

## 七、投递系统

### 7.1 投递目标解析

```
deliver 参数:
    - "origin"    → 创建时的原始聊天
    - "local"     → 仅保存本地文件
    - "feishu"    → 飞书 home channel
    - "telegram"  → Telegram home channel
    - "platform:chat_id:thread_id" → 指定目标
    - "all"       → 所有已连接的频道
```

### 7.2 Home Channel 映射

```python
_HOME_TARGET_ENV_VARS = {
    "matrix": "MATRIX_HOME_ROOM",
    "telegram": "TELEGRAM_HOME_CHANNEL",
    "discord": "DISCORD_HOME_CHANNEL",
    "feishu": "FEISHU_HOME_CHANNEL",
    # ... 更多平台
}
```

### 7.3 静默模式

```python
SILENT_MARKER = "[SILENT]"

# Agent 输出以 [SILENT] 开头时不投递
# 但仍保存到本地文件（审计用途）
```

## 八、Profile 支持

```python
@contextmanager
def _job_profile_context(job_id, profile):
    """临时在指定 Profile 下运行任务。
    
    - 快照当前 os.environ
    - 解析目标 Profile 目录
    - 设置 HERMES_HOME override
    - 运行任务
    - 恢复 os.environ（临时变异隔离）
    
    Profile 任务必须串行执行（环境不是线程安全的）。
    """
```

## 九、Context Chaining（任务链）

```python
# context_from 字段允许任务链式执行
context_from: ["job_id_a", "job_id_b"]

# 运行时注入上游任务的最新输出作为上下文
# 注意：注入的是最近完成的输出，不等待当前 tick 的上游任务
```

## 十、工具集解析优先级

```
1. Per-job enabled_toolsets（创建时设置）
    ↓ 未设置
2. Platform-specific 配置（hermes tools for cron platform）
    ↓ 查找失败
3. None → AIAgent 加载完整默认工具集（兜底）
```

始终排除 `_DEFAULT_OFF_TOOLSETS`（如 moa、homeassistant、rl），防止意外高成本运行。

## 十一、数据流全景

```
Gateway 后台线程
    ↓ 每 60 秒
tick()
    ├── 获取 .tick.lock
    ├── 加载 jobs.json → 筛选到期任务
    ├── Parallel Pool（普通任务） / Sequential Pool（workdir/profile 任务）
    │   ↓
    │   run_job(job)
    │       ├── 检查 no_agent → 脚本模式 or AI 模式
    │       ├── 执行 prerun_script（如有）
    │       ├── _build_job_prompt()
    │       │   ├── 加载 Skills
    │       │   ├── 注入 context_from
    │       │   └── 安全扫描
    │       ├── 加载最新 config.yaml + .env
    │       ├── 创建 AIAgent（cron session）
    │       ├── agent.run_conversation(prompt)
    │       └── 保存结果 → output/<job_id>/<ts>.md
    │   ↓
    ├── 检查 [SILENT] 标记
    ├── 投递到目标平台
    ├── advance_next_run() → 计算下次运行时间
    └── 释放 .tick.lock
```

## 十二、关键设计决策

| 设计决策 | 原因 |
|---|---|
| 60 秒 tick 间隔 | 平衡精度和开销 |
| 双线程池（并行 + 串行） | workdir/profile 任务修改全局状态，需串行 |
| 文件锁（.tick.lock） | 防止多进程重叠执行 |
| 禁用 cronjob/messaging/clarify | 安全底线：防递归、防阻塞、防无用户在场 |
| 运行时注入扫描 | 填补 Skill 加载时的安全漏洞（#3968） |
| 不可变 job_id | 路径安全，防止输出目录逃逸 |
| 每次运行重读 .env/config.yaml | 配置变更无需重启 Gateway |
| [SILENT] 标记 | 允许条件性静默（无新信息时不打扰） |
| ContextVars 代替 os.environ | 并行任务不互相污染投递目标 |
| Profile 串行执行 | os.environ 临时变异不是线程安全的 |

## 十三、学习检查点

- [ ] 能否描述 tick() → run_job() → 投递的完整流程？
- [ ] 理解双线程池的设计理由？
- [ ] 理解为什么禁用 cronjob/messaging/clarify？
- [ ] 理解 [SILENT] 标记的工作方式？
- [ ] 能否解释 Profile 任务的串行执行约束？
- [ ] 理解 context_from 的任务链语义？
