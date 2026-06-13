# 模块9：Gateway 网关系统

Gateway 是 Hermes Agent 的**消息路由中枢** — 连接各种即时通讯平台（Telegram、Discord、飞书、微信等）与 Agent Core 的桥梁。

## 一、架构概览

```
用户消息 → 平台 Adapter → GatewayRunner → AIAgent → 回复 → 平台 Adapter → 用户
                ↑                                    ↓
            事件循环                            Agent 缓存
            Cron tick                           会话管理
            线程池                              消息队列
```

## 二、核心文件

| 文件 | 职责 | 行数 |
|---|---|---|
| `gateway/run.py` | GatewayRunner 主类 — 生命周期、消息路由、Agent 管理 | 20,091 |
| `gateway/session.py` | 会话管理 — 上下文跟踪、持久化、重置策略 | 1,400 |
| `gateway/config.py` | 网关配置 — Platform 枚举、SessionResetPolicy | ~500 |
| `gateway/base.py` | BaseAdapter 抽象基类 | ~800 |

## 三、支持的平台（20+）

```
gateway/platforms/
├── telegram.py          # Telegram Bot API
├── discord.py           # Discord Bot
├── slack.py             # Slack Bot
├── feishu.py            # 飞书机器人
├── feishu_comment.py    # 飞书评论事件
├── wecom.py             # 企业微信
├── wecom_callback.py    # 企业微信回调
├── weixin.py            # 微信公众号
├── dingtalk.py          # 钉钉
├── whatsapp.py          # WhatsApp
├── signal.py            # Signal
├── matrix.py            # Matrix
├── mattermost.py        # Mattermost
├── email.py             # Email（IMAP/SMTP）
├── sms.py               # SMS（Twilio 等）
├── bluebubbles.py       # iMessage（BlueBubbles）
├── qqbot/               # QQ 机器人
├── yuanbao.py           # 元宝
├── webhook.py           # 通用 Webhook
└── api_server.py        # REST API Server
```

## 四、GatewayRunner 核心类

### 4.1 Agent 缓存

```python
_AGENT_CACHE_MAX_SIZE = 128        # 最大缓存 128 个 Agent 实例
_AGENT_CACHE_IDLE_TTL_SECS = 3600  # 空闲 1 小时后驱逐

# LRU 缓存，key = session_key（platform:chat_id:thread_id）
# 每个 Agent 实例持有 LLM 客户端、工具 schema、Memory provider 等
```

### 4.2 消息处理流程

```
GatewayRunner._handle_event(event)
    ↓
1. 解析事件 → SessionSource(platform, chat_id, thread_id, ...)
    ↓
2. 获取/创建 Agent（从缓存或新建）
    ↓
3. 检查 Agent 是否正在运行
    ├── 空闲 → 直接处理
    └── 忙碌 → 取决于策略：
        ├── "interrupt" → 中断当前任务，处理新消息
        ├── "queue"    → 排队等待当前任务完成
        └── "steer"    → 注入到当前运行中（mid-turn steering）
    ↓
4. 运行 Agent → run_sync(agent, event)
    ↓
5. 流式输出 → 逐 chunk 发送给用户
    ↓
6. 持久化会话 → 保存消息历史
```

### 4.3 Mid-Turn Steering

```python
# 当 Agent 正在运行时，新消息可以通过 "steer" 模式注入
# [OUT-OF-BAND USER MESSAGE — ...] 标记包裹
# Agent 在下一个工具调用后看到这个消息
```

## 五、会话管理

### 5.1 SessionSource

```python
@dataclass
class SessionSource:
    platform: Platform      # telegram, feishu, discord, ...
    chat_id: str            # 聊天 ID
    thread_id: str          # 线程/话题 ID
    chat_type: str          # dm, group, channel
    sender_id: str          # 发送者 ID
    sender_name: str        # 发送者名称
    message_id: str         # 消息 ID
```

### 5.2 会话重置策略

