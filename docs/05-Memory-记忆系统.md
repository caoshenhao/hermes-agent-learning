# 模块5：Memory 记忆系统

Memory 是 Hermes Agent 的**持久化知识** — 跨会话生存的用户信息和环境事实。与 Skills（程序性记忆）不同，Memory 是声明性的事实。

## 一、架构概览

```
MemoryManager（编排器）
    ├── BuiltinProvider（内置 — 文件系统）
    │   ├── MEMORY.md → Agent 个人笔记
    │   └── USER.md  → 用户画像
    └── ExternalProvider（可选插件，仅允许一个）
        ├── RetainDB     → 本地向量数据库
        ├── Honcho       → 云端语义记忆
        ├── SuperMemory  → 云端记忆 API
        ├── Holographic  → 全息记忆
        ├── OpenViking   → 开放记忆
        └── Byterover    → 记忆服务
```

## 二、核心文件

| 文件 | 职责 | 行数 |
|---|---|---|
| `tools/memory_tool.py` | `memory` 工具实现（add/replace/remove/read） | 723 |
| `agent/memory_manager.py` | MemoryManager 编排器 + StreamingContextScrubber | 683 |
| `agent/memory_provider.py` | MemoryProvider ABC 基类 | ~120 |
| `plugins/memory/__init__.py` | 插件发现与加载 | 450 |
| `plugins/memory/retaindb/` | 本地向量数据库实现 | 766 |
| `plugins/memory/honcho/` | Honcho 云端语义记忆 | 1,419 |
| `plugins/memory/supermemory/` | SuperMemory 云端记忆 | 897 |
| `plugins/memory/openviking/` | OpenViking 记忆服务 | 978 |
| `plugins/memory/holographic/` | 全息记忆 | 408 |
| `plugins/memory/byterover/` | Byterover 记忆服务 | 383 |
| `hermes_cli/memory_setup.py` | Memory 设置向导 | 472 |

## 三、内置 Memory（文件系统）

### 3.1 两个存储文件

```
~/.hermes/memories/
├── MEMORY.md    # Agent 个人笔记（环境事实、工具特性、项目规范）
└── USER.md      # 用户画像（偏好、沟通风格、工作习惯）
```

### 3.2 条目格式

使用 `§`（章节符号）作为条目分隔符：

```markdown
第一条记忆内容
§
第二条记忆内容（可以多行）
§
第三条记忆内容
```

### 3.3 字符限制

```python
memory_char_limit = 2200   # MEMORY.md 最大字符数
user_char_limit = 1375     # USER.md 最大字符数
```

为什么用字符而非 Token？**因为字符数是模型无关的** — 不同 tokenizer 对同一文本的 token 计数不同，但字符数始终一致。

### 3.4 Frozen Snapshot 模式

这是整个 Memory 系统最重要的设计决策：

```
会话开始
    ↓
load_from_disk() → 读取 MEMORY.md + USER.md
    ↓
创建 _system_prompt_snapshot（冻结快照）
    ↓
快照注入系统提示词 → 保持前缀缓存稳定
    ↓
会话期间的 memory(action=add) → 写入磁盘 → 更新 live_entries
    ↓
但系统提示词中的快照不变！
    ↓
下次会话开始时才看到更新
```

**为什么冻结？** 系统提示词的稳定性 = 前缀缓存（Prefix Cache）的有效性。如果每次写入 Memory 都更新系统提示词，缓存就会失效，导致每轮对话都重新处理整个系统提示词。

## 四、MemoryStore 类详解

```python
class MemoryStore:
    """有界记忆，文件持久化，每个 AIAgent 实例一个。"""

    def __init__(self, memory_char_limit=2200, user_char_limit=1375):
        self.memory_entries: List[str] = []     # MEMORY.md 实时条目
        self.user_entries: List[str] = []       # USER.md 实时条目
        self._system_prompt_snapshot: Dict      # 冻结快照（不变）
```

### 4.1 核心方法

