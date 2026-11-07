# ZCode 设置体系与环境变量调研

> 调研日期:2026-11-07。基于当前 main 分支源码,所有结论均标注源码位置(路径:行号)。
> 官方用户面文档见 `apps/zcode-cli/docs/usage/03-config.md`,本文是源码级补充,重点回答:项目级设置支持什么、能否给项目配 env、有哪些环境变量、能否自定义环境变量。

## TL;DR

| 问题 | 答案 |
|---|---|
| 支持项目级设置吗? | **支持**。项目根 `zcode.json` 或 `.zcode/config.json`,但字段是刻意收窄的白名单(仅 mcp/hooks/skill/command/plugins 部分字段),不能改代理、存储等运行参数 |
| 项目级能配 env 吗? | **设置文件里不能**。config.json(用户级和项目级)都没有顶层 `env` 字段。项目级 env 的实际途径:① 项目 `.env` 文件(仅 CLI/TUI 直启时加载,**桌面打包态不加载**);② MCP server 的 `env` 字段(只注入该 server 子进程);③ 用户 shell rc(经 snapshot/白名单部分进入) |
| 用户级有哪些设置? | 三套文件:`~/.zcode/cli/config.json`(agent 运行配置,字段最全)、`~/.zcode/v2/setting.json`(桌面 App 偏好)、`~/.zcode/v2/provider_config.json`(个人模型/provider) |
| 有哪些环境变量? | 运行配置类 9 个(`ZCODE_STORAGE_DIR` 等)+ 大量运行时/端点/模型/CUA/桌面类,详见第 6 节清单 |
| 支持自定义环境变量吗? | **不提供"向 Bash 工具/子代理注入任意 env"的设置项**。可写 env 的口子只有:MCP server `env`(覆盖继承环境)、`.env` dotenv(影响 ZCode 自身进程环境)、shell rc 文件 |

---

## 1. 设置文件全景:两套体系 + 三类用户级文件

ZCode 的设置不是一个统一系统,而是按消费方拆分的多个文件:

| 文件 | 位置 | 消费方 | schema 定义 |
|---|---|---|---|
| **agent 运行配置** `config.json` | `~/.zcode/cli/config.json`(用户级)、`<workspace>/zcode.json` 或 `<workspace>/.zcode/config.json`(项目级) | zcode-cli 运行时 | `apps/zcode-cli/packages/adapters/src/config/schema.ts:291-335`(zod) |
| **桌面 App 设置** `setting.json` | `~/.zcode/v2/setting.json` | 桌面端设置页 | `packages/shared/src/validationAppSettings.ts:586-696`(zod),类型 `AppSettings` 在 `packages/shared/src/protocol.ts:257-440` |
| **个人模型配置** `provider_config.json` | `~/.zcode/v2/provider_config.json` | provider 装配 | `packages/provider-node/src/provider-config-file-codec.ts` |
| 目录型资源(非 JSON 设置) | `~/.zcode/AGENTS.md`、`~/.zcode/skills|commands|agents/`、`~/.agents/...` | 指令/技能/命令/子代理发现 | 见各 roots 文件 |

- 用户级 config.json 路径定义:`apps/zcode-cli/packages/adapters/src/config/file-config.adapter.ts:62-63`(`~/.zcode/cli/config.json`),坏文件有隔离备份与原子写。
- 桌面 setting.json 路径:`packages/services/src/setting/settingService.ts:236`。
- provider_config.json 装配:`packages/services/src/model-provider/providerConfigRuntime.ts:43-44`,可用 `ZCODE_PERSONAL_PROVIDER_CONFIG_FILE` 覆盖路径。
- **不存在企业管理策略(managed settings)层**。`ConfigScope.System` 只是内置默认值。
- **仓库不发布独立的 JSON Schema 文件**;config.json 里的 `$schema` 字段只是可选字符串,收下但不解析(`schema.ts:293`)。文档中出现的 `https://zcode.ai/schema/config-v2.json` 只是设计稿示例。

## 2. 用户级 config.json 支持的全部字段

