# 模块2：Agent Loop 核心循环

Agent Loop 是 Hermes Agent 的心脏。它管理从用户消息到最终回复的整个生命周期：初始化上下文 → 构建系统提示词 → 进入工具调用循环 → 处理模型响应 → 返回结果。

## 一、核心类关系

| 类/模块 | 文件 | 行数 | 职责 |
|---|---|---|---|
| `AIAgent` | `run_agent.py` | 5,307 | 顶层 Agent 类，60+ 初始化参数 |
| `run_conversation()` | `agent/conversation_loop.py` | 4,965 | 对话主循环的真正实现 |
| `init_agent()` | `agent/agent_init.py` | 1,739 | AIAgent.__init__ 的实际逻辑 |
| `IterationBudget` | `agent/iteration_budget.py` | 62 | 线程安全的迭代预算管理器 |

### 委托模式

`run_agent.py` 中的 `AIAgent` 类大量使用转发模式（Forwarder Pattern）：

```python
# run_agent.py 第 320 行
class AIAgent:
    def __init__(self, ...60+ params...):
        """Forwarder — see agent.agent_init.init_agent."""
        from agent.agent_init import init_agent
        init_agent(self, ...)  # 实际初始化逻辑在 agent_init.py

    def run_conversation(self, user_message, ...):
        """Forwarder — see agent.conversation_loop.run_conversation."""
        from agent.conversation_loop import run_conversation
        return run_conversation(self, user_message, ...)
```

这种设计将 5,000+ 行的核心逻辑拆分到独立模块，同时保持 `AIAgent` 作为统一入口。

## 二、AIAgent 初始化参数

`AIAgent.__init__` 接收约 60 个参数，按功能分组如下：

### 2.1 模型与 Provider

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `base_url` | str | None | API 端点 URL |
| `api_key` | str | None | API 密钥 |
| `provider` | str | None | 提供商名称（openrouter/anthropic 等） |
| `api_mode` | str | None | API 模式（chat_completions/codex_responses 等） |
| `model` | str | "" | 模型名称（空字符串 → 从配置解析） |
| `max_tokens` | int | None | 最大输出 token 数 |
| `reasoning_config` | dict | None | 推理配置（thinking budget 等） |
| `service_tier` | str | None | 服务层级 |
| `fallback_model` | dict | None | 备用模型配置 |

### 2.2 工具与限制

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `max_iterations` | int | 90 | 工具调用最大迭代次数 |
| `enabled_toolsets` | list | None | 启用的工具集 |
| `disabled_toolsets` | list | None | 禁用的工具集 |
| `quiet_mode` | bool | False | 安静模式（减少输出） |

### 2.3 会话与平台

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `session_id` | str | None | 会话 ID |
| `platform` | str | None | 来源平台（cli/telegram/discord 等） |
| `user_id` | str | None | 用户 ID |
| `chat_id` | str | None | 聊天 ID |
| `thread_id` | str | None | 线程 ID（Telegram topics 等） |
| `parent_session_id` | str | None | 父会话 ID（子 Agent 场景） |

### 2.4 回调函数

| 参数 | 说明 |
|---|---|
| `tool_progress_callback` | 工具进度更新 |
| `tool_start_callback` | 工具开始执行 |
| `tool_complete_callback` | 工具执行完成 |
| `thinking_callback` | 推理过程输出 |
| `stream_delta_callback` | 流式文本增量 |
| `clarify_callback` | 用户澄清请求 |
| `step_callback` | 每步回调（网关钩子） |
| `notice_callback` | 通知回调 |

### 2.5 记忆与上下文

| 参数 | 说明 |
|---|---|
| `skip_context_files` | 跳过上下文文件加载 |
| `skip_memory` | 跳过记忆系统 |
| `load_soul_identity` | 加载个性化身份 |
| `credential_pool` | 凭证池（多 key 轮换） |
| `iteration_budget` | 迭代预算（子 Agent 场景） |

## 三、对话主循环详解

`run_conversation()` 在 `agent/conversation_loop.py` 中实现，是整个 Agent 的核心流程。

### 3.1 完整流程图