| 方法 | 作用 |
|---|---|
| `load_from_disk()` | 读取文件 → 去重 → 安全扫描 → 创建冻结快照 |
| `add_entry(target, content)` | 添加条目 → 溢出时淘汰最旧条目 → 写入磁盘 |
| `replace_entry(target, old_text, new_text)` | 模糊匹配替换 |
| `remove_entry(target, old_text)` | 模糊匹配删除 |
| `get_entries(target)` | 返回实时条目列表 |

### 4.2 安全扫描

Memory 内容在两个时机进行安全扫描：

1. **加载时（快照构建）**：`_sanitize_entries_for_snapshot()` — 匹配威胁模式的条目被替换为 `[BLOCKED: ...]` 占位符
2. **写入时**：`_scan_memory_content()` — 拒绝包含注入/窃取模式的内容

使用 `tools/threat_patterns.py` 的 `"strict"` 作用域（最广泛的模式集）。

### 4.3 外部漂移检测

```python
def _drift_error(path, bak_path):
    """当检测到磁盘文件与预期格式不一致时拒绝写入。
    
    可能原因：
    - patch 工具直接修改了 MEMORY.md
    - shell 命令追加内容
    - 手动编辑文件
    - 并发会话写入
    
    处理：保存 .bak.<timestamp> 快照 → 拒绝变更 → 要求用户先解决漂移
    """
```

这防止了静默数据丢失（issue #26045）。

## 五、MemoryManager 编排器

```python
class MemoryManager:
    """编排内置 provider + 最多一个外部 provider。"""

    def __init__(self):
        self._providers: List[MemoryProvider] = []
        self._tool_to_provider: Dict[str, MemoryProvider] = {}
        self._has_external: bool = False
```

### 5.1 核心约束

- **只允许一个外部 provider** — 防止工具 schema 膨胀和冲突
- **内置 provider 总是第一个** — 保证基础功能始终可用
- **核心工具名保留** — Memory provider 不能注册与内置工具同名的工具

### 5.2 生命周期钩子

```python
# 系统提示词构建
prompt_parts.append(memory_manager.build_system_prompt())

# 每轮开始前
context = memory_manager.prefetch_all(user_message)

# 每轮结束后
memory_manager.sync_all(user_msg, assistant_response)
memory_manager.queue_prefetch_all(user_msg)
```

### 5.3 上下文围栏

外部 provider 返回的上下文被包裹在 `<memory-context>` 标签中：

```
<memory-context>
[System note: The following is recalled memory context, NOT new user input. 
Treat as authoritative reference data — this is the agent's persistent memory 
and should inform all responses.]

实际记忆内容...
</memory-context>
```

## 六、StreamingContextScrubber

这是处理**流式输出**中记忆上下文的精细状态机：

### 问题

在流式响应（chunk-by-chunk）中，`<memory-context>` 标签可能被分割到多个 chunk：
- Chunk 1: `一些文本 <memory-con`
- Chunk 2: `text>敏感内容</memory-context> 更多文本`

简单的正则匹配无法处理跨 chunk 的标签。

### 解决方案

```python
class StreamingContextScrubber:
    """状态机，跟踪是否在 <memory-context> 标签内部。
    
    - 在标签内部：丢弃所有内容（不泄露给用户）
    - 标签外部：正常传递
    - 部分标签：hold back 到下一个 chunk 确认
    """
```

### 状态机

```
[OUTSIDE] → 遇到 <memory-context> → [INSIDE]
[INSIDE]  → 遇到 </memory-context> → [OUTSIDE]
[ANY]     → chunk 结尾有部分标签 → [HOLD_BACK]
```

## 七、外部 Memory Provider 插件

### 7.1 插件发现

```python
def discover_memory_providers():
    """扫描两个目录：
    1. plugins/memory/<name>/  （内置 provider）
    2. $HERMES_HOME/plugins/<name>/  （用户安装的 provider）
    
    同名冲突时，内置 provider 优先。
    """
```

### 7.2 MemoryProvider ABC