权威 schema:`apps/zcode-cli/packages/adapters/src/config/schema.ts:291-311`(`ZCodeConfigFileSchema`,zod strict + passthrough);运行时类型与默认值:`apps/zcode-cli/packages/contracts/src/config/index.ts:12-80`(ConfigKey)、`:202-258`(RuntimeConfig)、`:290-359`(默认值)。

| 字段 | 子字段 | 说明 |
|---|---|---|
| `$schema` | — | 可选,不校验 |
| `modelStream` | `idleTimeoutMs` | 模型流空闲超时 |
| `permission` | `mode`(plan/build/edit/yolo/guarded/auto)、`allowedTools`、`disallowedTools`、`autoApproveHighRisk`、`allowMediumRiskInAuto` | 权限模式与工具黑白名单 |
| `storage` | `dir`、`sessionDbPath` | 存储根(默认 `~/.zcode`)与会话 DB(默认 `~/.zcode/cli/db/db.sqlite`) |
| `network` | `httpProxy`、`noProxy`、`caCertFile`、`timeout` | 网络出口配置 |
| `features` | `compact`、`rewind`、`subagent`、`memory`、`skill`、`mcp` | 功能开关 |
| `memory` | `use` | 记忆开关 |
| `mcp.servers.<name>` | stdio:`type/command/args/cwd/env` + 公共 `alwaysLoad/isolation/enabled/timeoutMs/protocolVersion`;http/sse:`type/url/headers/oauth` | MCP server 声明 |
| `plugins` | `enabled`、`dirs`、`enabledPlugins`、`extraKnownMarketplaces`、`options`、`suppressedBuiltins` | 插件系统 |
| `skills` | `enabled`、`includeInstructions`、`metadataBudget`、`roots` | 技能系统 |
| `skill` / `command` | `Record<绝对路径, {enable?}>` | 按文件粒度禁用单个技能/命令 |
| `logging` | `level`(debug/info/warn/error)、`format`(text/json) | 日志 |
| `ui` | `locale`(auto/en-US/zh-CN)、`theme`(auto/dark/light) | 界面 |
| `toolConcurrency` | `maxConcurrency` | 工具并发上限(默认 10) |
| `modelAnomalyGuard` | `toolCallWarningThreshold`、`repeatedToolCallWarningThreshold`、`maxBudgetWarningsPerTurn` | 模型异常防护阈值 |
| `hooks` | `enabled`、`timeoutMs`、`maxOutputBytes`、`events.{SessionStart/UserPromptSubmit/PreToolUse/PermissionRequest/PostToolUse/PostToolUseFailure/Stop}[]` | 钩子;每条 hook 为 `{matcher?, hooks:[{type:"process"|"command", command, args?, enabled?, async?, shell?, timeout?, statusMessage?, onError?}]}` |

MCP server 输入归一化(旧键兼容):`environment`→`env`、`enable`→`enabled`、`remote`→`http`、`http_headers`→`headers`、无 `type` 时按 `command`/`url` 推断 —— `schema.ts:360-413`。

**注意:全表没有顶层 `env` 字段。** 唯一的 env 相关配置是 `mcp.servers.<name>.env`(仅 stdio 类型,`schema.ts:85`)。

## 3. 项目级设置:支持,但白名单刻意收窄

### 3.1 文件发现

- 候选文件:每个目录下的 `<dir>/zcode.json` 与 `<dir>/.zcode/config.json`(`packages/shared/src/workspace-hook-config.ts:178-183`)。
- 目录集合:从工作目录**向上逐级**到 `.git` 标记(worktree 根)为止,再反转为根→cwd 顺序(`workspace-hook-config.ts:339-361`),即支持仓库根与子目录多级配置叠加。
- 装载:`apps/zcode-cli/packages/adapters/src/config/project-config.adapter.ts:31-48`。
- 项目 stdio MCP 的相对 `cwd` 会解析为绝对路径(相对 config 所在目录,`project-config.adapter.ts:154-162`)。
- 项目 `plugins.dirs` 与 `extraKnownMarketplaces` 被剥除 —— 禁止工作区注入插件根目录/市场源(`project-config.adapter.ts:132-136`、`config-merger.ts:33-41`)。

### 3.2 项目级允许的字段(全部清单)

`schema.ts:315-335`(`projectConfigFileSchema`,从用户 schema `pick` 后 `strip`):