```
用户消息到达
    │
    ▼
[1] 初始化阶段
    ├── 安装安全 stdio（防 broken pipe）
    ├── 设置 session 上下文
    ├── 恢复/构建系统提示词（缓存机制）
    ├── 连接健康检查（清理死连接）
    └── 重置迭代预算
    │
    ▼
[2] Preflight 上下文压缩
    ├── 估算当前 token 数（含工具 schema）
    ├── 如果超出阈值 → 触发压缩（最多 3 轮）
    └── 压缩后重新估算
    │
    ▼
[3] 插件钩子：pre_llm_call
    └── 插件注入上下文到用户消息（不修改系统提示词）
    │
    ▼
[4] 记忆预取
    └── 外部记忆管理器 prefetch_all()
    │
    ▼
[5] ★ 主循环（while 循环）★
    │
    ├── 检查中断请求
    ├── 消耗迭代预算
    ├── 排空 /steer 指令
    ├── 构建 API 请求消息
    │   ├── 注入记忆上下文到用户消息
    │   ├── 注入插件上下文
    │   ├── 复制 reasoning_content
    │   ├── 清理 finish_reason 等内部字段
    │   ├── 应用 Anthropic prompt caching
    │   └── 清理孤立 tool results
    │
    ├── ★ API 调用（_interruptible_api_call）★
    │   ├── 创建 OpenAI 客户端
    │   ├── 设置超时（动态计算）
    │   ├── 发送请求（流式/非流式）
    │   └── 处理响应
    │
    ├── 处理模型响应
    │   ├── 有 tool_calls → 执行工具 → 回到循环
    │   ├── 纯文本 → 作为最终响应
    │   ├── 空响应 → 重试（最多 N 次）
    │   └── 错误 → 分类处理
    │
    └── 循环退出条件
        ├── 预算耗尽（+ grace call）
        ├── 中断请求
        ├── 最大迭代次数
        └── 获得最终响应
    │
    ▼
[6] 后处理阶段
    ├── 保存会话到数据库
    ├── 触发记忆审查（间隔性）
    ├── 触发技能审查（间隔性）
    ├── 保存轨迹（可选）
    └── 返回结果
```

### 3.2 主循环伪代码

```python
while (api_call_count < max_iterations 
       and iteration_budget.remaining > 0) or _budget_grace_call:
    
    # 检查中断
    if _interrupt_requested:
        break
    
    # 消耗预算
    if _budget_grace_call:
        _budget_grace_call = False  # 仅一次机会
    elif not iteration_budget.consume():
        break  # 预算耗尽
    
    # 构建并发送 API 请求
    api_messages = build_api_messages(messages, system_prompt)
    response = client.chat.completions.create(
        model=model, messages=api_messages, tools=tool_schemas
    )
    
    # 处理响应
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        final_response = response.content
        break
```

## 四、IterationBudget（迭代预算）

`agent/iteration_budget.py` 提供线程安全的迭代计数器：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total    # 父 Agent 默认 90，子 Agent 默认 50
        self._used = 0
        self._lock = threading.Lock()
    
    def consume(self) -> bool:
        """消耗一次迭代，返回是否允许"""
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True
    
    def refund(self) -> None:
        """退还一次迭代（execute_code 场景）"""
        with self._lock:
            if self._used > 0:
                self._used -= 1
    
    @property
    def remaining(self) -> int:
        return max(0, self.max_total - self._used)
```

### 关键设计

- **线程安全**：使用 `threading.Lock` 保护
- **父子独立**：每个 Agent 实例（父或子）有独立预算
- **Grace Call**：预算耗尽后给模型最后一次机会
- **退款机制**：`execute_code` 的迭代可退还（因为它是程序化调用）

## 五、Grace Call 机制

当迭代预算耗尽时，Hermes 不是立即中断，而是给模型一次"宽限调用"：

```python
# 条件中的 or _budget_grace_call
while (... or agent._budget_grace_call):
    if agent._budget_grace_call:
        agent._budget_grace_call = False  # 只能用一次
    elif not agent.iteration_budget.consume():
        break  # 真正耗尽