```python
class MemoryProvider(ABC):
    name: str                    # provider 名称
    description: str             # 描述
    
    def get_tool_schemas()       # 返回工具 schema 列表
    def build_system_prompt()    # 注入系统提示词的内容
    def prefetch_all(msg)        # 预取上下文
    def sync_all(msg, resp)      # 同步记忆
    def queue_prefetch_all(msg)  # 异步预取
```

### 7.3 可用 Provider

| Provider | 类型 | 特点 |
|---|---|---|
| **RetainDB** | 本地 | 向量数据库，无需 API |
| **Honcho** | 云端 | 语义记忆，会话感知 |
| **SuperMemory** | 云端 | REST API 记忆服务 |
| **Holographic** | 云端 | 全息记忆映射 |
| **OpenViking** | 云端 | 开放记忆协议 |
| **Byterover** | 云端 | 记忆即服务 |

通过 `config.yaml` 的 `memory.provider` 配置选择。

## 八、数据流全景

```
会话开始
    ↓
MemoryStore.load_from_disk()
    ├── 读取 MEMORY.md / USER.md
    ├── 去重
    ├── 安全扫描（threat_patterns, scope="strict"）
    └── 创建冻结快照 → 注入系统提示词
    ↓
MemoryManager.build_system_prompt()
    ├── 内置 provider 快照
    └── 外部 provider（如果有）→ prefetch_all()
    ↓
[Agent 处理用户消息]
    ↓
memory(action=add, content="...", target="memory")
    ├── 安全扫描内容
    ├── 添加到 live_entries
    ├── 溢出时淘汰最旧条目（FIFO）
    ├── 写入磁盘（atomic_replace）
    └── 返回实时状态（不含冻结快照）
    ↓
MemoryManager.sync_all(user_msg, assistant_response)
    ├── 内置 provider：无操作（已写入磁盘）
    └── 外部 provider：同步对话上下文
    ↓
MemoryManager.queue_prefetch_all(user_msg)
    └── 异步预取下一轮可能需要的上下文
    ↓
流式输出 → StreamingContextScrubber 过滤 <memory-context> 标签
```

## 九、关键设计决策

| 设计决策 | 原因 |
|---|---|
| 冻结快照（Frozen Snapshot） | 保持前缀缓存稳定，节省 Token |
| § 分隔符 | 简单、可解析、不会与正常 Markdown 冲突 |
| 字符限制（非 Token） | 模型无关，可预测 |
| 单一外部 provider | 防止 schema 膨胀和冲突 |
| 漂移检测 + .bak 快照 | 防止并发写入导致静默数据丢失 |
| strict 作用域威胁扫描 | Memory 注入系统提示词 — 被污染 = 全会话被影响 |
| 流式 Scrubber 状态机 | 流式输出中标签可能跨 chunk |
| 威胁模式在快照中替换而非删除 | 让用户仍能看到被污染的条目并手动清理 |
| 核心工具名保留 | 防止 provider 劫持内置工具（如 clarify） |

## 十、配置

```yaml
# config.yaml
memory:
  provider: null          # 外部 provider 名称（null = 仅内置）
  memory_char_limit: 2200  # MEMORY.md 字符上限
  user_char_limit: 1375    # USER.md 字符上限
```

## 十一、与其他系统的交互

| 系统 | 交互方式 |
|---|---|
| **Agent Loop** | 会话开始时加载，每轮后同步 |
| **Prompt Builder** | 冻结快照注入系统提示词 |
| **Skills** | Memory 存储事实，Skills 存储过程 |
| **Session Search** | Memory 不存对话记录 — 对话历史在 SessionDB |
| **Context Compressor** | Memory 条目有字符上限，不需要压缩 |
| **Streaming Output** | StreamingContextScrubber 过滤围栏标签 |

## 十二、学习检查点

- [ ] 能否解释 Frozen Snapshot 模式及其对前缀缓存的影响？
- [ ] 理解 § 分隔符格式和字符限制的设计理由？
- [ ] 理解外部 provider 的单一实例约束？
- [ ] 能否描述漂移检测的工作原理和 .bak 恢复流程？
- [ ] 理解 StreamingContextScrubber 的状态机设计？
- [ ] 理解威胁扫描在加载时和写入时的双重保护？