| 字段 | 允许内容 |
|---|---|
| `mcp` | 完整 `mcp.servers`(含每个 server 的 `env`) |
| `hooks` | 完整 hooks 声明,**但需 workspace trust 才激活**(见 3.4) |
| `skill`、`command` | 绝对路径 → `{enable}` 覆盖 |
| `skills` | 仅绝对路径 key 的 `{enable}` 开关(`:323-334`) |
| `plugins.enabledPlugins`、`plugins.options` | 插件启停与选项 |

源码注释(`schema.ts:313-314`)明确了设计意图:项目配置曾复用用户 schema(可改代理/CA/存储根),现只开放扩展声明,防止项目层改写运行参数、防止新增用户字段自动向项目开放。

**不支持的字段**:`storage`、`network`、`permission`、`logging`、`ui`、`toolConcurrency`、`features` 等全部运行参数,以及任何形式的顶层 `env`。

### 3.3 优先级叠加

六层 scope(`apps/zcode-cli/packages/contracts/src/config/index.ts:157-175`):

```
System(0) < User(10) < Project(20) < Session(30) < Env(40) < Cli(50)
```

装配顺序(`apps/zcode-cli/packages/adapters/src/config/config-factory.ts:119-288`):内置默认 → 用户 config.json → 项目配置(多个文件按根→cwd 序各成一层)→ `ZCODE_*` 环境变量 → CLI 启动参数。

合并实现(`config-merger.ts:26-154`):嵌套对象浅合并;`mcp.servers` 按名字合并;`plugins` 按 id/key 合并;hooks 各来源 events **追加**而非覆盖,项目 `enabled:false` 关不掉用户 hooks(有效值 = 任一来源为 true,`:170-196`)。

**MCP 特例**:同名 server **用户配置覆盖项目配置**(apply 顺序 system→project→user→env→cli,`config-factory.ts:372-398`)。这是扩展声明的专用规则,不改变其他字段的项目>用户优先级。

### 3.4 项目 hooks 的信任机制

项目 config 中的 `hooks` 不直接进入可执行合并,而是进入待信任 side-channel(`project-config.adapter.ts:63-91`,诊断码 `config_project_hooks_pending_trust`),需通过 workspace trust 审核后才激活。桌面保存项目 hooks 后同样要走 trust(`packages/services/src/hooks/hooksService.ts`、`apps/zcode-cli/packages/core/src/hooks/workspace-hook-runtime-admission.ts`)。

### 3.5 项目级其他目录型资源

`<ws>/.zcode/skills|commands|agents`、`<ws>/.agents/skills|commands`、`<ws>/.agents/mcp.json`(兼容格式,顶层 `mcpServers` 键,`packages/services/src/mcp-sync/mcpSyncService.ts:49-56`)。发现逻辑同为 cwd→git 根逐级、同级 `.zcode` 先于 `.agents`。

## 4. 环境变量的注入与清洗机制(理解 env 行为的钥匙)

ZCode 对子进程环境采用**"清洗—封存—定向恢复"**模型,中枢是 `packages/shared/src/runtimeEnv.ts`:

1. **清洗(sanitize)**:host/agent 运行时及其子进程环境中剔除一批键(`:47-102`):`NODE_ENV`、全部 `HTTP_PROXY/HTTPS_PROXY/ALL_PROXY/NO_PROXY`、`NODE_EXTRA_CA_CERTS`、`SSL_CERT_*`、`REQUESTS_CA_BUNDLE`、`CURL_CA_BUNDLE`、`GIT_SSL_CAINFO`、CUA broker 凭据、`OTEL_*` 全家、telemetry 设备标识等;另按正则剔除 `(npm_config|yarn|pnpm)_(...proxy|ca...)`(`:119-120`)。
2. **封存(passthrough)**:清洗前把网络类键打包成单个 JSON 变量 `ZCODE_TOOL_ENV_PASSTHROUGH_JSON`(`:237-248`)。
3. **定向恢复**:Bash/工具子进程边界由 `readZCodeToolEnvPassthroughEnv`(`:250`)恢复原名;代理/CA 则统一从 `network.*` 配置(或 `ZCODE_HTTP_PROXY` 等)派生写回(`apps/zcode-cli/packages/adapters/src/network/subprocess-env.ts:22-57`)。CUA broker 凭据只在启动目标 MCP server 时定向注入,绝不回写全局 env。

