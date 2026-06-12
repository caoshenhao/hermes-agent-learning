# 模块8：LLM 调用层

LLM 调用层是 Hermes Agent 与各种大语言模型提供商交互的桥梁，包括主 Agent Loop 的调用和辅助任务（压缩、视觉、搜索等）的调用。

## 一、架构概览

```
AIAgent（主循环）
    ↓
run_agent.py → OpenAI SDK → 提供商 API
    ↓
Auxiliary Tasks（辅助任务）
    ├── 上下文压缩 → call_llm(task="compression")
    ├── 视觉分析   → call_llm(task="vision")
    ├── 网页提取   → call_llm(task="web_extract")
    ├── 会话搜索   → call_llm(task="session_search")
    └── 标题生成   → call_llm(task="title_generation")
    ↓
auxiliary_client.py → 统一调用路由
```

## 二、核心文件

| 文件 | 职责 | 行数 |
|---|---|---|
| `agent/auxiliary_client.py` | 辅助模型调用路由 + 客户端管理 | 5,928 |
| `agent/model_metadata.py` | 模型元数据、上下文长度、Token 估算 | 1,925 |
| `agent/redact.py` | API 密钥脱敏 | 496 |
| `model_tools.py` | 主 Agent 的工具编排 + handle_function_call | 1,216 |
| `hermes_cli/runtime_provider.py` | 运行时 Provider 解析 | 1,694 |
| `hermes_cli/auth.py` | 认证管理（API Key、OAuth） | 7,706 |

## 三、Provider 生态

### 3.1 支持的 Provider 前缀

```python
_PROVIDER_PREFIXES = frozenset({
    # 主流提供商
    "openrouter", "nous", "openai-codex", "anthropic",
    "gemini", "ollama-cloud", "zai", "deepseek",
    
    # 国产模型
    "kimi-coding", "kimi-coding-cn", "stepfun",
    "minimax", "minimax-oauth", "minimax-cn",
    "alibaba", "xiaomi", "qwen-oauth", "tencent-tokenhub",
    
    # 辅助
    "copilot", "copilot-acp", "opencode-zen", "opencode-go",
    "kilocode", "arcee", "gmi", "novita",
    "custom", "local",
    
    # 常见别名
    "google", "claude", "glm", "z.ai", "zhipu",
    "github", "github-copilot", "github-models",
    "kimi", "moonshot", "deep-seek",
})
```

### 3.2 自动检测优先级（文本任务）

```
1. 用户的主 Provider + 主模型（无论什么类型）
2. OpenRouter（OPENROUTER_API_KEY）
3. Nous Portal（~/.hermes/auth.json）
4. Custom endpoint（config.yaml model.base_url + OPENAI_API_KEY）
5. Native Anthropic
6. 直接 API-key 提供商（z.ai/GLM、Kimi、MiniMax 等）
7. None（报错）
```

### 3.3 自动检测优先级（视觉任务）

```
1. 主 Provider（如果支持视觉）
2. OpenRouter
3. Nous Portal
4. Native Anthropic
5. Custom endpoint（本地视觉模型：Qwen-VL、LLaVA 等）
6. None（报错）
```

## 四、call_llm() 核心函数

```python
def call_llm(
    task: str = None,          # "compression", "vision", "web_extract", ...
    *,
    provider: str = None,       # 显式 Provider 覆盖
    model: str = None,          # 显式模型覆盖
    base_url: str = None,       # 显式 API 端点
    api_key: str = None,        # 显式 API 密钥
    messages: list,             # 聊天消息列表
    temperature: float = None,  # 采样温度
    max_tokens: int = None,     # 最大输出 Token
    tools: list = None,         # 工具定义
    timeout: float = None,      # 超时
    extra_body: dict = None,    # 额外请求体字段
) -> Any:
    """统一的同步 LLM 调用接口"""
```

### 4.1 解析流程