```

这避免了模型在中间状态被突然截断（如已调用工具但还没生成最终回复）。

## 六、错误处理与 Failover

### 6.1 错误分类器

`agent/error_classifier.py` 将 API 错误分类为不同 `FailoverReason`：

| 错误类型 | 处理策略 |
|---|---|
| 速率限制（429） | 凭证池轮换 → 等待冷却 → fallback 模型 |
| 上下文过长 | 触发上下文压缩 → 重试 |
| 认证失败（401/403） | 刷新凭证 → 重试 |
| 服务器错误（5xx） | 指数退避重试 |
| 空响应 | 重试（最多 N 次） |
| 工具调用格式错误 | 修复参数 → 重试 |

### 6.2 Fallback 模型

当主模型失败时，自动切换到备用模型：

```python
fallback_model = {
    "provider": "openrouter",
    "model": "anthropic/claude-sonnet-4"
}
```

下一轮对话开始时，自动恢复到主模型。

### 6.3 凭证池

支持多 API Key 轮换，避免单 Key 限流：

```python
# _pool_may_recover_from_rate_limit() 逻辑
if pool has available entries > 1:
    轮换到下一个凭证
else:
    直接 fallback 到备用模型
```

## 七、中断处理

### 7.1 中断信号

用户可以在 Agent 循环中发送中断（如发送新消息、点击停止）：

```python
# 主循环检查
if agent._interrupt_requested:
    interrupted = True
    break
```

### 7.2 线程作用域

中断信号绑定到执行线程，避免多线程交叉干扰：

```python
agent._execution_thread_id = threading.current_thread().ident
_ra()._set_interrupt(False, agent._execution_thread_id)
```

## 八、系统提示词三层架构

系统提示词是 Hermes 省 Token 的核心设计。`agent/system_prompt.py` 将系统提示词分为**三个有序层级**，按缓存友好性排序：

### 8.1 三层结构

```
┌─────────────────────────────────────────────────┐
│  Layer 1: stable（稳定层）                        │
│  整个进程生命周期不变 → prefix cache 命中率最高      │
│  ├── SOUL.md 身份（或 DEFAULT_AGENT_IDENTITY）     │
│  ├── Hermes Agent 帮助指引                         │
│  ├── 任务完成/反编造指引（TASK_COMPLETION_GUIDANCE） │
│  ├── 工具行为指引（memory/session_search/skills）     │
│  ├── 工具使用强制规则（TOOL_USE_ENFORCEMENT）         │
│  ├── 模型特定指引（Google/OpenAI/xAI 运营规范）       │
│  ├── Skills 系统提示词                              │
│  ├── 环境探测（Python 版本、PEP 668、uv 等）          │
│  ├── 当前 Profile 提示                             │
│  └── 平台提示（PLATFORM_HINTS）                     │
├─────────────────────────────────────────────────┤
│  Layer 2: context（上下文层）                      │
│  会话内稳定，不同工作目录会变化                        │
│  ├── 调用方传入的 system_message                   │
│  └── 上下文文件（AGENTS.md / .cursorrules /         │
│      CLAUDE.md / SOUL.md）                         │
├─────────────────────────────────────────────────┤
│  Layer 3: volatile（易变层）                       │
│  每次构建可能变化 → 不参与 prefix cache              │
│  ├── 内置记忆快照（MEMORY.md 内容）                  │
│  ├── 用户画像（USER.md 内容）                       │
│  ├── 外部记忆 Provider 块（honcho/mem0 等）          │
│  └── 时间戳行（仅日期精度，非分钟精度）               │
└─────────────────────────────────────────────────┘
```

### 8.2 构建流程

```python
# agent/system_prompt.py
def build_system_prompt_parts(agent, system_message=None) -> dict:
    """返回 {"stable": ..., "context": ..., "volatile": ...}"""
    stable_parts = [...]   # 身份 + 指引 + 环境
    context_parts = [...]  # system_message + 上下文文件
    volatile_parts = [...] # 记忆 + 用户画像 + 时间戳
    return {
        "stable":   "\n\n".join(...),
        "context":  "\n\n".join(...),
        "volatile": "\n\n".join(...),
    }

def build_system_prompt(agent, system_message=None) -> str:
    parts = build_system_prompt_parts(agent, system_message)
    return "\n\n".join(p for p in (parts["stable"], parts["context"], parts["volatile"]) if p)
