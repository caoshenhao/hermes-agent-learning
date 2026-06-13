# 模块7：Context Compression 上下文压缩

当对话历史超过模型的上下文窗口限制时，Context Compressor 自动触发，将中间轮次压缩为结构化摘要，保护头尾上下文不被丢失。

## 一、架构概览

```
对话消息列表（messages）
    ↓
should_compress() → 检查是否超过阈值
    ↓
compress(messages)
    ├── Pass 1: 工具输出修剪（确定性，无 LLM 调用）
    │   ├── 去重相同工具输出
    │   ├── 替换旧工具输出为摘要
    │   ├── 裁剪工具调用参数（保持 JSON 有效性）
    │   └── 剥离图片部分（screenshots）
    ├── Pass 2: 切割保护尾（Token 预算）
    ├── Pass 3: LLM 摘要（辅助模型）
    │   ├── 序列化中间轮次
    │   ├── 调用辅助模型生成结构化摘要
    │   └── 合并摘要到尾部第一条消息
    └── 返回压缩后的消息列表
```

## 二、核心文件

| 文件 | 职责 | 行数 |
|---|---|---|
| `agent/context_compressor.py` | 压缩核心逻辑 | 2,078 |
| `hermes_cli/partial_compress.py` | 手动压缩 CLI 命令 | 235 |
| `agent/auxiliary_client.py` | 辅助模型调用客户端 | ~400 |
| `agent/model_metadata.py` | Token 估算与模型上下文长度 | ~500 |

## 三、核心类：ContextCompressor

```python
class ContextCompressor:
    def __init__(self,
        model_context_length: int,     # 模型上下文窗口大小
        threshold_pct: float = 0.80,    # 触发压缩的百分比阈值
        quiet_mode: bool = False,
        auxiliary_model: str = None,    # 用于摘要的辅助模型
    ):
        self.threshold_tokens = int(model_context_length * threshold_pct)
        self.max_summary_tokens = min(
            int(model_context_length * 0.05),
            _SUMMARY_TOKENS_CEILING
        )
```

## 四、触发条件

### 4.1 should_compress()

```python
def should_compress(self, prompt_tokens=None) -> bool:
    """检查是否需要压缩"""
    # 1. Token 数 < 阈值 → 不压缩
    if tokens < self.threshold_tokens:
        return False
    
    # 2. 反抖动保护：最近两次压缩各节省 <10% → 跳过
    if self._ineffective_compression_count >= 2:
        return False
    
    return True
```

### 4.2 延迟压缩（Smart Defer）

```python
def _defer_compression(self, rough_tokens) -> bool:
    """智能延迟压缩。
    
    当粗略估算超过阈值，但上次实际 API 调用证明可以容纳时，
    允许一定的增长空间（5% 或 4096 tokens），避免过早压缩。
    """
```

### 4.3 反抖动保护

```python
_ineffective_compression_count: int = 0

# 每次压缩后检查节省比例
# 如果连续 2 次节省 < 10% → 停止自动压缩
# 建议：/new 开启新会话 或 /compress <topic> 焦点压缩
```

## 五、三阶段压缩流程

### 阶段 1：工具输出修剪（确定性，无 LLM）

这是最廉价的压缩手段，直接裁剪大块 Token：

#### Pass 1a：去重相同工具输出

```python
# 多次读取同一文件 → 保留最新完整副本
# 较旧的副本替换为：
"[Duplicate tool output — same content as a more recent call]"
```

#### Pass 1b：替换旧工具输出为信息性摘要

```python
# 原来：
"terminal ran `npm test` → exit 0, 2000 lines output..."

# 压缩后：
"[terminal] ran `npm test` -> exit 0, 47 lines output"
"[read_file] read config.py from line 1 (3,400 chars)"
```

仅替换 >200 字符的内容，保护尾部消息。

#### Pass 1c：裁剪工具调用参数

```python
# write_file 50KB 的 content 参数 → 截断到 200 chars
# 关键：保持 JSON 有效性！
# 错误做法：直接切片 → 破坏 JSON → 提供商返回 400
# 正确做法：解析 JSON → 截断字符串值 → 重新序列化
```

