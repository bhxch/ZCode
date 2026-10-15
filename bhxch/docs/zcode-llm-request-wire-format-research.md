# ZCode 向大模型后端发请求的 wire 格式调研（header + body）

- 调研日期：2026-10-16
- 基线版本：`main @ 189509f6`（docs: add project-level .mcp.json support research notes）
- 调研方式：源码走读 + wire 层测试断言交叉验证（`apps/zcode-cli/packages/adapters/tests/*wire*.test.ts`）
- 路径约定：下文相对路径均相对仓库根 `F:\github\ZCode`

## TL;DR

| 问题 | 结论 |
| --- | --- |
| 请求由谁发出？ | Vercel AI SDK v6（`ai@6.0.193` + `@ai-sdk/anthropic@3.0.81` / `@ai-sdk/openai@3.0.58` / `@ai-sdk/openai-compatible@2.0.60`），外面包一层 ZCode 自定义 fetch 链。 |
| 有几条请求管道？ | **只有一条**。主对话、子代理、标题生成、compact 等所有场景共用同一 runner + fetch 链，差异只在参数与归因 header。 |
| body 里 reasoning/max_tokens 谁决定？ | **不是代码硬编码**，而是 `config/provider/zcode-builtin.json` 里声明式的 Option Map（受限 CEL 表达式），在 fetch 层以 JSON merge-patch 注入最终 body（`model-option-map-fetch.ts`）。 |
| header 分几层？ | 四层合并：进程级来源头 → provider 静态头（用户配置）→ 请求期认证头 → 每请求归因头。 |
| OpenAI 与 Anthropic 格式的核心差异？ | 认证头（`x-api-key` vs `Authorization: Bearer`）、token 上限字段（`max_tokens` / `max_completion_tokens` / `max_output_tokens`）、reasoning 字段方言（`thinking`/`output_config` vs `reasoning_effort`/`reasoning`/`enable_thinking`）、cache_control（仅 Anthropic）。 |

---

## 1. 请求管道总览

```
调用点（core/services 各场景）
  → Model.generateText / streamText          apps/zcode-cli/packages/contracts/src/model/model.ts:53
  → ExecutableModel（参数校验）               apps/zcode-cli/packages/adapters/src/model/model.ts:72
  → AiSdkModelAdapter（runner）               apps/zcode-cli/packages/adapters/src/model/runner.ts
  → AI SDK generateText/streamText            apps/zcode-cli/packages/adapters/src/model/runner-runtime.ts:43
  → fetch 链（内→外）：
      proxy fetch（HTTP(S) 代理 + 自定义 CA）  adapters/src/network/proxy-fetch.ts
      endpoint-routing fetch（URL 改写）       adapters/src/model/provider-endpoint-routing-fetch.ts
      request-security（开源版恒等透传）       adapters/src/model/request-security-edition/index.ts
      business-error 检测 fetch（仅响应侧）    adapters/src/model/model-execution.ts:474
      option-map fetch（body merge-patch ★）  adapters/src/model/model-option-map-fetch.ts
      anthropic-stream-compat / openai-responses-json-compat（协议兼容修正）
```

Provider kind 判定（`apps/zcode-cli/packages/adapters/src/model/model-execution.ts:383-405`）：

| `api.type` | kind | SDK | 路径 |
| --- | --- | --- | --- |
| `anthropic-messages` | anthropic | `createAnthropic` | `POST {base}/v1/messages` |
| `openai-chat-completions` | openai-compatible | `createOpenAICompatible` | `POST {base}/chat/completions` |
| `openai-responses` | openai | `createOpenAI` → `provider.responses` | `POST {base}/responses` |

Coding Plan 官方网关的 URL 会被 endpoint-routing fetch 改写：`open.bigmodel.cn/api/anthropic/v1/messages → {origin}/api/v1/ultra/anthropic/v1/messages`，`api.z.ai → /api/v1/ultra-zai/...`（`official-coding-plan-gateway.ts:8-11`），改写后删除 `host` 头。

---

## 2. Header 构成

### 2.1 进程级来源头（defaultHeaders，所有 provider、所有模型请求都带）

`apps/zcode-cli/packages/bootstrap/src/model-config.ts:50-69` + `runtime-platform-headers.ts`：