### 4.1 Bash 工具子进程的环境构造链

```
process.env
  → sanitizeZCodeRuntimeEnvInPlace          (剔除敏感/网络键)
  → applyExecutionTextEnv                   (强制 UTF-8: PYTHONIOENCODING/LANG 等)
  → applyNetworkEgressEnv                   (恢复代理/CA,来源是 network.* 配置而非继承值)
  → 应用 overlay                            (ZCODE_SESSION_ID/ZCODE_PROJECT_DIR/CLAUDE_* 别名)
  → spawn(可选 shell init snapshot 前置)
```

源码:`apps/zcode-cli/packages/adapters/src/exec/execution-command.ts:28-69`(`buildExecutionEnv`)、`apps/zcode-cli/packages/core/src/tool/handlers/bash.ts:457-473`(overlay)。

**login shell 环境白名单**(`packages/services/src/runtime-tools/runtimeCommandEnv.ts:48-110`):login shell 探测(`shell -ilc 'env -0'`)得到的变量中,只有匹配白名单的键(KUBECONFIG、SSH_AUTH_SOCK、AWS_*、LANG/LC_*、XDG_*、NVM/FNM/VOLTA、CARGO、GOPATH、JAVA_HOME、CONDA/PYENV/VIRTUAL_ENV、MISE/DIRENV 等)才继承进运行时,其余丢弃;`NODE_OPTIONS` 明确不带回。PATH 由 snapshot + 运行时工具目录合成。shell alias/函数经 shell init snapshot(`exec/shell-init-snapshot.ts`,固化成 `~/.zcode/.../shell-snapshots/snapshot-*.sh`,30 天保留)。

**结论:Bash 工具能看到的自定义变量 = 用户 shell rc 中定义的变量(受白名单/snapshot 机制约束)+ ZCode 定向注入的会话变量;设置文件里的任何字段都不会变成 Bash 环境变量(网络类除外——由 `network.*` 派生成代理键)。**

### 4.2 MCP server 子进程的 env 注入(唯一的设置文件 env 口子)

`apps/zcode-cli/packages/adapters/src/mcp/index.ts:1960-1990`(`createTransport`):

```
env = applyMcpNodeFallbackEnv({
  ...buildMcpStdioEnv({ env: hostEnv, network }),  // ① 清洗后的宿主环境 + 网络出口 + node PATH 前置
  ...config.env,                                    // ② config.json 里该 server 声明的 env(覆盖 ①)
}, nodeFallback)
```

即 **`mcp.servers.<name>.env` 覆盖继承环境**,只注入该 server 的 spawn 环境。项目级 `.zcode/config.json` 声明的 MCP env 同样生效(需经 workspace MCP 信任审核,审核界面会展示声明的 env 键:`apps/zcode-cli/packages/bootstrap/src/workspace-mcp-trust/index.ts:210,242`)。HTTP/SSE 型 MCP 无 `env`,只有 `headers`/`oauth`。

### 4.3 `.env` 文件(dotenv)——项目级 env 的主要途径,但有形态限制

- **CLI/TUI 直启**(`zcode` 进目录、`zcode run`、login):从 cwd 向上查找 `.env`,`override:false`(不覆盖已有变量)—— `apps/zcode-cli/packages/cli/src/env.ts:50-108`,调用点 `prompt-command.ts:177`、`run.ts:559`、`tui-auth.ts` 等。加载后这些值进入 ZCode 进程自身环境,并按第 4 节规则继承给子进程(网络类键仍会被清洗/封存处理)。
- **桌面 App(打包态):不加载项目 `.env`**。两道闸:
  - 桌面 host 进程:`loadHostProcessEnvFromLocalFiles` 在打包态直接返回(`packages/desktop/src/main/desktopRuntimeEnv.ts:160-163`),仅开发态加载 workspace 根与 desktop 包的 `.env*`;
  - agent 协议子进程(app-server):`shouldLoadCliDotenvForProtocolServer` 仅 development 运行时为真,`run.ts:263-273` 的注释明确说明原因——打包态是 desktop host 的内部协议子进程,读用户 `.env` 的失败或污染会在协议建立前退出,外层只能看到 transport closed。