```python
class SessionResetPolicy:
    manual     # 仅手动 /new 重置
    on_message # 每条消息重置
    timed      # 超时重置
    hybrid     # 组合策略
```

### 5.3 消息历史重放

```python
# Gateway 需要将存储的消息历史转换为 Agent 可理解的重放格式
# 特殊处理：
# - Assistant reasoning 字段保留（DeepSeek/Kimi 思考模式）
# - Tool call 结果保留
# - Telegram observed group context（群聊观察模式）

_ASSISTANT_REPLAY_FIELDS = (
    "reasoning",
    "reasoning_content",
    "reasoning_details",
    "codex_reasoning_items",
    "codex_message_items",
    "finish_reason",
)
```

## 六、Cron 集成

Gateway 后台线程每 60 秒触发一次 Cron tick：

```python
# Gateway 启动时
cron_thread = threading.Thread(target=_cron_ticker, daemon=True)
cron_thread.start()

def _cron_ticker():
    while True:
        time.sleep(60)
        scheduler.tick()  # 检查并执行到期任务
```

## 七、平台 Adapter 接口

所有平台 Adapter 继承自 `BaseAdapter`：

```python
class BaseAdapter:
    async def start(self)           # 启动平台连接
    async def stop(self)            # 停止连接
    async def send(self, chat_id, content, ...)  # 发送消息
    async def receive(self, event)  # 接收消息
    
    # 常用功能
    _send_with_retry()              # 带重试的发送
    _pending_messages               # 待处理消息队列
```

## 八、Busy Ack（忙碌确认）

当 Agent 正在处理任务时，新消息会收到确认反馈：

```python
# 三种模式的确认消息：
# steer:    "⏩ Steered into current run. Your message arrives after the next tool call."
# queue:    "⏳ Queued for the next turn. I'll respond once the current task finishes."
# interrupt: "⚡ Interrupting current task. I'll respond to your message shortly."

# 防抖：每 30 秒最多发送一次确认
_BUSY_ACK_COOLDOWN = 30

# 状态详情（可选）：
# "⏳ Queued (3 min elapsed, iteration 5/90, running: terminal)"
```

## 九、数据流全景

```
平台消息到达（如飞书）
    ↓
FeishuAdapter.receive(event)
    ├── 解析消息内容、发送者、聊天 ID
    └── 转换为统一事件格式
    ↓
GatewayRunner._handle_event(event)
    ├── 构建 SessionSource
    ├── 计算 session_key（platform:chat_id:thread_id）
    ├── 检查 Agent 缓存
    │   ├── 缓存命中 → 使用现有 Agent
    │   └── 缓存未命中 → 创建新 AIAgent
    ├── 检查是否忙碌
    │   ├── 空闲 → 直接运行
    │   └── 忙碌 → 处理策略（interrupt/queue/steer）
    ↓
run_sync(agent, event)
    ├── 加载消息历史 → 构建 conversation_history
    ├── 注入系统提示词（Memory、Skills、平台上下文）
    ├── agent.run_conversation(user_message)
    │   ├── Agent Loop → LLM 调用 → 工具调用
    │   └── 流式输出 → 逐 chunk 回调
    ↓
流式输出处理
    ├── StreamingContextScrubber（过滤 <memory-context>）
    ├── Media 路由（MEDIA:/path → 上传文件）
    └── adapter.send(chat_id, content)
    ↓
会话持久化
    ├── 保存消息到 SessionDB（SQLite）
    └── 更新 Agent 缓存
```

## 九B、访问控制与安全策略

源码：`gateway/run.py`（`_adapter_dm_policy` / `_check_access`）、`gateway/pairing.py`

### 9B.1 三层访问策略

Gateway 对每个传入消息执行三层访问控制：

| 层 | 配置项 | 说明 |
|---|---|---|
| **DM 策略** | `dm_policy` | `open`（接受所有私聊）/ `allowlist`（仅白名单用户）/ `deny`（拒绝所有私聊） |
| **群聊策略** | `group_policy` | `open` / `allowlist` / `deny`（同上，针对群聊） |
| **来源白名单** | `allow_from` | 按 `platform:user_id` 精确匹配的白名单 |