| Header | 值 | 说明 |
| --- | --- | --- |
| `User-Agent` | `ZCode/<appVersion>` | 版本取 `ZCODE_APP_VERSION` 环境变量 → 启动参数 → `unknown` |
| `HTTP-Referer` | ZCode 端点 origin | `resolveRuntimeZCodeEndpointOrigin(env)` |
| `X-ZCode-App-Version` | appVersion | 有值才发 |
| `X-Title` | `Z Code@cli` 或 `Z Code@electron` | 按 `process.argv` 含 `app-server`/`agent-server` 判定 |
| `X-Release-Channel` | `resolveRuntimeZCodeEnv(env)` | 发布渠道 |
| `X-Client-Language` | `Intl` locale | |
| `X-Client-Timezone` | `Intl` timeZone | |
| `X-ZCode-Agent` | 固定 `"glm"` | |
| `X-Platform` | `${process.platform}-${arch}`，如 `win32-x64` | |
| `X-Os-Category` | `macos` / `windows` / `linux` | |
| `X-Os-Version` | `os.release()` | |
| `X-OpenRouter-Title` / `X-OpenRouter-Categories` | `ZCode` / `programming-app` | 仅 baseURL 为 `openrouter.ai`（`packages/shared/src/openrouter-attribution.ts`） |

CLI 模型请求不带 `X-Device-Mid`；该头只出现在 electron/off-peak ticket 等走 `buildZCodeSourceHeaders`（`packages/services/src/providers/sourceHeaders.ts`）的路径。

### 2.2 请求级归因头（每次请求动态生成）

`apps/zcode-cli/packages/adapters/src/model/runner-attribution.ts:29-53`（作为 AI SDK 调用级 `headers` 传入，覆盖静态同名头）：

| Header | 值 | 条件 |
| --- | --- | --- |
| `x-request-id` | requestId | 总是 |
| `x-zcode-session-type` | `main` / `subagent` / `side_chat` / `other` | 总是；Coding Plan 服务端用它区分请求来源（源码注释明示） |
| `x-zcode-trace-id` | traceId | 总是 |
| `x-query-id` | queryId（剥内部 `query_` 前缀） | 有 queryId |
| `x-zcode-query-source` | querySource（正则 `^[A-Za-z0-9_.:-]{1,128}$` 校验） | 有值；静态配置中的同名头必被剔除 |
| `x-session-id` | sessionId（剥 `sess_`/`subagent_agent_` 前缀） | 有 sessionId |
| `x-opencode-session` | sessionId | 仅 opencode.ai Go 网关 baseURL |

### 2.3 认证头（按 provider / 场景）

| 场景 | Header 组成 | 来源 |
| --- | --- | --- |
| 自定义 API key（anthropic kind） | `x-api-key`（SDK 注入）+ `Authorization: Bearer <key>`（`withAnthropicAuthorizationHeader`，model-execution.ts:425-438） | 用户配置 `apiKey` |
| 自定义 API key（openai kind） | `Authorization: Bearer <key>`（SDK 注入） | 同上 |
| Coding Plan OAuth（zhipu-account） | 同上（PAT 作为 apiKey），每次 attempt 前经 `refreshRuntimeHeadersBeforeAttempt` 重取 | OAuth JWT → 换短期 PAT（`auth/coding-plan-api-key.ts`） |
| Off-peak 闲时 | `x-api-key` + `Authorization: Bearer` 均为 zcode JWT；另加 `X-Coding-Plan-Api-Key: <PAT>`、`X-Off-Peak-Ticket-ID: <ticketId>`；Team 凭证再加 `bigmodel-organization` / `bigmodel-project` | `packages/services/src/session/offPeakRuntimeModel.ts:229-265`；每次 attempt 仅刷新 PAT（`off-peak-request-auth.ts`） |
| Highspeed 加速卡 | `Authorization` + `X-Bigmodel-Authorization` 双 JWT、`Bigmodel-Target-Type`（+ org/project）、`X-Highspeed-Card-ID` | `packages/services/src/highspeed/highspeedCardService.ts:72-152` |

### 2.4 SDK / 协议头

