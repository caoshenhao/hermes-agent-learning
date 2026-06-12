# Hermes Agent ç Token / ææ¬ä¼åæºå¶å¨æ¯## æ¦è§Hermes å¨è°ç¨å¤§æ¨¡åæ¶è®¾è®¡äº 20+ ç§ token èçæºå¶ï¼è¦çä»è¯·æ±åéåçå·¥å·ç­éãSchema ç²¾ç®ï¼å°è¯·æ±è¿ç¨ä¸­çç¼å­ãåç¼©ï¼åå°ç»æåçåªæãå¥ç¦»ãæç§è°ç¨é¾è·¯åä¸º 5 å¤§ç±»ã

## ä¸ãè¯·æ±åéå â åå°åç» API ç Token### 1.1 å·¥å·éè¿æ»¤ï¼Enabled/Disabled Toolsetsï¼æä»¶:`model_tools.py` L264-510

æ ¸å¿èçæºå¶ â åªå API åéå®éå¯ç¨çå·¥å· schemaã

`50+ ä¸ªå·¥å· â æéç­éå° 4-20 ä¸ª`- `enabled_toolsets` ç½ååï¼åªå è½½æå®å·¥å·é- `disabled_toolsets` é»ååï¼ä»ç»æä¸­åå»ç¦ç¨å·¥å·é- Cron ä½ä¸å§ç»ç¦ç¨ `cronjob`ã`messaging`ã`clarify` ä¸ç»å·¥å·ï¼`cron/scheduler.py` L62-113ï¼- Webhook æ¥æºåªæ´é² 4 ä¸ªå®å¨å·¥å·ï¼`web_search`ã`web_extract`ã`vision_analyze`ã`clarify`ï¼`toolsets.py` L78-83ï¼### 1.2 æ¸è¿å¼å·¥å·åç°ï¼Tool Search Bridgeï¼æä»¶:`tools/tool_search.py` (735 è¡)

å½ MCP + æä»¶å·¥å·ç schema è¶è¿ä¸ä¸æçªå£ç 10% æ¶ï¼å°è¿äºå·¥å·æ¿æ¢ä¸º 3 ä¸ªæ¡¥æ¥å·¥å·ï¼`tool_search`ã`tool_describe`ã`tool_call`ï¼ï¼æéå è½½ãæ ¸å¿å·¥å·ï¼`_HERMES_CORE_TOOLS`ï¼æ°¸ä¸å»¶è¿ã

èçææï¼20-30K tokens ç schema å¼éã

### 1.3 å¨æ Schema éå»ºæä»¶:`model_tools.py` L409-469

æ ¹æ®å®éè¿è¡ç¯å¢å¨æç²¾ç®å·¥å· schemaï¼

- `execute_code` â åªæ´é² sandbox ä¸­å®éå¯ç¨çå·¥å·- Discord â åªæ´é² bot å·å¤æéçæä½- æµè§å¨ â `web_search` ä¸å¯ç¨æ¶ï¼ä» `browser_navigate` æè¿°ä¸­ç§»é¤å¼ç¨### 1.4 å·¥å· Schema ååæä»¶:`tools/schema_sanitizer.py` (445 è¡)

éå½ç²¾ç®å·¥å· JSON Schemaï¼

ææ¯

å½æ°

ææ

Nullable union æå 

`strip_nullable_unions()`

`anyOf: [string, null]` â `type: string, nullable: true`

Pattern/Format å¥ç¦»

`strip_pattern_and_format()`

ç§»é¤ `pattern`/`format` å³é®å­ï¼æéè§¦åï¼

Slash Enum å¥ç¦»

`strip_slash_enum()`

ç§»é¤å« `/` ç enum å¼

é¡¶å±ç»åå¨å¥ç¦»

`_strip_top_level_combinators()`

ç§»é¤ `allOf`/`anyOf`/`oneOf`/`not`

Gemini éé

`agent/gemini_schema.py`

åªä¿ç Gemini æ¥åç ~25 ä¸ª key

Moonshot éé

`agent/moonshot_schema.py`

å¼ºå¶æ¯ä¸ªå±æ§æ `type` å­æ®µ