- 官方文档同样提醒:评测工作区及父目录不要遗留意外的 `.env`(`apps/zcode-cli/docs/usage/03-config.md`)。

### 4.4 hooks / 自定义命令的 env

- hook schema **没有 env 字段**(`schema.ts:237-262`);hook 子进程走 `executionPort.run`,env = `buildExecutionEnv` + 插件 overlay(`ZCODE_SESSION_ID/ZCODE_PROJECT_DIR/ZCODE_PLUGIN_*` 及 CLAUDE_* 别名,`apps/zcode-cli/packages/core/src/hooks/configured-runner-input.ts:74-98`),命令串支持 `${ZCODE_SESSION_ID}` 等模板展开(`:100-140`)。
- 自定义 slash 命令的 `!` bash 块转交 Bash 工具执行,环境同 4.1。
- **AGENTS.md 只是指令文本,不支持声明 env**(用户级 `~/.zcode/AGENTS.md`,`packages/services/src/paths.ts:72-75`)。

## 5. 设置的修改入口写到哪个层级

| 入口 | 写入文件 |
|---|---|
| 桌面设置页 general/appearance/browser 等 | `~/.zcode/v2/setting.json`(用户 App 级) |
| 桌面设置页 MCP/Hooks/Skills/Plugins(无 workspace 上下文) | `~/.zcode/cli/config.json`(用户 agent 级) |
| 桌面设置页 MCP/Hooks/Skills(带 workspace 上下文) | `<ws>/.zcode/config.json`(项目级;hooks 另需 trust) |
| TUI `/locale` | config.json(默认用户级;仅当项目文件已声明 `ui.locale` 时写项目,`bootstrap/src/app/locale-selection.ts:3-10`) |
| TUI `/plugins` enable/disable | `~/.zcode/cli/config.json`(`bootstrap/src/plugins.ts:472`) |
| TUI `/mode` `/model` `/effort` `/mcp` | 仅 session 级内存(`ConfigPort.set` 不落盘,`adapters/src/config/index.ts:343-346`) |

CLI 没有通用 `/config` 命令;内建命令集见 `packages/shared/src/zcode-slash-command-help.ts:9-199`。

## 6. 环境变量完整清单

### 6.1 运行配置类(进入配置叠加的 Env 层)

来源:`apps/zcode-cli/packages/adapters/src/config/env-config.adapter.ts`。

| 变量 | 作用 | 默认值 |
|---|---|---|
| `ZCODE_STORAGE_DIR` | 存储根目录(beta 入口默认 `~/.zcode-beta`,`cli/src/env.ts:133-139`) | `~/.zcode` |
| `ZCODE_SESSION_DB_PATH` / `ZCODE_SESSION_DB` | 会话 DB 路径 | `~/.zcode/cli/db/db.sqlite` |
| `ZCODE_HTTP_PROXY` | 出站 HTTP 代理(优先级低于 config `network.httpProxy`,`adapters/src/network/http-config.ts:71-92`) | — |
| `ZCODE_NO_PROXY` | 代理绕过规则 | — |
| `ZCODE_AGENT_CA_CERT` | 自定义 CA PEM 路径 | — |
| `ZCODE_HTTP_TIMEOUT` / `ZCODE_TIMEOUT` | 网络超时(ms) | — |
| `ZCODE_LOG_FORMAT` | 日志格式 text/json | text |
| `ZCODE_MAX_TOOL_CONCURRENCY` | 工具并发上限 | 10 |
| `ZCODE_LOG_DIR` | 日志目录(`adapters/src/logging/index.ts:203`) | `~/.zcode/cli/log` |
| `ZCODE_LOG_CONSOLE` | =1 时日志镜像 stderr | 关 |

`ZCODE_LOG_LEVEL` **不是**受支持的设置(官方文档 `03-config.md:92` 明确)。旧 `ZCODE_MODEL`/`ZCODE_API_KEY` 不再是通用模型覆盖入口(`03-config.md:70`)。