- `anthropic-version: 2023-06-01`：`@ai-sdk/anthropic` SDK 默认注入（wire 测试断言 `adapters/tests/model-execution.test.ts:56-59`）。
- `anthropic-beta`：发 PDF 内容时 SDK 自动加 `pdfs-2024-09-25`；会话中段 system（MCS）场景按 provider 配置注入 `mid-conversation-system-2026-04-07`（静态头，`adapters/tests/memory-provider-body.test.ts:181-186`）。
- `content-type: application/json` 等传输层标准头由 SDK 注入。

### 2.5 fetch 链对 header 的处理

只有 option-map fetch 改 body，其余层对 header 基本不动：endpoint-routing 改写 URL 后删 `host`；business-error/compat fetch 只删响应侧 `content-length`/`content-encoding`；诊断日志对 `authorization`/`x-api-key`/`cookie`/`x-off-peak-ticket-id` 等脱敏（`runner-network-headers.ts:4-37`），但不影响外发请求。

---

## 3. Body — Anthropic 格式（/v1/messages）

组装顺序：AI SDK 序列化（`runner-options.ts` 的调用参数）→ option-map fetch merge-patch → anthropic-stream-compat 最后修正（memory selector 消息边界还原、中段 system string 化）。

| 字段 | 取值 | 来源 |
| --- | --- | --- |
| `model` | resolved modelId | 用户/Registry 选型 |
| `max_tokens` | Option Map 注入 `{'max_tokens': maxOutputTokens}` | 主 turn = 模型声明上限（zcode-builtin.json，如 GLM 系 131072、kimi-k2.6 65536），未声明默认 32,000，再按 `contextWindow − 估算输入 − 1000` 收缩（`core/src/model/model-token-limits.ts`）；辅助调用 `min(5000, max)`；compact `min(effective, 20K)`；WebSearch/WebFetch 内调用 `min(4096, max)` |
| `system` | **最多 3 条消息**转为顶层 system 数组 | `core/src/context/builder.ts:234-280`：① CLI prefix `"You are ZCode, an interactive coding agent"`（无 cache 标记）；② stable body（自定义 systemPrompt / 子代理身份 / 默认身份+SECURITY_NOTICE+#Harness，带 `cache_control:{type:"ephemeral"}`）；③ dynamic body（Desktop 上下文、`# Environment`（cwd/git/platform/OS/模型自述）、gitStatus 快照、Memory 段、Output Style、Context Management，带 ephemeral） |
| `messages` | user（text/image/video/PDF）、assistant（text + tool-call + reasoning 带 `signature`）、tool-result | `adapters/src/model/transform.ts:46-157`；空 content 用 `"(no content)"` 占位；发送前做历史归一化（删跨模型签名 reasoning、reasoning-only assistant 等，`reasoning-history-normalization.ts`，仅 anthropic kind） |
| `tools` | function 工具（`input_schema`）；strict 仅裸 `claude-` 首方模型（不支持的约束折进 description）；provider-native 工具仅 `webSearch_20260209`（maxUses 默认 8） | `tool-transform.ts:20-79`、`strict-tool-schema.ts` |
| `tool_choice` | 透传（主 turn 基本不设） | `runner-options.ts:74` |
| `thinking` / `output_config` | **Option Map 注入**。catch-all 规则（zcode-builtin.json modelApiRules[1]）：`disabled → {"thinking":{"type":"disabled"}}`；否则 `{"thinking":{"type":"adaptive"},"output_config":{"effort": <low/medium/high/max，enabled→high>}}`。模型级覆盖：`glm-5.x → thinking:{type:enabled|disabled}`；`kimi-k3 → 仅 output_config.effort`；openrouter GPT 系 → 仅 `output_config.effort` | **全仓库无 `budget_tokens` 映射**（wire 测试明确断言） |
| `metadata` | `metadata.user_id = JSON.stringify({device_id, account_uuid:"", session_id})` | `anthropic-request-metadata.ts:7-32`，device_id 来自本地 telemetry-state 的 deviceMid |
| `stream` | 流式场景 `true` | 主 turn 按 `shouldStreamModelText()` 决定 |
| 结构化输出 | `output_config.format:{type:"json_schema",...}` | 有 `responseJsonSchema` 且 anthropic 时强制（`runner-options.ts:193-209`） |
| `temperature`/`top_p`/`top_k`/`stop_sequences`/`seed` | 通道存在但 core 从不设置 → **实际请求不发送**，用 provider 默认采样 | `runner-options.ts:75-81` + `removeUndefined` |

