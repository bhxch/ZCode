# ZCode 系统提示词组装与 AGENTS.md 加载机制调研

> 调研日期：2026-11-06。基于 main 分支（16dc79ba）源码。
> 核心问题：① 组装系统提示词时加载哪些文件；② 用户级 AGENTS.md 是否支持 `@` 符号引入其他文件内容。

## TL;DR

1. **加载的指令文件只有 AGENTS.md 一种**，两个层级同时生效：用户级 `~/.zcode/AGENTS.md` + 项目级（从 cwd 逐级向上、止于 git root 的第一份 AGENTS.md）。CLAUDE.md 不被加载。
2. **不支持 `@path/to/file` 引入语法**。指令文件按原始字节直读（前 100KB），从读取到注入全程无任何内容级解析/展开；文件里写 `@xxx` 只会作为普通文本原样进入提示词。
3. 系统提示词分两条通道：固定的 **system 角色消息**（身份、行为规范、memory 使用说明等）+ 每次请求前动态宣告的 **`<system-reminder>` meta_user 附件**（AGENTS.md、MEMORY.md、环境、git、skills、日期等）。AGENTS.md 属于后者，严格说"不是 system prompt 的一部分"，而是每轮请求的上下文公告。

---

## 1. 组装链路总览

```
AgentRuntime
 └─ ensureContextInitialized()                      core/src/runtime/methods/context.ts:43-100
     ├─ contextSourcePort.resolveContextSources()   环境探测 + AGENTS.md 发现读取（见 §3）
     ├─ discoverSkillsForContext()                  技能发现
     ├─ loadProjectMemoryRoot() / loadProjectMemoryIndexContent()   memoryRoot + MEMORY.md
     ├─ createContextBuilderFromSnapshot()          context.ts:122-176，组装 ContextBuilderConfig
     │    （userInstructions / memoryRoot / memoryIndexContent / skills /
     │      customSystemPrompt / workflowActor / outputStyle / guidanceToolNames）
     └─ initializeMessageHistoryFromContext()       contextBuilder.build() → systemMessages 写入历史前缀
```

核心组装器：`apps/zcode-cli/packages/core/src/context/builder.ts` 的 `ContextBuilder.build()`（82-229 行）。每个 section 带 `injectionTarget`（`system` 或 `meta_user`）与 `cacheHint`（`stable`/`dynamic`），最终：

- **system 通道**（`assembleSystemMessages`，builder.ts:236-282）：产出 3 条 system 角色消息——① CLI prefix（"You are ZCode, an interactive agent"）单独一条；② stable 正文一条（带 ephemeral cacheControl）；③ dynamic 正文一条。
- **meta_user 通道**（`buildMetaUserAttachments`）：以 `<system-reminder>` 附件形态在**每次模型请求前**由运行时动态宣告，不进 system prompt。宣告入口在 `core/src/runtime/methods/context-refresh.ts`：
  - `appendRequestAnnouncements`（234 行）：instructions(context_prefix，含 AGENTS.md + MEMORY.md) → session_context(git) → date，每轮请求前去重后宣告；
  - `appendRuntimeContextAnnouncements`（79 行）：environment → model → output_style_instructions；
  - `appendRuntimeSkillsAnnouncement`（221 行）：skills_listing。

历史前缀只保留 system 消息（`context-history-entries.ts:5-9`）；指令文件、Skill、Git 等由请求前公告宣告。冷恢复/compaction 后由 `refreshRestoredAnnouncements` 原位更新公告（context-refresh.ts:195-213）。

## 2. 加载的全部内容清单（主 agent，按 build() push 顺序）