### 1.5 å·¥å·å®ä¹ç¼å­æä»¶:`model_tools.py` L243-333

æ¨¡åçº§ç¼å­é¿åæ¯æ¬¡ API è°ç¨é½éæ°æå»ºå·¥å· schemaï¼

`cache_key = (frozenset(enabled), frozenset(disabled), registry._generation, cfg_fp)`### 1.6 å­ Agent å·¥å·ééå¶æä»¶:`tools/delegate_tool.py` L45-53

å­ Agent è¢«æ°¸ä¹å¥å¤º 5 ä¸ªå·¥å·ï¼`delegate_task`ã`clarify`ã`memory`ã`send_message`ã`execute_code`ï¼ï¼ä¸å·¥å·éå¿é¡»ä¸ç¶ Agent äº¤åéªè¯ãå­ Agent ä½¿ç¨éç¦»ä¸ä¸æï¼æ ç¶çº§åå²ï¼ï¼ç¶ Agent åªçå°æç»æè¦ã

## äºãè¯·æ±è¿ç¨ä¸­ â å©ç¨ç¼å­åå°éå¤ Token è´¹ç¨### 2.1 Anthropic Prompt ç¼å­æä»¶:`agent/prompt_caching.py` (79 è¡)

å¨ Anthropic æ¶æ¯ä¸æ¾ç½®æå¤ 4 ä¸ª `cache_control` æ­ç¹ï¼

system prompt- æå 3 æ¡é system æ¶æ¯æ¯æ 5 åéå 1 å°æ¶ TTLã

èçææï¼~75% è¾å¥ token ææ¬éä½ã

### 2.2 Gemini åç API ééæä»¶:`agent/gemini_native_adapter.py`

ç»è¿ OpenAI å¼å®¹å±ï¼ç´æ¥ä½¿ç¨ Gemini åç REST APIï¼é¿åä¸­é´è½¬æ¢å¼éï¼æ¯æ Gemini çä¸ä¸æç¼å­æºå¶ã

## ä¸ãä¸ä¸æåç¼© â å½å¯¹è¯åé¿æ¶èªå¨ç¦èº«### 3.1 ä¸ä¸æåç¼©å¨ï¼Context Compressorï¼æä»¶:`agent/context_compressor.py` (2,078 è¡)

å½å¯¹è¯ token æ°è¶è¿æ¨¡åä¸ä¸æçªå£ç 50% æ¶èªå¨è§¦åï¼4 é¶æ®µææåç¼©ï¼

é¶æ®µ

æä½

æ¯å¦éè¦ LLM

Phase 1

åªææ§å·¥å·è¾åº

â æ  LLM è°ç¨

Phase 2

ç¡®å®ä¿æ¤è¾¹çï¼å¤´é¨ system + å°¾é¨ ~20K tokenï¼

â

Phase 3

LLM æè¦ä¸­é´è½®æ¬¡

â ç¨ä¾¿å®è¾å©æ¨¡å

Phase 4

ç»è£åç¼©åæ¶æ¯åè¡¨

â

åæå¨ä¿æ¤ï¼ è¿ç»­ 2 æ¬¡åç¼©èç &lt;10% åè·³è¿ï¼é¿åæ æåç¼©æµªè´¹ tokenã

### 3.2 æ§å·¥å·è¾åºæºè½åªææä»¶:`agent/context_compressor.py` L754-920

ä¸éæ«æï¼

- Pass 1: å»é â ç¸ååå®¹çå·¥å·ç»æåªä¿çææ°ä¸ä»½- Pass 2: æºè½æè¦ â ç¨å·¥å·ç¹å®æ¨¡æ¿çæ 1 è¡æè¦ï¼`[terminal] ran 'npm test' -&gt; exit 0, 47 lines output
[read_file] read main.py from line 100 (3,200 chars)
[delegate_task] 'research X' (25,000 chars result)`- Pass 3: æªæ­å·¥å·è°ç¨åæ° â &gt;500 å­ç¬¦ç tool_call åæ° JSON å®å¨æªæ­### 3.3 å·¥å·è°ç¨åæ° JSON å®å¨æªæ­æä»¶:`agent/context_compressor.py` L246-289