```

### 8.3 缓存与恢复策略

`_restore_or_build_system_prompt()`（conversation_loop.py L218）实现了**四状态恢复**：

| 存储状态 | 含义 | 处理 |
|---|---|---|
| `missing` | 无会话行（首次对话） | 正常路径，从头构建 |
| `null` | 会话行存在但 `system_prompt` 列为 NULL | 警告（遗留会话），从头构建 |
| `empty` | 存储了空字符串 | 警告（持久化 bug），从头构建 |
| `present` | 有可用提示词 | **直接复用**，保证 prefix cache 命中 |

**关键原则**：系统提示词在整个会话期间**绝不部分重建**。只有上下文压缩时才会完整重建。

### 8.4 时间戳的缓存友好设计

```python
# volatile 层的时间戳仅使用日期精度（非分钟精度）
timestamp_line = f"Conversation started: {now.strftime('%A, %B %d, %Y')}"
# 不用 %H:%M — 分钟精度会在每次重建时改变，导致 prefix cache 失效
```

### 8.5 插件上下文为什么不放进系统提示词

插件钩子 `pre_llm_call` 的输出注入到**用户消息尾部**，而非系统提示词。这是因为系统提示词必须保持字节级稳定才能命中 prefix cache，而插件输出每次可能不同。

## 八B、Anthropic Prompt Caching

`agent/prompt_caching.py` 实现了 Anthropic 原生 prompt caching 策略：

### 策略：`system_and_3`

在消息列表中放置**最多 4 个 `cache_control` 断点**：

```
消息列表:
  [0] system prompt  ← cache_control: ephemeral  (断点 1)
  [1] user msg       ← (历史消息，无断点)
  [2] assistant msg  ← (历史消息，无断点)
  ...
  [N-2] user msg     ← cache_control: ephemeral  (断点 2)
  [N-1] assistant    ← cache_control: ephemeral  (断点 3)
  [N] tool result    ← cache_control: ephemeral  (断点 4)
```

```python
# agent/prompt_caching.py L49
def apply_anthropic_cache_control(api_messages, cache_ttl="5m", native_anthropic=False):
    """在 system prompt + 最后 3 条非 system 消息上注入 cache_control"""
    messages = copy.deepcopy(api_messages)
    marker = {"type": "ephemeral"}  # 或 {"type": "ephemeral", "ttl": "1h"}

    # 断点 1: system prompt
    if messages[0].get("role") == "system":
        _apply_cache_marker(messages[0], marker)
        breakpoints_used = 1

    # 断点 2-4: 最后 3 条非 system 消息
    remaining = 4 - breakpoints_used
    non_sys = [i for i in range(len(messages)) if messages[i]["role"] != "system"]
    for idx in non_sys[-remaining:]:
        _apply_cache_marker(messages[idx], marker)

    return messages
```

### 触发条件

```python
# conversation_loop.py L1058
if agent._use_prompt_caching:  # 自动检测 Anthropic 模型
    api_messages = apply_anthropic_cache_control(api_messages, cache_ttl="5m")
```

### 效果

- 多轮对话中 input token 成本降低约 **75%**
- 需要与系统提示词缓存配合：系统提示词不变 + cache_control 断点 = 连续命中

## 九、Preflight 上下文压缩

在进入主循环前，检查对话历史是否已经超出上下文窗口：

```python
if compression_enabled and len(messages) > threshold:
    preflight_tokens = estimate_request_tokens_rough(messages, ...)
    if compressor.should_compress(preflight_tokens):
        # 最多 3 轮压缩
        for _pass in range(3):
            messages, system_prompt = agent._compress_context(messages, ...)
            if not compressor.should_compress(new_tokens):
                break