### 6.2 运行时形态/端点类

| 变量 | 作用 | 读取位置 |
|---|---|---|
| `ZCODE_RUNTIME_ENV` | 运行形态 development/production/test(取代 NODE_ENV 的权威键) | `packages/shared/src/runtimeEnv.ts:122-135` |
| `ZCODE_ENV` | 产品环境 test/production(编译期 define + 运行时) | `packages/shared/src/env.ts:15-21` |
| `ZCODE_BASE_URL` / `ZCODE_ENDPOINT_ORIGIN` | ZCode API origin(前者优先) | `packages/shared/src/zcodeEndpoint.ts:217-218` |
| `ZCODE_TEST_BASE_URL` / `ZCODE_PRODUCTION_BASE_URL` | 按环境分流地址 | `zcodeEndpoint.ts:209-212` |
| `ZCODE_DATA_BASE_DIR` | 数据基目录(provider/凭据) | `packages/services/src/paths.ts:14,36-43` |
| `ZCODE_WINDOWS_APP_INSTALL_DIR` | Windows 安装目录守卫 | `paths.ts:13,135` |
| `ZCODE_BETA` | =1 时默认 storage 切 `~/.zcode-beta` | `cli/src/env.ts:133-139` |
| `ZCODE_HOME` | 遥测 deviceMid 解析根 | `apps/zcode-cli/packages/telemetry/src/bootstrap.ts:129` |
| `ZCODE_APP_VERSION` / `ZCODE_BUILD_COMMIT_ID` | 版本身份注入 | `packages/shared/src/env.ts:43-44` |
| `ZCODE_DEBUG` | host inspect 端口 | `shared/src/env.ts:48-49` |
| `ZCODE_SETTING_WRITE_QUEUE_TIMEOUT_MS` | 设置写队列超时 | `packages/services/src/setting/settingsWriteQueue.ts:2-7` |

### 6.3 模型/provider 类

| 变量 | 作用 |
|---|---|
| `ZCODE_BUILTIN_PROVIDER_CONFIG_FILE` / `ZCODE_BUILTIN_PROVIDER_BUNDLED_CONFIG_FILE` | 内置 provider 配置活动/随包文件路径(`cli/src/provider-runtime-env.ts:58-100`) |
| `ZCODE_PERSONAL_PROVIDER_CONFIG_FILE` | 个人 provider 配置文件路径(默认 `~/.zcode/v2/provider_config.json`) |
| `ZCODE_MODEL_RETRY_MAX_RETRIES` / `_BASE_DELAY_MS` / `_BACKOFF_FACTOR` / `_MAX_DELAY_MS` | 模型请求重试策略(默认 10/2000ms/2/60000ms,`adapters/src/model/retry-policy.ts:18-66`) |
| `ZCODE_BIGMODEL_USAGE_API_KEY` | 用量查询 key(回退 `BIGMODEL_USAGE_API_KEY`) |
| `ZCODE_FEEDBACK_API_BASE` | 反馈上报 API |
| `GLM_BINARY_PATH` | agent 二进制路径(注意无 ZCODE_ 前缀,`packages/shared/src/zcode-agent-runtime.ts:28`) |
| `ZCODE_AGENT_WORKDIR` | agent 运行时工作目录 |
| `ZCODE_BFS_BINARY` / `ZCODE_RG_BINARY` / `ZCODE_UGREP_BINARY` | 内置搜索工具二进制覆盖 |
| `ZCODE_LARK_CLI_BINARY` | Lark CLI 二进制 |

### 6.4 ZCode 写入子进程的会话变量(供插件/hook/脚本读取)

| 变量 | 说明 |
|---|---|
| `ZCODE_SESSION_ID` / `ZCODE_PROJECT_DIR`(+ `CLAUDE_SESSION_ID`/`CLAUDE_CODE_SESSION_ID`/`CLAUDE_PROJECT_DIR` 别名) | Bash(`core/src/tool/handlers/bash.ts:427-436`)、hook、插件 MCP 均注入 |
| `ZCODE_PLUGIN_ID/NAME/ROOT/DATA`(+ CLAUDE_PLUGIN_* 别名) | 插件 MCP/hook;`ZCODE_PLUGIN_ID` 由 resolver 权威覆写防伪造 |
| `ZCODE_OFFICIAL_ENDPOINT(_TOKEN)` | 官方命令端点本地代理,每 session 生成,只注入该 session 的子进程 |
| `ZCODE_TOOL_ENV_PASSTHROUGH_JSON` | 封存的网络变量(见第 4 节) |
| `ZCODE_WORKSPACE_IDENTITY` | 远端 workspace 身份键 |