```
call_llm(task, messages, ...)
    ↓
_resolve_task_provider_model(task, provider, model, ...)
    ├── 显式参数 → 直接使用
    ├── config.yaml → auxiliary.<task>.provider/model
    └── "auto" → 完整自动检测链
    ↓
_get_cached_client(provider, model, ...)
    ├── 缓存命中 → 返回
    └── 缓存未命中 → resolve_provider_client() → 缓存
    ↓
_build_call_kwargs(provider, model, messages, ...)
    ↓
client.chat.completions.create(**kwargs)
    ↓
异常处理链（多层重试）
```

### 4.2 多层异常处理

```python
try:
    return client.chat.completions.create(**kwargs)
except Exception as first_err:
    # Layer 1: 不支持 temperature → 重试不带 temperature
    if _is_unsupported_temperature_error(first_err):
        kwargs.pop("temperature", None)
        retry...
    
    # Layer 2: 不支持 max_tokens → 重试不带 max_tokens
    if "max_tokens" in err_str:
        kwargs.pop("max_tokens", None)
        retry...
    
    # Layer 3: 旧模型自我修复（Nous Portal）
    if _is_model_not_found_error(first_err) and is_nous:
        healed_model = _refresh_nous_recommended_model(...)
        retry with healed_model...
    
    # Layer 4: 支付/额度耗尽 → 切换到下一个 Provider
    if _is_payment_error(first_err):
        next_provider = _next_in_auto_chain(...)
        retry with next_provider...
    
    # Layer 5: 认证错误 → 刷新 token 重试
    if _is_auth_error(first_err):
        refreshed = _refresh_oauth_token(...)
        retry...
    
    # Layer 6: 连接错误 → 标记客户端中毒 + 重试
    if _is_connection_error(first_err):
        _poison_client(client)
        retry...
```

## 五、客户端缓存

### 5.1 缓存设计

```python
_client_cache: Dict[str, Tuple[client, default_model, event_loop]] = {}
_CLIENT_CACHE_MAX_SIZE = 50  # 最大缓存条目

# 缓存键 = (provider, async_mode, base_url, api_key_hash, api_mode, ...)
```

### 5.2 异步客户端的 Loop 绑定问题

```python
# 问题：AsyncOpenAI 绑定到创建时的事件循环
# 在不同的循环上使用 → 死锁或 RuntimeError

# 解决方案：
# 1. 缓存时记录绑定的 event_loop
# 2. 每次使用时验证 loop 仍然是当前且打开的
# 3. 不匹配 → 关闭旧客户端 → 创建新客户端
```

### 5.3 客户端中毒处理

```python
# 连接错误时标记客户端为"中毒"
# 中毒的客户端不再被缓存命中
# 防止重用已关闭的 httpx transport
```

## 六、Anthropic 兼容层

### 6.1 API 模式检测

```python
def _endpoint_speaks_anthropic_messages(base_url: str) -> bool:
    """检测端点是否使用 Anthropic Messages 协议
    - URL 以 /anthropic 结尾
    - api.anthropic.com
    - api.kimi.com/coding
    """
```

### 6.2 AnthropicAdapter

```python
class _AnthropicCompletionsAdapter:
    """将 Anthropic Messages API 包装为 OpenAI 兼容接口
    
    处理：
    - 消息格式转换
    - 工具调用格式转换
    - usage 格式转换
    - temperature/top_p 参数过滤（Opus 4.7+ 拒绝非默认采样参数）
    """
```

### 6.3 支持的 API 模式

| 模式 | 说明 |
|---|---|
| `chat_completions` | OpenAI 标准格式 |
| `anthropic_messages` | Anthropic 原生格式 |
| `codex_responses` | OpenAI Codex Responses API |

## 七、Token 估算

### 7.1 粗略估算

```python
_CHARS_PER_TOKEN = 4       # 平均每 token 约 4 字符
_IMAGE_TOKEN_ESTIMATE = 1600  # 每张图片约 1600 tokens

def estimate_messages_tokens_rough(messages, tools=None):
    """快速估算，不依赖 tokenizer。
    故意高估，以在提供商拒绝前触发压缩。
    """
```

### 7.2 模型上下文长度