```

## 十、消息处理管线

每轮 API 调用前，消息列表经过以下处理：

| 步骤 | 说明 |
|---|---|
| 1. 修复工具参数 | `_sanitize_tool_call_arguments()` |
| 2. 修复消息序列 | `_repair_message_sequence()` |
| 3. 注入记忆上下文 | 外部记忆预取结果 → 用户消息尾部 |
| 4. 注入插件上下文 | pre_llm_call 钩子结果 → 用户消息尾部 |
| 5. 复制推理内容 | reasoning → reasoning_content（多轮推理保持） |
| 6. 清理内部字段 | finish_reason, _thinking_prefill 等 |
| 7. Anthropic 缓存 | 注入 cache_control 断点 |
| 8. 清理孤立结果 | 移除没有对应 tool_call 的 tool result |

## 十一、流式响应处理与 Scrubber

### 11.1 流式 vs 非流式

| 模式 | 触发条件 | 特点 |
|---|---|---|
| 非流式 | 默认（CLI 模式） | 等待完整响应 |
| 流式 | `stream_callback` 不为空 | 逐 token 回调，TTS 管线可用 |

### 11.2 StreamingThinkScrubber（流式推理过滤）

`agent/think_scrubber.py` 是一个**有状态流式过滤器**，解决跨 delta 的 reasoning 标签泄露问题。

**问题场景**：MiniMax-M2.7 等模型将 `<think>` 标签和内容分成不同 delta 发送：

```
delta1 = "<think>"           → 旧的 per-delta 正则直接删除
delta2 = "Let me check..."    → 下游看不到开标签，当作正文 → 推理泄露给用户！
delta3 = "</think>"
```

**解决方案**：`StreamingThinkScrubber` 在上游统一过滤，所有 `stream_delta_callback` 收到的文本已经过清理。

```python
# agent/think_scrubber.py
class StreamingThinkScrubber:
    _OPEN_TAG_NAMES = ("think", "thinking", "reasoning", "thought", "REASONING_SCRATCHPAD")

    def __init__(self):
        self._in_block = False         # 是否在 reasoning 块内
        self._buf = ""                 # 缓存的可能是部分标签的尾部
        self._last_emitted_ended_newline = True  # 上一次输出是否以换行结尾

    def feed(self, text: str) -> str:
        """输入一个 delta，返回已过滤的可见部分（可能为空）"""
        buf = self._buf + text
        # 状态机：在 block 内 → 找关闭标签；在 block 外 → 找开标签
        # 部分标签（如 "<thin"）被缓存到 _buf，等下一个 delta 解决

    def flush(self) -> str:
        """流结束时调用，释放缓存的内容"""

    def reset(self):
        """每轮对话开始时重置，防止上一轮的残留状态污染"""
```

### 11.3 _fire_stream_delta 管线

```python
# run_agent.py L4022
def _fire_stream_delta(self, text: str) -> None:
    # 1. 如果上一轮是工具调用，在首次文本前插入段落分隔
    if self._stream_needs_break and text and text.strip():
        text = "\n\n" + text

    # 2. ★ StreamingThinkScrubber：过滤 reasoning/thinking 块
    think_scrubber = self._stream_think_scrubber
    if think_scrubber is not None:
        text = think_scrubber.feed(text)    # 有状态过滤

    # 3. ★ Context Scrubber：过滤记忆上下文泄露
    scrubber = self._stream_context_scrubber
    if scrubber is not None:
        text = scrubber.feed(text)          # 防止 MEMORY 上下文泄露到 UI

    # 4. 分发给所有回调（CLI 显示 + TTS）
    for cb in (self.stream_delta_callback, self._stream_callback):
        if cb:
            cb(text)
```

### 11.4 双 Scrubber 重置

每轮对话开始时，两个 scrubber 都被重置：

```python
# conversation_loop.py L550-560
scrubber = agent._stream_context_scrubber
if scrubber is not None:
    scrubber.reset()          # 清除记忆上下文过滤状态

think_scrubber = agent._stream_think_scrubber
if think_scrubber is not None:
    think_scrubber.reset()    # 清除 reasoning 过滤状态
```

### 11.5 完整响应的 Think Block 清理

对于非流式完整响应（或流式结束后的 fallback），使用基于正则的 `_strip_think_blocks()`：

```python
# conversation_loop.py 中多处调用
final_response = agent._strip_think_blocks(final_response).strip()
```

## 十二、模型响应处理与工具执行管线

### 12.1 响应分支

```
模型返回响应
    │
    ├── 有 tool_calls → ★ 工具执行管线（见 12.2）→ 回到循环
    ├── 纯文本 → 作为 final_response → 退出循环
    ├── 空响应 → 重试（最多 _empty_content_retries 次）
    └── finish_reason="length" → length continuation（追加 continuation prompt）