其余如 `ZCODE_CUA_*`(CUA broker/开发开关)、`ZCODE_DESKTOP_*`(桌面 e2e 隔离)、`ZCODE_PACKAGED_AGENT_OTEL_*`(构建期)、`ZCODE_E2E_*`/`ZCODE_OFFPEAK_MOCK*`(仅测试)等属于内部/测试用途,清单见 `packages/shared/src/runtimeEnv.ts` 与 `packages/desktop/src/main/desktopRuntimeEnv.ts`。

### 6.5 非 ZCODE_ 前缀的相关变量

| 变量 | 行为 |
|---|---|
| `HTTP_PROXY/HTTPS_PROXY/ALL_PROXY/NO_PROXY` | 用户 shell 值被剔除封存;子进程代理统一由 `network.*`/`ZCODE_HTTP_PROXY` 派生写回(`subprocess-env.ts:22-30,71-111`) |
| `NODE_EXTRA_CA_CERTS`、`SSL_CERT_FILE` 等 | 同上,CA 从 `network.caCertFile` 派生 |
| `NODE_ENV` | 刻意不作为 ZCode 判据(`cli/src/env.ts:125-127`) |
| `NODE_OPTIONS` | 从 agent 运行时剔除,login shell 白名单也不带回 |
| `OTEL_EXPORTER_*` 等 | 仅 agent telemetry bootstrap 定向注入 |
| `HOME`/`USERPROFILE` | 用户级 AGENTS.md 解析;agent 子进程 HOME 重定向到 dataBaseDir |
| `PYTHONIOENCODING`/`LANG`/`LC_*` | ZCode 主动写入子进程强制 UTF-8 |

## 7. "我想让某个项目有自己的 env"——可行路径总结

按"生效范围 × 可靠性"排列(均为当前源码实际行为):

1. **MCP server env**(最正规):`<ws>/.zcode/config.json` 或 `zcode.json` 的 `mcp.servers.<name>.env`,只注入该 MCP server 子进程,首次需信任审核;同名 server 用户配置优先。
2. **项目 `.env`**:CLI/TUI 直启(`zcode`、`zcode run`)时从 cwd 向上加载进 ZCode 进程环境,子进程(含 Bash)按继承+清洗规则可见非网络类键;**桌面打包态会话完全不加载**(host 与 app-server 两道闸,见 4.3)。若在桌面会话里需要,只能让 agent 在命令里显式 `source` 或用 dotenv 工具读取。
3. **用户 shell rc**(`~/.profile` 等):对所有项目生效,经 login shell 白名单/snapshot 机制部分进入(自定义变量名需匹配 `runtimeCommandEnv.ts:48-110` 白名单才能进入运行时;Bash 工具侧靠 shell init snapshot source rc)。
4. **做不到的**:在 config.json(任何层级)写顶层 `env` 让 Bash 工具/子代理/Hook 拿到任意变量;AGENTS.md 声明 env;项目级改 `network`/`storage`/`permission` 等运行参数。

## 8. 桌面 setting.json 与 config.json 的关系补充

桌面设置页(`packages/ui/src/settings/settingsPageConfig.ts:54-157`)同时写两类文件:App 偏好写 `~/.zcode/v2/setting.json`,扩展资源(MCP/Hooks/Skills/Plugins)写 config.json 并按是否带 workspace 上下文决定用户级/项目级。setting.json 中与网络相关的 `httpProxy`/`httpProxyNoProxy`/`httpProxyCaCertPath` 会经 `packages/services/src/runtime-tools/agentProxyEnv.ts:20-51` 注入 agent 进程 env(`HTTP_PROXY` 派生/`NODE_EXTRA_CA_CERTS`),但同样不是任意 env 通道。