| # | Section | 来源 | 通道 | cacheHint |
|---|---------|------|------|-----------|
| 1 | CLI Prefix | sections/cli-prefix.ts:8 | system | stable |
| 2 | Agent Identity（默认身份 + 安全行 + `# Harness`）或 Custom System Prompt 或 Workflow Actor Identity | sections/identity.ts:54；builder.ts:109-123；sections/workflow-actor.ts:73 | system | stable |
| 3 | Desktop Context（仅 `presentationSurface === "zcode_desktop"`） | sections/desktop.ts:4 | system | stable |
| 4 | Dynamic Behavior（文风/代词/行动谨慎） | dynamic-sections.ts:58 | system | dynamic |
| 5 | Session Guidance（Skill 触发行） | dynamic-sections.ts:21 | system | dynamic |
| 6 | Memory 使用说明（需 memoryRoot 存在） | sections/memory.ts:9 | system | dynamic |
| 7 | Environment Info（cwd/platform/shell/OS/附加目录/不可信下载告示） | sections/env-info.ts:140 | meta_user | dynamic |
| 8 | Model Identity（"You are powered by the model named ..."） | sections/env-info.ts:254 | meta_user | dynamic |
| 9 | Context Management（compaction 指引） | dynamic-sections.ts:66 | system | dynamic |
| 10 | Session Context（git 快照：branch/status/recent commits） | sections/env-info.ts:158 | meta_user | dynamic |
| 11 | Skills 清单（预算 20 000 字符，超限降级为名字+路径） | sections/skills.ts:17 | meta_user | dynamic |
| 12 | **Request User Context（AGENTS.md + MEMORY.md 索引）** | sections/request-user-context.ts:15 | meta_user | dynamic |
| 13 | Current Date | sections/current-date.ts:8 | meta_user | dynamic |
| 14 | customSections（`addSection` 扩展点，当前无内置调用方） | builder.ts:65-77 | - | - |

特殊分支：

- **customSystemPrompt**（设置页/运行时整段替换）：跳过全部动态 system 段（desktop/dynamic behavior/session guidance/memory/context management/git），仅保留环境与模型 meta_user 公告（builder.ts:137-179）。
- **workflowActor** 与 customSystemPrompt 互斥，同时存在直接抛错（builder.ts:94-98）。
- Git 段对 subagent 与 custom prompt 一律不宣告（context-refresh.ts:397）。
- `ProjectContext`（语言/包管理器/scripts）在 adapter 中探测并存入 snapshot（adapters/src/context/index.ts:279-347），但 ContextBuilder 目前未消费——疑似留作后续的占位（推测）。

## 3. AGENTS.md 的发现与加载（重点）

实现全部在 `apps/zcode-cli/packages/adapters/src/context/index.ts` 的 `NodeContextSourceAdapter`：

- `DEFAULT_PRIORITY_FILES = ["AGENTS.md"]`（26 行）——优先级文件清单，**CLAUDE.md 不在其中**；
- `DEFAULT_MAX_BYTES = 100 * 1024`（27 行）——单文件读取上限 100KB，超出静默截断并标 `truncated`（渲染时追加 `[File truncated: AGENTS.md]`）。

### 3.1 两个层级

1. **用户级**：`findDefaultUserInstructionFile`（258-272 行）→ `<home>/.zcode/AGENTS.md`。home 解析顺序 `HOME || USERPROFILE || homedir()`（274-277 行）。仅当 priorityFiles 包含 `AGENTS.md` 时才查（当前恒真）。
2. **项目级**：`findInstructionFile`（233-256 行）——从 `workingDirectory` 开始**逐级向上**（dirname），每级按 priorityFiles 顺序检查，**命中第一个文件即返回，不合并多层**；终止于 projectRoot（最近的含 `.git` 目录，`findProjectRoot` → git-snapshot.ts 的 `findGitRoot`）或文件系统根。即子目录里的 AGENTS.md 会遮蔽仓库根的（nested-first），但一次会话只取一份 workspace 指令，**不向下递归收集**。

### 3.2 合并

`resolveUserInstructions`（123-158 行）把 user 级 + workspace 级候选按路径去重后都读取，`mergeInstructionSources`（216-231 行）按 user→workspace 顺序 `join("\n\n")`；元信息 `filePath` 取 workspace 那份（primary）。

**结论：用户级与项目级 AGENTS.md 同时生效、顺序拼接（user 在前）。**

### 3.3 注入格式

`sections/request-user-context.ts`（64-133 行）：

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.