#### Pass 1d：剥离图片部分

```python
# computer_use 截图（~1MB base64）→ 替换为文本占位符
"[screenshot removed to save context]"
```

### 阶段 2：Token 预算保护尾

```python
# 从尾部向前遍历，累积 Token 数
# 当累积超过 protect_tail_tokens 或达到 protect_tail_count 时停止
# 保护尾 = 最近的消息（包含当前任务的完整上下文）
```

### 阶段 3：LLM 摘要

```python
# 1. 序列化中间轮次为文本
serialized = _serialize_for_summary(turns_to_summarize)

# 2. 计算摘要预算（压缩内容的 20%，有上下限）
budget = _compute_summary_budget(turns_to_summarize)

# 3. 调用辅助模型生成结构化摘要
summary = call_llm(
    model=auxiliary_model,
    system=summarizer_system_prompt,
    user=serialized,
    max_tokens=budget,
)
```

## 六、结构化摘要模板

摘要使用固定的 Markdown 结构：

```markdown
## Active Task
当前正在执行的任务描述

## Goal
用户想要达成的目标

## Constraints & Preferences
约束条件和偏好

## Completed Actions
1. ACTION_1 — 描述 [tool: terminal]
2. ACTION_2 — 描述 [tool: read_file]
...

## Active State
当前运行状态（工作目录、进程、文件等）

## In Progress
正在进行的工作

## Blocked
阻塞项

## Key Decisions
已做出的关键决策

## Resolved Questions
已解决的问题及答案

## Pending User Asks
等待用户确认的请求

## Relevant Files
相关文件列表

## Remaining Work
剩余工作

## Critical Context
关键上下文信息
```

## 七、摘要前缀（Context Compaction Notice）

每个摘要前面都有一段重要的系统说明：

```python
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION — REFERENCE ONLY] "
    "Earlier turns were compacted into the summary below. "
    "This is a handoff from a previous context window — "
    "treat it as background reference, NOT as active instructions. "
    "Do NOT answer questions or fulfill requests mentioned in this summary; "
    "they were already addressed. "
    "Respond ONLY to the latest user message that appears AFTER this summary — "
    "that message is the single source of truth for what to do right now. "
    # ...
)
```

这段前缀至关重要 — 它告诉模型：
1. 摘要只是**参考**，不是**指令**
2. 不要执行摘要中提到的已完成任务
3. **最新用户消息**才是行动的唯一依据
4. 如果最新消息与摘要冲突，**最新消息优先**

## 八、迭代摘要（跨多次压缩）

```
第一次压缩：
  Turn 1-20 → Summary V1

第二次压缩：
  Summary V1 + Turn 21-40 → Summary V2

第三次压缩：
  Summary V2 + Turn 41-60 → Summary V3
```

每次压缩时，前一次的摘要也会被包含在输入中，确保信息不会在多次压缩中丢失。

## 九、降级处理（Fallback）

当 LLM 摘要失败时（API 错误、超时等），使用确定性降级：

```python
_FALLBACK_SUMMARY_MAX_CHARS = 8_000   # 最大 8000 字符
_FALLBACK_TURN_MAX_CHARS = 700        # 每轮最多 700 字符

# 从每轮中提取关键信息（用户意图 + 工具调用 + 结果）
# 保留路径引用
# 不调用 LLM，纯文本操作
```

## 十、Token 估算

```python
# 粗略估算（用于快速判断）
_CHARS_PER_TOKEN = ~4    # 平均每 token 约 4 字符
_IMAGE_TOKEN_ESTIMATE = 1600  # 每张图片约 1600 tokens

# 内容长度计算（考虑多模态）
def _content_length_for_budget(content):
    """文本 → len(text)，图片 → 固定 _IMAGE_CHAR_EQUIVALENT"""
```

## 十B、焦点压缩（Focus Compression）

源码：`agent/context_compressor.py:compress(focus_topic=...)`

用户可以通过 `/compress <topic>` 指定焦点主题，引导压缩器优先保留相关信息：