å°é¿ tool_call arguments å­ç¬¦ä¸²å¼æªæ­å° 200 å­ç¬¦ï¼åæ¶ä¿æ JSON æææ§ã

### 3.4 åå²å¾çå¥ç¦»æä»¶:`agent/context_compressor.py` L343-397

æ¾å°æåä¸æ¡åå«å¾ççç¨æ·æ¶æ¯ä½ä¸ºéç¹ï¼å°éç¹ä¹åæææ¶æ¯ä¸­çå¾çæ¿æ¢ä¸ºææ¬å ä½ç¬¦ï¼

`[Attached image â stripped after compression]`èçææï¼æ¯å¼ å¾èç ~1MB base64 æ°æ®ã

### 3.5 æè¦åºååçåå®¹æªæ­æä»¶:`agent/context_compressor.py` L937-999

åéç»æè¦æ¨¡åçåå®¹æç¡¬æ§ä¸éï¼

åæ°

å¼

è¯´æ

`_CONTENT_MAX`

6,000

æ¯æ¡æ¶æ¯æ»å­ç¬¦ä¸é

`_CONTENT_HEAD`

4,000

ä¿çå¤´é¨å­ç¬¦

`_CONTENT_TAIL`

1,500

ä¿çå°¾é¨å­ç¬¦

`_TOOL_ARGS_MAX`

1,500

å·¥å·åæ°å­ç¬¦ä¸é

### 3.6 è½¨è¿¹åç¼©ï¼Trajectory Compressorï¼æä»¶:`trajectory_compressor.py` (1,579 è¡)

åå¤çå·²å®æç agent è½¨è¿¹ï¼ç¨äºè®­ç»æ°æ®/æ¹éå¤çï¼ï¼å°è¶é¿è½¨è¿¹åç¼©å°ç®æ  token é¢ç®åãé»è®¤ä½¿ç¨ä¾¿å®å¿«éçæè¦æ¨¡åï¼`google/gemini-3-flash-preview`ï¼ã

## åãå·¥å·è¾åºéå¶ â æ§å¶æ¯è½®è¿åç Token é### 4.1 å·¥å·è¾åºç¡¬ä¸éæä»¶:`tools/tool_output_limits.py` (110 è¡)

åæ°

é»è®¤å¼

è¯´æ

`max_bytes`

50,000

terminal stdout/stderr ä¸é

`max_lines`

2,000

read_file åé¡µä¸é

`max_line_length`

2,000

åè¡é¿åº¦ä¸é

å¯éè¿ `config.yaml` ç `tool_output` èèªå®ä¹ã

### 4.2 è¿è¡æ¶å¾çå¥ç¦»æä»¶:`run_agent.py` L4444

éå° provider body-size éå¶éè¯¯æ¶ï¼èªå¨å¥ç¦»å·¥å·æ¶æ¯ä¸­çå¾çé¨åä½ä¸ºæ¢å¤ææ®µã

## äºãæ¨¡åè°ç¨å±é¢çä¼å### 5.1 è¾å©æ¨¡åè·¯ç±ï¼ä¾¿å®æ¨¡ååæè¦ï¼æä»¶:`agent/conversation_compression.py` L64-200

åç¼©ä½¿ç¨è¾å©ï¼ä¾¿å®/å¿«éï¼æ¨¡åèéä¸»æ¨¡åãèªå¨æ£æµè¾å©æ¨¡åä¸ä¸æçªå£æ¯å¦è¶³å¤ï¼ä¸è¶³æ¶èªå¨éä½åç¼©éå¼ã

### 5.2 Token ç²ç¥ä¼°ç®é¢æ£æä»¶:`agent/model_metadata.py` L1828-1925

å¿«é token ä¼°ç®ï¼ä¸è°ç¨ tokenizerï¼ç¨äºé¢æ£ï¼