```

### 12.2 工具执行管线（完整 6 步）

当模型返回 `tool_calls` 时，经过以下严格管线：

**Step 1: 工具名验证与修复**

```python
# conversation_loop.py L3870-3921
for tc in assistant_message.tool_calls:
    if tc.function.name not in agent.valid_tool_names:
        repaired = agent._repair_tool_call(tc.function.name)  # 模糊匹配修复
        if repaired:
            tc.function.name = repaired

# 仍有无效工具名 → 返回错误给模型自我纠正（最多 3 次）
invalid_tool_calls = [tc.function.name for tc in ... if tc.function.name not in valid_tool_names]
if invalid_tool_calls:
    agent._invalid_tool_retries += 1
    if agent._invalid_tool_retries >= 3:
        return {"partial": True, "error": "invalid tool call"}
    # 返回工具错误消息，让模型下一轮纠正
```

**Step 2: JSON 参数验证**

```python
# conversation_loop.py L3923-4011
for tc in assistant_message.tool_calls:
    args = tc.function.arguments
    if not args or not args.strip():
        tc.function.arguments = "{}"     # 空参数 → 空对象
        continue
    try:
        json.loads(args)
    except json.JSONDecodeError as e:
        # 检查是否为截断（args 不以 } 或 ] 结尾）
        if truncated:
            return {"partial": True, "error": "truncated"}
        # 重试（最多 3 次），超过则注入恢复性 tool result
```

**Step 3: 后置护栏**

```python
# conversation_loop.py L4016-4022
# 限制 delegate_task 并发数
assistant_message.tool_calls = agent._cap_delegate_task_calls(tool_calls)
# 去重（相同工具+相同参数的重复调用）
assistant_message.tool_calls = agent._deduplicate_tool_calls(tool_calls)
```

**Step 4: 分发到执行器**

```python
# run_agent.py L4950-4971
def _execute_tool_calls(self, assistant_message, messages, task_id, api_call_count):
    if not _should_parallelize_tool_batch(tool_calls):
        return self._execute_tool_calls_sequential(...)    # 串行路径
    return self._execute_tool_calls_concurrent(...)         # 并行路径
```

**Step 5: 并行 vs 串行决策**

```python
# agent/tool_dispatch_helpers.py L103
def _should_parallelize_tool_batch(tool_calls) -> bool:
    if len(tool_calls) <= 1:
        return False

    # 绝不并行的工具
    _NEVER_PARALLEL_TOOLS = frozenset({"clarify"})
    if any(name in _NEVER_PARALLEL_TOOLS for name in tool_names):
        return False

    # 安全并行的只读工具
    _PARALLEL_SAFE_TOOLS = frozenset({
        "read_file", "search_files", "session_search",
        "skill_view", "skills_list", "vision_analyze",
        "web_extract", "web_search",
        "ha_get_state", "ha_list_entities", "ha_list_services",
    })

    # 路径作用域工具：不同路径可并行
    _PATH_SCOPED_TOOLS = frozenset({"read_file", "write_file", "patch"})
    # → 检查路径是否重叠，不重叠才并行

    # 其他工具（terminal、execute_code 等）→ 默认串行
    # MCP 工具：检查 server 是否 opt-in parallel
```

**Step 6: 并行执行细节**

```python
# agent/tool_executor.py L561-604
max_workers = min(len(runnable_calls), _MAX_TOOL_WORKERS)  # 最多 8 线程
with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
    for i, tc, name, args in runnable_calls:
        # 传播 ContextVars + 线程本地审批回调到工作线程
        f = executor.submit(propagate_context_to_thread(_run_tool), i, tc, name, args, ...)
        futures.append(f)

    # 带心跳等待（5s 超时轮询），防网关不活跃监控杀进程
    while True:
        done, not_done = concurrent.futures.wait(futures, timeout=5.0)
        if not not_done:
            break
        # 用户中断 → 取消未开始的 future，给已运行的 3s 优雅退出
        if agent._interrupt_requested:
            for f in not_done:
                f.cancel()
            concurrent.futures.wait(not_done, timeout=3.0)
            break