```python
def compress(self, messages, focus_topic=None):
    """当 focus_topic 非空时，注入焦点指导到摘要提示词"""
    if focus_topic:
        guidance = f'''FOCUS TOPIC: "{focus_topic}"
The user has requested that this compaction PRIORITISE preserving all 
information related to the focus topic above. For content related to 
"{focus_topic}", include full detail — exact values, file paths, command 
outputs, error messages, and decisions. For content NOT related to the 
focus topic, summarise more aggressively. The focus topic sections should 
receive roughly 60-70% of the summary token budget. Even for the focus 
topic, NEVER preserve API keys, tokens, passwords, or credentials.'''
```

### 焦点压缩 vs 全量压缩

| 维度 | 全量压缩 | 焦点压缩 |
|---|---|---|
| 触发方式 | 自动（80% 阈值）或 `/compress` | `/compress <topic>` |
| Token 预算分配 | 均匀分配 | 焦点主题占 60-70% |
| 非焦点内容 | 正常摘要 | 更激进摘要或省略 |
| 适用场景 | 一般超长对话 | 需要保留特定主题细节 |

## 十一、关键设计决策

| 设计决策 | 原因 |
|---|---|
| 三阶段压缩（修剪→保护→摘要） | 优先廉价操作，LLM 只处理必要的部分 |
| Token 预算保护尾（而非固定消息数） | 更精确地保留重要上下文 |
| 结构化摘要模板 | 让模型快速理解上下文，而非解析散文 |
| 摘要前缀 | 防止模型"复活"已完成任务 |
| 反抖动保护 | 避免压缩节省 <10% 的无效循环 |
| JSON 安全的工具参数截断 | 防止破坏工具调用参数导致 400 错误 |
| 图片剥离 | 截图 ~1MB base64 但只有 ~1500 tokens |
| 降级处理 | LLM 失败时仍可保持对话连续性 |
| 迭代摘要（V1→V2→V3） | 多次压缩不丢失信息 |
| 辅助模型（非主模型） | 降低成本 |
| 焦点压缩（`/compress <topic>`） | 用户引导保留特定主题，60-70% 预算倾斜 |
| Smart Defer（延迟压缩） | 粗略高估时避免过早压缩，允许 5% 增长空间 |

## 十二、数据流全景

```
Agent Loop（每轮后检查）
    ↓
should_compress(last_prompt_tokens)
    ├── Token 数 < 80% 阈值 → 跳过
    ├── 连续 2 次无效压缩 → 跳过
    └── 需要压缩 → 继续
    ↓
compress(messages)
    ↓
    ├── _prune_old_tool_results(messages, tail_count, tail_tokens)
    │   ├── Pass 1: 去重相同输出（MD5 hash）
    │   ├── Pass 2: 替换旧输出为摘要
    │   └── Pass 3: 裁剪工具参数（JSON 安全）
    ↓
    ├── 切割：保护尾（Token 预算）vs 中间轮次
    ↓
    ├── _serialize_for_summary(middle_turns)
    │   └── 限制每条消息 _CONTENT_MAX chars
    ↓
    ├── _compute_summary_budget(turns)
    │   └── max(MIN, min(content * 20%, CEILING))
    ↓
    ├── call_llm(auxiliary_model, system, serialized, max_tokens=budget)
    │   ├── 成功 → 结构化摘要
    │   └── 失败 → 确定性降级（每轮 700 chars，总 8K）
    ↓
    └── 合并摘要到尾部第一条用户消息
        └── 返回压缩后的 messages
```

## 十三、学习检查点

- [ ] 能否描述三阶段压缩的流程？
- [ ] 理解 Token 预算保护尾 vs 固定消息数的优劣？
- [ ] 理解反抖动保护的工作原理？
- [ ] 理解摘要前缀的设计意图？
- [ ] 能否解释工具参数截断为什么要保持 JSON 有效性？
- [ ] 理解迭代摘要的信息保留机制？
- [ ] 理解降级处理的约束（8K 总，700/轮）？