**cache_control（prompt caching）策略**：system 的 stable/dynamic 两条固定带 ephemeral；消息级只保留**最后一条非 system 消息**作为唯一 cache write breakpoint，其余非 system 消息上的标记被清除（`core/src/runtime/helpers/provider-request-messages.ts:292-333`）；memory recall 的 manifest/selector 消息也带 ephemeral。

**哪些 provider 走此格式**：anthropic 官方、zai-api、bigmodel-api、moonshot-kimi、minimax、deepseek、qwen-cn、xiaomi-mimo、openrouter（anthropic 端点）、opencode zen/go messages；账号型（zai/bigmodel individual/team coding-plan、start-plan、off-peak、highspeed）**一律 anthropic-messages**。

---

## 4. Body — OpenAI 格式

### 4.1 chat/completions

| 字段 | 取值 | 来源 |
| --- | --- | --- |
| `model` | modelId 原样 | |
| `messages` | 开头多条 system **合并为一条**（旧式 chat template 只认开头一个 system，`system-message-compat.ts`）；空 user 补 `"(no content)"`；空工具名替换为 `"empty_tool_name"`；**不回放** DeepSeek 式 `reasoning_content`（wire 测试断言） | `transform.ts` |
| `tools` | `{type:"function", function:{name,description,parameters}}`，**从不带 strict** | `tool-transform.ts` |
| `max_completion_tokens` | Option Map 注入 `{'max_completion_tokens': maxOutputTokens}` | 值来源同 §3 |
| `stream` + `stream_options` | `stream:true`；工厂 `includeUsage:true` → `stream_options:{include_usage:true}` | `model-execution.ts:345` |
| `response_format` | `{type:"json_schema", json_schema:{...}}` | 仅 generateText + schema 且模型声明 `supportsJsonSchemaOutput`，否则 SDK 静默降级 |
| reasoning 四字段**并发**（catch-all，zcode-builtin.json modelApiRules[2]） | `thinking:{type:enabled|disabled}` + `enable_thinking:bool` + `reasoning_effort:none|<level>` + `reasoning:{effort:...}`，一次写入，兼容端各取所需；模型级覆盖如 `kimi-k3 → 仅 reasoning_effort`、`qwen3.5-plus → 仅 enable_thinking`、openrouter 部分站 → `reasoning:{enabled}` | **没有代码级映射**，全部是配置声明 |
| 不存在的字段 | `user`、`service_tier`、`verbosity`、`logprobs`、`n`、`logit_bias`、`temperature` 等 | 全仓库无引用，core 不设置采样参数 |

### 4.2 responses（OpenAI Responses API）

- 走 `provider.responses` → `POST {base}/responses`；使用方：openai 官方、xai、opencode zen/go responses 模板及 gpt-5.x、deepseek 等模型的 responses 变体。
- reasoning：`reasoning:{effort}`（disabled→`"none"`，enabled→`"high"`）；token 上限字段为 `max_output_tokens`；结构化输出走 `text:{format:{type:"json_schema"}}`。
- ZCode 不设置 `store`/`previousResponseId`/`conversation`/`verbosity`；历史回放时丢弃带 `itemId` 的 stored reasoning 引用。
- 有专属兼容 fetch 给缺 `message.id` 的响应补 id、给 `output_text` 补 `annotations: []`（`openai-responses-json-compat.ts`）。

---

## 5. 场景区分（全部走同一条管道）