Contents of <userPath> (user's private global instructions for all projects):

<用户级内容>

Contents of <projectPath> (project instructions, checked into the codebase):

<项目级内容>

Contents of <memoryRoot>/MEMORY.md (user's auto-memory, persists across conversations):

<MEMORY.md 索引>
```

实测验证：本调研会话自身的系统提示词中即可观察到上述两段 AGENTS.md（用户级 + 项目级）以该精确格式注入，与代码渲染逻辑吻合。

### 3.4 子代理继承

`runtime/methods/subagent.ts:135-138`：profile 的 `injectAgentsMd !== false` 时，把父会话已解析的 `userInstructions` 传给子代理，由 `SubagentContextBuilder`（core/src/subagent/context-builder.ts:112-117）同样渲染进 meta_user。内置 Explore profile 为 `injectAgentsMd: false`，general-purpose 为 true（core/src/subagent/profile.ts:31, 81）。

### 3.5 priorityFiles 扩展点与 CLAUDE.md

- 类型定义在 `apps/zcode-cli/packages/contracts/src/interfaces/context-source.port.ts:38-43`（`UserInstructionsOptions.priorityFiles?: string[]`），adapter 完整支持；
- 但全仓无任何调用方传值（`bootstrap/src/app/runtime-config.ts:63, 154-157` 只补 workingDirectory）→ 现状恒为 `["AGENTS.md"]`。是可扩展但无人使用的挂点。
- 全部 `CLAUDE.md` 命中仅为 memory 提示词文案措辞与测试用例，无加载逻辑。

## 4. `@` 符号文件引入能力：不支持

**结论：AGENTS.md（及其他一切用户指令文件）不支持 `@path/to/file` 语法引入其他文件内容。**

代码证据：

- 指令文件内容由 `readFirstTextBytes` 读取（adapters/src/context/index.ts:160-163）：`readFile` → `subarray(0, maxBytes).toString("utf8")`，即**原始字节直读**；
- 读取后无任何内容级解析、正则匹配或展开：`readInstructionSource` 只做 stat/truncate 标注，`mergeInstructionSources` 只做字符串拼接，`request-user-context.ts` 渲染时只做 trim 与截断标注；
- 搜索过且无命中的关键词：`resolveImport`、`expandImport`、`at-import`、`importFile`、`atImport`、`@import`、`expandAtMentions`、`inlineFile`、`transclude`、`startsWith("@")`、字面量 `"@"`（context/memory/session-context 目录）。仓库级 `resolveImport|expandImport|importFile|atImport` 命中的 8 个文件均属浏览器导入/会话分享导入/Claude 历史导入，与指令文件无关。

因此 AGENTS.md 里写 `@~/.zcode/rules/style.md` 之类只会作为普通文本原样进入提示词，模型会看到这行字但不会自动读该文件（模型可能主动用 Read 工具去读，但那是模型行为而非机制保证）。

### 易混淆的相邻能力（均非指令文件 import）

- **TUI composer 的 @ 文件提及**：`apps/zcode-cli/packages/tui/src/app-file-mentions.ts`（`FILE_MENTION_SIGIL = "@"`）——用户在输入框里 @ 文件，作为用户消息附件（`prompt_attachment` system-reminder）注入，属**用户输入侧**，与 AGENTS.md 无关。
- TS 源码中 `@zcode/...` 是包引用，非内容引用。

## 5. 其他注入源

- **Memory**：`buildMemorySection`（"# Memory" 使用说明，system/dynamic，builder.ts:159-167）需 memoryRoot 存在；开关 = `features.memory`（默认 true）∧ `memory.use`（默认 true）∧ taskType 为主线程类型（runtime/helpers/project-memory.ts:4-17）。根目录 `<cliStorageRoot>/memories/projects/<slug>-<hash>/memory`。MEMORY.md 索引内容与 AGENTS.md 同在 context_prefix 公告里（§3.3）。`semantic-recall` 分支经 `relevant_memory` reminder 注入召回内容（runtime/helpers/project-memory-recall.ts:104-134）。
- **Skills 清单**：`buildSkillsSection`（meta_user `skills_listing`），条件为 skills 存在且 Skill 工具已注册（builder.ts:185, 231-234）；预算 `skills.metadataBudget` 默认 20 000 字符。
- **Subagent system prompt**：`SubagentContextBuilder`（core/src/subagent/context-builder.ts:29-75）组装顺序：CLI Prefix → agentPrompt（profile systemPrompt + persistent memory prompt 拼接，runtime/methods/subagent.ts:166-182）→ 启动方消息声明（SUBAGENT_LAUNCHER_MESSAGES_NOTE，点明 AGENTS.md 内容不代表用户授权）→ Notes → 环境/模型 → AGENTS.md（若注入）→ 日期 → Skills。产出单条合并 system 消息。
- **Persistent agent memory**：`buildPersistentAgentMemoryPrompt`（core/src/subagent/persistent-memory-prompt.ts:150）由 `loadPersistentAgentMemory` 加载，根目录 `storageRoot/agent-memory/<agent>` 或 workspace `.zcode/agent-memory[-local]/<agent>`。
- **Output Style**：设置后 Identity 开场句替换 + `output_style_instructions` 公告（context-refresh.ts:104-112）。

## 6. 相关配置项

配置定义：`apps/zcode-cli/packages/contracts/src/config/index.ts`（221-258, 290-340）；schema：`adapters/src/config/schema.ts`；装配：`bootstrap/src/app/runtime-config.ts`（159, 190-199）。

| 配置 | 默认 | 作用 |
|------|------|------|
| `features.memory` | true | 关掉则无 memoryRoot、无 Memory 段、无 MEMORY.md 注入 |
| `memory.use` | true | memoryRoot 的另一前提 |
| `memory.extractionEnabled` | true | Main turn 后自动 Extraction 调度 |
| `skills.enabled` / `includeInstructions` / `metadataBudget` / `roots` | true/true/20000/[] | 技能发现与清单预算 |
| runtime `systemPrompt` | - | 整段替换 stable system body 并跳过动态 system 段 |
| runtime `workflowActor` | - | 工作流子代理身份路径（与 systemPrompt 互斥） |

- **没有"禁用 AGENTS.md 加载"的配置项**（唯一门控是 subagent profile 的 `injectAgentsMd`）。协议侧 `startupPreferences.memoryEnabled` 只影响 memory。
- 字节上限双轨：桌面端设置页写入侧用 `packages/shared/src/userInstructions.ts` 的 `USER_INSTRUCTIONS_MAX_BYTES`（100KB，超限拒绝写入）；CLI 读取侧用 adapter 自己的 `DEFAULT_MAX_BYTES`（超限静默截断）。两侧靠该常量文件头部注释约定同步，无代码级共享。

## 7. 关键文件索引

| 文件 | 职责 |
|------|------|
| `apps/zcode-cli/packages/adapters/src/context/index.ts` | AGENTS.md 发现/读取/合并/截断；环境与项目上下文探测 |
| `apps/zcode-cli/packages/core/src/context/builder.ts` | ContextBuilder：section 组装、system/meta_user 分流 |
| `apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts` | AGENTS.md + MEMORY.md 渲染格式 |
| `apps/zcode-cli/packages/core/src/runtime/methods/context.ts` | ensureContextInitialized 入口、ContextBuilderConfig 装配 |
| `apps/zcode-cli/packages/core/src/runtime/methods/context-refresh.ts` | 每请求前动态宣告（instructions/git/date/env/model/skills） |
| `apps/zcode-cli/packages/core/src/subagent/context-builder.ts` + `profile.ts` | 子代理提示词组装与 injectAgentsMd 门控 |
| `apps/zcode-cli/packages/contracts/src/interfaces/context-source.port.ts` | priorityFiles/maxBytes 扩展点类型 |
| `packages/shared/src/userInstructions.ts` | 桌面端写入侧 100KB 上限常量 |
| `packages/services/src/setting/settingService.ts` + `paths.ts:73` | 桌面端「个性化」读写 ~/.zcode/AGENTS.md |