`estimate_messages_tokens_rough(messages)     # å¾çæ ~1500 tokens/å¼ 
estimate_request_tokens_rough(messages, ...)  # åå« system + tools çå®æ´ä¼°ç®`å½ç²ç³ä¼°ç®åé«æ¶ï¼schema å¼éå¤§ï¼ï¼å»¶è¿å° provider è¿åçå® `prompt_tokens` ååå³ç­ï¼é¿åä¸å¿è¦çåç¼©ã

### 5.3 è¿­ä»£é¢ç®ç®¡çæä»¶:`agent/iteration_budget.py` (62 è¡)

- ç¶ Agent é»è®¤ 90 æ¬¡è¿­ä»£- å­ Agent é»è®¤ 50 æ¬¡è¿­ä»£- Grace Callï¼ é¢ç®èå°½æ¶ç»äºæ¨¡åé¢å¤ 1 æ¬¡æºä¼å®æåç­- `execute_code` è¿­ä»£å¯éæ¬¾### 5.4 æç»´/æ¨çåæ¸çæä»¶:`agent/think_scrubber.py` (386 è¡)

æµå¼è¾åºä¸­å®æ¶ç§»é¤ `&lt;think`&gt;ã`&lt;thinking`&gt;ã`&lt;reasoning`&gt; ç­æ¨çæ ç­¾ï¼é²æ­¢æ¨¡ååé¨æ¨çæ³é²ç»ç¨æ·å¹¶å ç¨ tokenã

## æ»ç»è¡¨#

æºå¶

ä¸»è¦æä»¶

èçææ

1

å·¥å·éè¿æ»¤

`model_tools.py`

50+ å·¥å· â 4-20 ä¸ª

2

æ¸è¿å¼å·¥å·åç°

`tools/tool_search.py`

èç 20-30K token schema

3

å¨æ Schema éå»º

`model_tools.py`

åªæ´é²å¯ç¨å·¥å·

4

Schema åå

`tools/schema_sanitizer.py`

ç§»é¤åä½ schema å­æ®µ

5

å·¥å·å®ä¹ç¼å­

`model_tools.py`

é¿åéå¤æå»º schema

6

å­ Agent å·¥å·éå¶

`tools/delegate_tool.py`

éç¦»ä¸ä¸æ + 5 å·¥å·ç¦ç¨

7

Cron å·¥å·éå¶

`cron/scheduler.py`

3 å·¥å·éå§ç»ç¦ç¨

8

Anthropic Prompt ç¼å­

`agent/prompt_caching.py`

~75% è¾å¥ token ææ¬éä½

9

Gemini åç API

`agent/gemini_native_adapter.py`

é¿åä¸­é´è½¬æ¢å¼é

10

ä¸ä¸æåç¼©å¨

`agent/context_compressor.py`

~50-80% token èç

11

æ§å·¥å·è¾åºåªæ

`agent/context_compressor.py`

å¤§å¹åå°æ¯è½® token

12

å·¥å·è°ç¨åæ°æªæ­

`agent/context_compressor.py`

é²æ­¢ 50KB+ åæ°è¨è

13

åå²å¾çå¥ç¦»

`agent/context_compressor.py`

æ¯å¼ å¾èç ~1MB

14

æè¦åå®¹æªæ­

`agent/context_compressor.py`

ç¡¬æ§å­ç¬¦ä¸é

15

è½¨è¿¹åç¼©

`trajectory_compressor.py`

è®­ç»æ°æ®åç¼©

16

å·¥å·è¾åºç¡¬ä¸é

`tools/tool_output_limits.py`

éå¶ 50KB è¾åº

17

è¾å©æ¨¡åè·¯ç±

`agent/conversation_compression.py`

ç¨ä¾¿å®æ¨¡ååæè¦

18

Token ç²ç¥ä¼°ç®é¢æ£

`agent/model_metadata.py`

é¿åè¶éè¯·æ±

19

è¿­ä»£é¢ç®ç®¡ç

`agent/iteration_budget.py`

ç¡¬æ§è¿­ä»£ä¸é

20

æç»´åæ¸ç

`agent/think_scrubber.py`

ç§»é¤æ¨çè¾åº

ä¸ä¸æ­¥ï¼éæ©æå´è¶£çæºå¶ï¼æ·±å¥éè¯»å¯¹åºæºç ã