```python
# 内置已知模型的上下文长度
MODEL_CONTEXT_LENGTHS = {
    "gpt-4o": 128000,
    "claude-sonnet-4": 200000,
    "deepseek-chat": 64000,
    # ... 更多模型
}

def get_model_context_length(model, provider=None):
    """获取模型的上下文窗口大小。
    优先级：内置表 > OpenRouter 模型列表 > 默认值（128K）
    """
```

## 八、凭证管理

### 8.1 API Key 池

```python
# 支持多 API Key 轮换
# 当一个 Key 额度耗尽 → 自动切换到下一个
# Key 状态：active → exhausted → cooling → active
```

### 8.2 OAuth 支持

```python
# 支持的 OAuth 流：
# - Codex OAuth（ChatGPT 账号认证）
# - MiniMax OAuth
# - Qwen OAuth
# - Kimi Coding
```

## 九、Per-Task 配置

```yaml
# config.yaml
auxiliary:
  compression:
    provider: openrouter
    model: anthropic/claude-sonnet-4
    timeout: 120
  vision:
    provider: anthropic
    model: claude-sonnet-4
  web_extract:
    provider: auto
  session_search:
    provider: auto
  title_generation:
    provider: auto
```

每个辅助任务可以独立配置 provider、model 和 timeout。

## 十、数据流全景

```
call_llm(task="compression", messages=[...])
    ↓
_resolve_task_provider_model("compression")
    ├── auxiliary.compression.provider = "openrouter"
    ├── auxiliary.compression.model = "anthropic/claude-sonnet-4"
    └── auxiliary.compression.timeout = 120
    ↓
_get_cached_client("openrouter", "anthropic/claude-sonnet-4")
    ├── 缓存命中 → 返回 (client, model)
    └── 缓存未命中 →
        ├── resolve_provider_client("openrouter", ...)
        │   ├── OPENROUTER_API_KEY → 创建 OpenAI client
        │   └── base_url = "https://openrouter.ai/api/v1"
        └── 缓存结果 → 返回 (client, model)
    ↓
_build_call_kwargs(provider, model, messages, ...)
    ├── model → "anthropic/claude-sonnet-4"
    ├── messages → [...]
    ├── temperature → None
    ├── max_tokens → budget
    └── timeout → 120
    ↓
client.chat.completions.create(**kwargs)
    ↓
异常处理链
    ├── temperature 不支持 → 去掉重试
    ├── max_tokens 不支持 → 去掉重试
    ├── 模型不存在 → 刷新 Nous 推荐
    ├── 额度耗尽 → 切换下一个 Provider
    ├── 认证失败 → 刷新 OAuth
    └── 连接错误 → 中毒客户端 + 重试
    ↓
_validate_llm_response(response, task)
    ├── 检查空响应
    ├── 检查 content filter
    └── 返回 response
```

## 十一、关键设计决策

| 设计决策 | 原因 |
|---|---|
| 统一 call_llm 接口 | 所有辅助任务共用一个入口，避免重复的 fallback 逻辑 |
| 多层异常处理 | 不同的提供商有不同的限制，需要逐个处理 |
| 客户端缓存（LRU 50 条） | 避免每次调用都创建新连接 |
| 异步 Loop 绑定验证 | 防止跨 loop 使用导致死锁 |
| 客户端中毒标记 | 防止重用已关闭的 transport |
| OpenAI SDK 延迟导入 | 避免 240ms 的冷启动开销 |
| Provider 前缀剥离 | 支持 "openrouter/claude-sonnet-4" 格式 |
| 粗略估算故意高估 | 宁可早压缩，不要被提供商拒绝 |
| ZAI 专用错误处理 | ZAI 的错误码格式不同于标准 |

## 十二、学习检查点

- [ ] 能否描述 call_llm 的完整解析流程？
- [ ] 理解多层异常处理的设计理由？
- [ ] 理解客户端缓存的 Loop 绑定问题？
- [ ] 能否列出自动检测优先级链？
- [ ] 理解 Anthropic 兼容层的设计？
- [ ] 理解 Per-Task 配置的灵活性？
