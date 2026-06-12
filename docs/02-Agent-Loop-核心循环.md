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

## 八、系统提示词缓存

系统提示词是 Hermes 省 Token 的关键设计之一：

```python
# 构建一次，缓存整个会话
if agent._cached_system_prompt is None:
    _restore_or_build_system_prompt(agent, system_message, conversation_history)

active_system_prompt = agent._cached_system_prompt
```

- **只在首次调用时构建**，后续复用
- **上下文压缩后重建**（因为消息历史变了）
- **网关场景**：从 SessionDB 恢复（而非重建），保持 Anthropic prefix cache 命中

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

## 十一、流式 vs 非流式

| 模式 | 触发条件 | 特点 |
|---|---|---|
| 非流式 | 默认（CLI 模式） | 等待完整响应 |
| 流式 | `stream_callback` 不为空 | 逐 token 回调，TTS 管线可用 |

## 十二、模型响应处理

### 有 tool_calls

```
响应包含 tool_calls
    → 并行/串行执行工具
    → 将结果追加到 messages
    → 继续循环
```

### 纯文本响应

```
响应不包含 tool_calls
    → 作为 final_response
    → 退出循环
```

### 空响应

```
响应内容为空
    → 重试（最多 _empty_content_retries 次）
    → 仍为空 → 返回上一次有内容的响应
```

### 不完整响应

```
模型返回 finish_reason="length"
    → length_continue_retries
    → 追加 continuation prompt
    → 继续请求
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
| 系统提示词缓存 | 省钱：Anthropic prefix cache 命中 |
| Grace Call | 避免模型在中间状态被截断 |
| 独立迭代预算 | 父子 Agent 解耦，子 Agent 不消耗父预算 |
| Preflight 压缩 | 主动预防而非被动等待 429 |
| 线程作用域中断 | 多线程环境安全 |
| 插件上下文注入到用户消息 | 保护系统提示词缓存前缀 |

## 十五、学习检查点

- [ ] 能否用自己的话描述 Agent Loop 的完整流程？
- [ ] 理解 Grace Call 的设计意图？
- [ ] 为什么系统提示词要缓存而不是每次重建？
- [ ] IterationBudget 为什么需要线程安全？
- [ ] 能解释 Preflight 压缩的触发条件和流程？
- [ ] 理解消息处理管线的 8 个步骤？