```

### 12.3 handle_function_call 分发器

```python
# model_tools.py L863
def handle_function_call(function_name, function_args, task_id, ...):
    # 1. 类型强制转换（coerce_tool_args: "42" → 42）
    function_args = coerce_tool_args(function_name, function_args)

    # 2. Tool Search bridge 分发
    if is_bridge_tool(function_name):  # tool_search / tool_describe / tool_call
        return handle_bridge_dispatch(...)

    # 3. 正常工具注册表分发
    handler = get_tool_handler(function_name)
    result = handler(**function_args)
    return json.dumps(result)
```

### 12.4 工具结果追加

工具执行完成后，结果按**原始顺序**追加到消息列表（无论并行还是串行执行）：

```python
for i in sorted(completed_indices):
    messages.append({
        "role": "tool",
        "name": tool_name,
        "tool_call_id": tc.id,
        "content": result,
    })
```

### 12.5 空响应处理

```
响应内容为空
    → 重试（最多 _empty_content_retries 次）
    → 仍为空 → 尝试 thinking prefill（注入推理提示）
    → 仍为空 → 返回上一次有内容的响应或 "I apologize..."
```

## 十三、后处理阶段

主循环结束后：

1. **会话持久化**：将完整对话保存到 SessionDB
2. **记忆审查**：每 N 轮触发一次（`_memory_nudge_interval`）
3. **技能审查**：每 N 次工具迭代触发（`_skill_nudge_interval`）
4. **轨迹保存**：可选的 JSONL 格式对话记录
5. **文件变更验证**：检查是否有写入失败但模型声称成功的文件

## 十四、关键设计决策总结

| 设计决策 | 原因 |
|---|---|
| 转发模式（Forwarder） | 将 5K+ 行逻辑拆分到独立模块 |
| 系统提示词三层架构 | stable/context/volatile 分层，最大化 prefix cache 命中 |
| 系统提示词四状态恢复 | missing/null/empty/present 区分，定位缓存失效原因 |
| Prompt Caching `system_and_3` | 4 断点策略，降低 ~75% input token 成本 |
| 时间戳仅日期精度 | 分钟精度会破坏 prefix cache |
| StreamingThinkScrubber | 有状态过滤，防止跨 delta 的 reasoning 泄露 |
| 双 Scrubber 重置 | 每轮清零，防残留状态污染 |
| 工具并行执行（8 线程） | 只读工具 + 不重叠路径工具可安全并行 |
| 工具名自动修复 | 模糊匹配纠正模型幻觉的工具名（最多 3 次） |
| 工具 JSON 参数恢复 | 截断检测 + 注入恢复性 tool result |
| delegate_task 并发限制 | `_cap_delegate_task_calls` 防止子 Agent 过多 |
| Grace Call | 避免模型在中间状态被截断 |
| 独立迭代预算 | 父子 Agent 解耦，子 Agent 不消耗父预算 |
| Preflight 压缩 | 主动预防而非被动等待 429 |
| 线程作用域中断 | 多线程环境安全 |
| 插件上下文注入到用户消息 | 保护系统提示词缓存前缀 |

## 十五、学习检查点

- [ ] 能否用自己的话描述 Agent Loop 的完整流程？
- [ ] 理解 Grace Call 的设计意图？
- [ ] **系统提示词三层架构各包含什么？为什么这样分层？**
- [ ] **`system_and_3` prompt caching 策略的 4 个断点放在哪里？**
- [ ] **为什么时间戳用日期精度而非分钟精度？**
- [ ] **StreamingThinkScrubber 解决了什么问题？状态机如何工作？**
- [ ] **_fire_stream_delta 的双 Scrubber 管线顺序是什么？**
- [ ] **工具执行管线的 6 个步骤分别是什么？**
- [ ] **_should_parallelize_tool_batch 如何判断是否并行？三类工具是什么？**
- [ ] 为什么系统提示词要缓存而不是每次重建？
- [ ] IterationBudget 为什么需要线程安全？
- [ ] 能解释 Preflight 压缩的触发条件和流程？
- [ ] 理解消息处理管线的 8 个步骤？