| 场景 | 调用 | 模型 | maxOutputTokens | reasoning | 工具 |
| --- | --- | --- | --- | --- | --- |
| 主 agent 对话 turn | streamText（配置开启）否则 generateText | 会话当前模型 | 模型上限 / 默认 32K，按剩余窗口收缩 | 会话绑定档位 | 完整工具集 |
| 子代理（Task/Agent 工具） | 创建完整子 AgentRuntime → 同主 turn 管道 | Server override > agent profile frontmatter > 继承父模型 | 同主 turn | selection 自带 | profile 工具集 |
| Dynamic workflow actor | 同上 | persona.model > run 级 `subagent_model` > journal pinnedModel > 会话模型 | 同主 turn | selection 自带 | actor 工具 |
| 会话标题/目标标题 sidecar | generateText | `config.titleGeneration?.modelSelection ?? 会话模型` | `min(5000, max)` | 最低公开档位 | 无；专用 system prompt（要求返回 `{"title":"..."}` JSON），输入截断 1200 字符 |
| Compact 摘要 | streamText 优先 + 特殊重试链 | 会话模型 | `min(effective, 20K)` | 继承档位 | 工具数超阈值时为空 |
| Goal verifier | generateText | 会话模型快照 | **模型上限**（明确不套辅助降档） | 继承档位 | 无 |
| 连通性探测 | streamText | 调用方传入 | **1 token** | 最低档位 | 无；system 固定 `"You are ZCode connectivity probe."` + user `"hi"` |
| Git commit message / workspace 生成 | generateText | 调用方传入 | auxiliary（5k） | 最低档位 | 一般无 |
| WebSearch 工具 | streamText | 当前 turn 模型（需 `supportsNativeWebSearch`） | `min(4096, max)` | 最低档位 | 单个 provider-native web_search |
| WebFetch 加工 / ReadSessionContext 提取 | generateText | 当前 turn 模型 | `min(4096/用户值, max)` | 最低档位 | 无 |
| MCP sampling | generateText | 请求指定模型 | `min(params.maxTokens, max)` | 默认 | 无 |
| Memory recall selector / memory agent / dream | generateText | turn 模型 | auxiliary（5k） | 最低档位 | selector 无；memory agent 有完整工具目录但权限收窄为只读 |
| Off-peak 闲时推理 | 同主 turn 完整管道 | off-peak 票据绑定模型 | 同主 turn | selection 自带 | 同主 turn；header 见 §2.3 |

场景间 header 集合完全一致，仅归因头的值不同（`x-zcode-session-type` 等）；差异集中在 max_tokens、reasoning 档位、工具集与 system prompt。

非推理请求（管理 API，不走上述管道）：off-peak 票据控制面（`/api/v1/off-peak*`）、用量 quota 查询、订阅账单、OAuth 登录换证。这些请求由通用 `NodeHttpClientAdapter` 发出，自动注入 `x-zcode-trace-id`。模型列表为纯本地 Registry 视图，无远程拉取。

---

## 6. 关键文件索引

- 工厂/kind 映射：`apps/zcode-cli/packages/adapters/src/model/model-execution.ts`
- SDK 调用参数：`apps/zcode-cli/packages/adapters/src/model/runner-options.ts`、`runner.ts`、`runner-generate.ts`
- body 注入权威：`apps/zcode-cli/packages/adapters/src/model/model-option-map-fetch.ts`；`packages/model-option-map/src/{option-maps,merge-patch}.ts`
- 规则数据：`config/provider/zcode-builtin.json`（`modelRules` → `modelApiRules` → `providerSiteRules` → `templateModelRules`/`builtinProviderModelRules`，顺序 overlay）
- messages/tools 转换：`transform.ts`、`tool-transform.ts`、`strict-tool-schema.ts`、`system-message-compat.ts`
- header：`bootstrap/src/model-config.ts`、`bootstrap/src/runtime-platform-headers.ts`、`adapters/src/model/{runner-attribution,model-request-headers,off-peak-request-auth}.ts`
- 认证：`adapters/src/auth/{bigmodel-oauth,coding-plan-api-key,zai-business-token-cache}.ts`、`packages/shared/src/project-access-token-client.ts`
- system prompt：`apps/zcode-cli/packages/core/src/context/builder.ts` 及 `sections/`
- 场景入口：`apps/zcode-cli/packages/core/src/runtime/methods/`（model、subagent、compact-active、title-generation-sidecar、workspace-generate-text、target-completion-verification）
- wire 证据测试：`apps/zcode-cli/packages/adapters/tests/new-model-wire.test.ts`、`new-model-round2-wire.test.ts`、`openai-compatible-*-wire.test.ts`、`memory-provider-body.test.ts`、`connectivity-probe-wire.test.ts`