**重要**：飞书的 `group_policy` 通过环境变量 `FEISHU_GROUP_POLICY` 读取（而非 config.yaml 中的字段），默认 `allowlist` 会静默丢弃群消息。

### 9B.2 配置方式

```yaml
# config.yaml — 全局默认
gateway:
  dm_policy: allowlist
  group_policy: open

# 环境变量 — 平台级覆盖
TELEGRAM_ALLOWED_USERS=123456,789012    # Telegram 白名单
DISCORD_ALLOWED_CHANNELS=111,222        # Discord 白名单
FEISHU_GROUP_POLICY=open                 # 飞书群聊策略
```

无白名单且非 `open` 时，Gateway 启动日志会输出警告。

### 9B.3 DM Pairing（代码配对）

源码：`gateway/pairing.py:PairingStore`

对于没有稳定用户 ID 的平台，Gateway 支持代码配对认证：

```
用户发送 /start → Gateway 生成 6 位配对码
    ↓
用户在 CLI 执行 hermes pair <code>
    ↓
PairingStore 记录 (platform, user_id) → user_authorized
```

### 9B.4 速率限制

```python
# PairingStore 内置速率限制
if self.pairing_store._is_rate_limited(platform_name, source.user_id):
    # 拒绝请求，不创建 Agent
    return
    
self.pairing_store._record_rate_limit(platform_name, source.user_id)
```

Agent 层面也有 `RateLimitTracker`（`agent/rate_limit_tracker.py`），跟踪每个 Provider 的 rate limit 状态（剩余请求数、重置时间），在 `/usage` 命令中展示。

### 9B.5 启动安全检查

```python
# Gateway 启动时检查
_any_allowlist = any([
    TELEGRAM_ALLOWED_USERS, DISCORD_ALLOWED_CHANNELS,
    FEISHU_ALLOWED_USERS, ...
])
if not _any_allowlist and not _allow_all:
    logger.warning(
        "No user allowlists configured. All unauthorized users will be denied. "
        "Set allow_all=true or configure platform allowlists."
    )
```

## 十、关键设计决策

| 设计决策 | 原因 |
|---|---|
| LRU Agent 缓存（128 条） | 避免每次消息都创建新 Agent |
| Mid-turn Steering | 允许用户在 Agent 运行时调整方向 |
| 统一 Adapter 接口 | 新平台只需实现 BaseAdapter |
| reasoning 字段保留 | DeepSeek/Kimi 思考模式需要完整 reasoning 历史 |
| Busy Ack 防抖（30s） | 避免快速发送多条消息时刷屏 |
| 线程池执行 Agent | 不阻塞事件循环 |
| PII 哈希存储 | 日志中的 chat_id/sender_id 脱敏 |
| 60 秒 Cron tick | 在 Gateway 进程内运行，无需额外进程 |
| 三层访问控制 | dm_policy + group_policy + allow_from 防止未授权访问 |
| 环境变量覆盖 config | 平台级配置优先于全局默认，运维友好 |
| 启动安全检查 | 无白名单时警告，防止意外开放访问 |
| DM Pairing 速率限制 | 防止配对码暴力枚举 |

## 十一、学习检查点

- [ ] 能否描述消息从平台到 Agent 的完整路由流程？
- [ ] 理解三种忙碌策略（interrupt/queue/steer）的区别？
- [ ] 理解 Agent 缓存的 LRU + 空闲 TTL 机制？
- [ ] 理解消息历史重放中的特殊字段处理？
- [ ] 能否列举支持的平台和 Adapter 接口？
- [ ] 理解 StreamingContextScrubber 在流式输出中的作用？
- [ ] 能解释三层访问控制（dm_policy / group_policy / allow_from）的配置方式？
- [ ] 理解飞书 group_policy 必须通过环境变量配置的原因？
