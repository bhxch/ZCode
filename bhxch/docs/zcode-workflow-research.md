# ZCode Workflow 功能调研

- 调研日期：2026-10-11
- 基线版本：`main @ aac47556`（feat: update to v3.15.1）
- 调研方式：源码走读（`apps/zcode-cli/packages/*`）+ 官方 spec（`docs/dynamic-workflow/`）+ 本机安装包与插件缓存对照
- 路径约定：下文相对路径均相对仓库根 `E:\repos\github\ZCode`

## TL;DR

| 问题 | 结论 |
| --- | --- |
| 有没有内置工作流？ | **没有预置的可运行工作流**（全仓库无 `.dwf.ts` 模板）。内置的只有入口与教学材料：`/workflow` 斜杠命令、`dynamic-workflows` 内置技能及其 5 个示例脚本。 |
| 动态工作流用什么语言编排？ | **TypeScript（ES2022 严格模式）**。虚拟 TS 编译器做类型检查，降级（lowering）后丢进 Node 子进程 + `vm.createContext` 独立 realm 沙箱执行，无 `process`/`require`/`fetch`，禁 `Date.now`/`Math.random`，保证可确定性重放。 |
| 有没有 Claude Code 那样的 deep research 内置工作流？ | **没有**。全仓库 deep-research 关键词仅 2 处非功能命中（UI 空态文案、超时注释）。研究能力走官方市场插件（如 `write-research`），但那是 agents/skills/MCP 组合，不是工作流。 |
| 能不能做插件提供预置工作流？ | **manifest 原生不支持** `workflows` 贡献点，但有三条可行路径：① 插件携带脚本 + 技能教模型用 `CreateWorkflow` 的 `path` 源直接运行（已有 Claude 插件先例，零源码改动）；② 引导落盘为 saved workflow；③ 改源码给 manifest 增加 `workflows` 字段（代码里已预留 `BUILTIN_WORKFLOW_ALLOWLIST` 挂点）。 |

---

## 1. 全景：两套同名不同代的工作流

代码里有两个都叫 "workflow" 的功能，调研时最容易混淆：

| | Legacy "Expert Workflow" | 动态工作流（Dynamic Workflow） |
| --- | --- | --- |
| 模型工具 | `Workflow` 工具、`/expert` 命令 | `CreateWorkflow` 等 10 个工具、`/workflow` 命令 |
| 定义方式 | 固定的 `WorkflowDefinition`（内置一个 `builtInExpertWorkflowDefinition`） | 模型现场编写的 TypeScript 脚本 |
| 实现位置 | `apps/zcode-cli/packages/bootstrap/src/app/workflow-methods.ts`、`core/src/workflow/expert.ts` | `apps/zcode-cli/packages/dynamic-workflow*`（专用包） |
| 状态 | 旧系统，仍在代码中，但内置 `/workflow` 命令正文明确写 "Do not use the legacy `Workflow` tool" | 当前主推 |

官方 spec 文档在 `docs/dynamic-workflow/`（authoring / concurrency / launch / presentation / transcript-and-notifications 五篇），`docs/dynamic-workflow/authoring.md` 是完整的创作规范。

### 动态工作流的 10 个模型工具

schema 契约在 `apps/zcode-cli/packages/contracts/src/tools/`，handler 在 `apps/zcode-cli/packages/core/src/tool/handlers/`：

| 工具 | 用途 |
| --- | --- |
| `CreateWorkflow` | 提交脚本：typecheck → 用户确认（展示脚本与命令集）→ 后台运行 |
| `AmendWorkflow` | 修改任意状态（含运行中）的既有 run 的脚本并重跑，已完成步骤从 journal 缓存复用 |
| `EvalWorkflowSnippet` | 同步试跑脚本片段（同一编译器/沙箱/world 读取路径），不落盘 |
| `SaveWorkflow` / `ListSavedWorkflows` | 把脚本保存为命名模板 / 枚举 |
| `ListWorkflowRuns` / `GetWorkflowRun` | 枚举项目 runs / 查看单个 run 详情 |
| `ResumeWorkflowRun` | 恢复被停止（user/model/interrupt/provider）的 run |
| `ResolveWorkflowQuestion` | 回答子代理升级上来的阻塞提问 |

**技能门**：四个创作工具（CreateWorkflow/AmendWorkflow/SaveWorkflow/EvalWorkflowSnippet）要求会话先加载 `dynamic-workflows` 技能才肯运行（`contracts/src/tools/create-workflow.ts` 的 `DYNAMIC_WORKFLOW_SKILL_NAME`，core 侧 `workflow-skill-gate.ts`）。运行已保存的工作流是唯一不要求加载技能的调用。

**防止嵌套**：子代理内禁用 CreateWorkflow/AmendWorkflow/SaveWorkflow/ResumeWorkflowRun/ResolveWorkflowQuestion（`core/src/tool/handlers/tool-allowlist.ts`），避免确认窗死锁。

---

## 2. 编排语言与执行机制

### 2.1 语言：TypeScript

脚本就是一段 TypeScript（ES2022），在脚本内定义普通 `interface` 描述子代理返回类型，用 `agent()`/`ask<T>()` 编排。来源三种：inline `script`、磁盘 `path`、命名 `saved`。

### 2.2 编译（`packages/dynamic-workflow/src/compiler/compile.ts`）

- 脚本被包进 `async function __workflowScript__()`，用 TypeScript Compiler API 在**全虚拟 CompilerHost**（内存 Map、`noEmit`、`types: []`、内嵌 `lib.es2022.d.ts`）中 typecheck。
- 没有 `@types`，所以 `process`/`fetch`/`require` 在编译期就过不了——"纯度契约从编译期开始"。
- 同一个 `ts.Program` 同时产出四样东西：诊断、参数 schema 合成、站点图提取（模型用于把脚本画成阶段图）、lowering。不变式是"编译恰好一次"。
- 刻意关闭 `noUncheckedIndexedAccess`（注释说明：模型写 TS 的习惯导致 TS2532 占编译失败的 96%）。

### 2.3 降级与沙箱（`lowering/lower.ts`、`packages/dynamic-workflow-runtime`）

- Lowering 做类型擦除，并把 facade 调用打桩为 `__host.*`：`agent()` → `__host.createActor`、`ask()` → `__host.ask`、`files/git` → `__host.worldRead`、`report()` → `__host.report`、`artifact.file()` → `__host.publishArtifact`、`phase()` → `__host.enterPhase`。
- 执行是**子进程 + vm 双层隔离**：
  1. lowered 代码写进 `<cwd>/.zcode/workflow-runs/<runId>.mjs`（绕开 Windows 32k 命令行上限），`spawn(process.execPath, ['--max-old-space-size=256', entry])`；
  2. 子进程内 `vm.createContext` 建独立 realm：只有 ES intrinsics + 注入的 `__host`，无 `process`/`require`/`Buffer`/`fetch`，动态 `import()` 直接抛错；
  3. `Date.now()`、无参 `new Date()`、`Math.random()` 在运行期被替换为抛错（编译期诊断之外再上一道保险），保证 journal 重放确定。
- 父子进程用 **NDJSON over stdio** 桥接；脚本本身没有网络原语，对世界的一切读写都经 `__host` 回父进程执行。

### 2.4 脚本可用的 facade API

完整声明在 `packages/dynamic-workflow/src/facade/dts.ts`，规范在 `docs/dynamic-workflow/authoring.md`：

| API | 说明 |
| --- | --- |
| `agent(name?, persona?)` → `Agent.ask<T>(instructions)` | 创建子代理（每调用一个新上下文）；name 是身份、run 内必须唯一，也是 AmendWorkflow 缓存复用的 key；同一 agent 的多个 ask FIFO 串行 |
| `log(msg)` / `report(item, tag?)` | 进度叙述 / 中间结果上报（run 失败也随完成通知送达） |
| `phase("...")` | 阶段标记（用户看到的进度图按它渲染），须为编译期字面量，每阶段须含至少一个 ask 或 world.run |
| `files.glob/read/grep`、`git.changedFiles/diff/status/log` | journaled 只读世界观察（grep 超宽会整体拒绝而非截断） |
| `world.run(cmd, args, {timeoutMs})` | 确定性门：命令名必须是编译期字面量（用户在确认窗审批命令集），非零退出是返回值不是异常 |
| `artifact.file/markdown/chart/table/metrics/board` | 产物发布：内容型（可 catch 拒绝后修复重发）与预置型（配 `report(item, id)` 喂数） |
| `args: Readonly<Record<string, unknown>>` | saved workflow 的声明式实参（string/number/boolean/json 四型） |
| `channel` / `future`、`hole<T>()` | 流式通道与编译期"占位洞"（spec 有专章，高级特性） |

Snippet（EvalWorkflowSnippet）的 facade 刻意不含 `agent/report/artifact/phase`，用 TS2304 在编译期拒绝——只能试跑"非编排"部分。

### 2.5 子代理调度

- 并发上界 = `max(1, min(16, availableParallelism() - 2))`（`bootstrap/src/app/workflow-concurrency-ceiling.ts`）；`max_concurrency` 只能压低不能抬高。
- 进程级 **AIMD 自适应并发**（`dynamic-workflow/src/engine/concurrency.ts`）：429 → cap×0.75，连续 4 次成功 +1，空闲 5 分钟回天花板。
- 每个 actor 是一个**持久的 child `AgentRuntime`**（每次 ask 执行一个 turn，会话落库并建 task link）；类型化 ask 经注入的 `submit_result` 工具回传，turn 结束未提交会被 nudge 重驱。
- 模型优先级：run 的 `subagent_model` > resume pin > 父会话当前模型（`workflow-actor-model.ts`）。
- 子代理可用 `escalate` 工具向用户提问（每 ask 限 3 次），答案经主代理的 `ResolveWorkflowQuestion` 送回。

### 2.6 存储

| 对象 | 位置 | 负责代码 |
| --- | --- | --- |
| saved workflow 定义 | 项目档 `<cwd>/.zcode/workflows/*.dwf.ts`、全局档 `~/.zcode/workflows/`，项目档遮蔽全局档（first-wins） | `core/src/tool/handlers/saved-workflows/store.ts`；常量在 `contracts/src/tools/saved-workflow.ts` |
| 脚本草稿 | `<cwd>/.zcode/workflow-drafts/`（机器自有、自带 gitignore） | 同上 |
| run 记录 | SQLite 会话库表 `dwf_run` / `dwf_actor` / `dwf_node` / `dwf_event`（journal） | `adapters/src/storage/session-store/migrations.ts` |
| GUI 面 | zcode-protocol v4：`workflows/list/get/updateMeta/delete/move/runs` | `bootstrap/src/zcode-protocol/saved-workflows.ts`（UI 设置页在 `packages/ui/src/settings/saved-workflows/`） |

saved workflow 文件格式：`.dwf.ts` 扩展名 + `/* zcode-workflow` 哨兵块注释作 frontmatter（`description`/`whenToUse`/`args`，`.strict()` 校验，拼错键会显式报 invalid）。名字正则 `^[A-Za-z0-9_.-]+$`——这条正则就是路径穿越防线。journal 重放同时支撑崩溃恢复（ResumeWorkflowRun）与 AmendWorkflow 的缓存复用。

---

## 3. 是否有内置工作流？

**没有预置的可运行工作流。** 具体证据：

1. 仓库内不存在任何 `.dwf.ts` 文件；本机 `~/.zcode/workflows/` 与项目 `.zcode/workflows/` 也都不存在。
2. `bootstrap/src/app/script-workflow-tool-port.ts:26` 有一个 `BUILTIN_WORKFLOW_ALLOWLIST = new Map<string, string>()`——**空 Map**，是给内置工作流预留的挂点（名字 → 路径的映射位），目前没有条目。
3. 产品内置的是三类"入口与教学"，不是可运行模板：
   - **内置 `/workflow` 斜杠命令**（`bootstrap/src/builtin-workflow-command.ts`）：prompt 型命令，正文让模型加载 `dynamic-workflows` 技能、设计子代理拓扑后调 `CreateWorkflow`。它原本是 zcode-guide 插件里的 `commands/workflow.md`，2026-09-19 因插件卸载导致入口消失的事故而编译进 CLI，命令名进保留字表。
   - **内置技能 `dynamic-workflows`**（`apps/zcode-cli/packages/bundled-skills/skills/`，安装包镜像在 `resources/glm/packages/bundled-skills/`）：SKILL.md（约 1500 行的完整创作契约）+ examples.md + patterns.md，是安装包里唯一的 bundled 技能，在插件系统之外（`source: "bundled"`，不可卸载）。
   - **技能文档内嵌 5 个示例脚本**：对抗式实现↔验证环、flaky 测试分诊、调用点迁移、prover 门控环、变更文件评审——都是文档代码，非安装的模板。
4. Legacy Expert Workflow 有一个 `builtInExpertWorkflowDefinition`（`core/src/workflow/expert.ts`），但那是旧系统的固定定义，不属于动态工作流体系。

---

## 4. 有没有 deep research 类内置工作流？

**没有。** 全仓库搜索 `deep-research`/`DeepResearch`/`deep research` 仅两处非功能性命中：

- `packages/ui/src/i18n/locales/en-US.ts:5139`：全局工作流空态文案 "No global workflows yet. Good for flows that depend on no particular project, such as deep research." ——只是把 deep research 当示例场景提了一句；
- `core/src/tool/executor/timeout.ts:15`：注释里引用 deep-research 实例的限流教训。

没有任何以 research 命名的工作流模板或内置编排。

**最接近的现成能力**是官方市场插件 `write-research`（本机缓存 `~/.zcode/cli/plugins/cache/zcode-plugins-official/write-research/0.1.1/`）：捆绑 agents + commands + skills + 8 个 http 型 MCP server（`auth.type: "zcode_official"`，`requiresPaidPlan: true`），做深度研报，但它是"技能+子代理+检索工具"的组合，不是可参数化重跑的工作流。

**与 Claude Code 的对照**：Claude Code 的 `Workflow` 工具脚本是 JS `export const meta = {...}` 格式（本机 claude-security 插件带 `workflows/scan.js` 即此格式）。ZCode 的插件加载器兼容 Claude 插件 manifest（`.claude-plugin/plugin.json`），但动态工作流是自研的 TypeScript/journal 体系，两者脚本格式不互通。

---

## 5. 能否制作提供预置工作流的插件？

### 现状：manifest 不支持 workflows 贡献点

插件 manifest（`contracts/src/plugins/index.ts:141-161` 的 `PluginManifest`）支持的能力字段：`skills`、`commands`、`agents`（子代理）、`hooks`、`mcpServers`、`userConfig`、`dependencies`；`channels/lspServers/outputStyles/settings` 仅识别并警告"本 runtime 不支持"。**没有 `workflows` 字段**——`PluginLoadOutcome` 只有 `commandRoots/hooks/mcpServers/skillRoots` 四类组件，全部插件加载/同步代码中 grep "workflow" 零命中。

saved workflow 的查找根硬编码为两档（`core/src/tool/handlers/saved-workflows/store.ts`），**没有 plugin 档**；名字正则不允许冒号，`插件名:工作流名` 的命名空间形态也走不通。

### 三条可行路径

| 路径 | 做法 | 代价 | 效果 |
| --- | --- | --- | --- |
| **A. 脚本随插件分发（推荐，零源码改动）** | 插件目录放 `workflows/<name>.dwf.ts`（带 `/* zcode-workflow` frontmatter 声明 args），再带一个 SKILL.md 教模型何时、如何以 `CreateWorkflow` 的 `path` 源运行 `${CLAUDE_PLUGIN_ROOT}/workflows/<name>.dwf.ts` | 纯插件，不用改 ZCode | 每次运行走完整确认窗（用户能看到脚本与命令集），无参数 schema 校验（path 源不读 frontmatter args） |
| **B. 引导落盘为 saved workflow** | 技能或 SessionStart hook 把插件内脚本复制/引导用户保存到 `.zcode/workflows/`（或调 `SaveWorkflow`） | 纯插件 | 获得 saved 源的完整能力：args 校验、`saved: { name }` 调用、UI 设置页管理、项目遮蔽全局 |
| **C. 改源码加贡献点** | `PluginManifest` 增加 `workflows` 字段 → `adapters/src/plugins/index.ts` loader 产出新组件根 → 扩展 `savedWorkflowRoots`/`resolveNamedWorkflowPath` 增加 plugin 档 | 需要动 contracts/adapters/core 三层 + 测试 | 插件安装即得预置工作流，`BUILTIN_WORKFLOW_ALLOWLIST`（现有空 Map）正是预留的内置白名单挂点 |

**路径 A 的关键依据**：`CreateWorkflow` 的 `path` 源接受磁盘绝对路径，`core/src/tool/handlers/workflow-path-source.ts` 第 49-52 行注释明说"当前不硬拦工作区之外的路径"——插件缓存目录里的脚本可以直接运行。这个形态已有先例：claude-security 插件（Claude 格式）就是靠 SKILL.md 指导模型按路径执行插件内 `workflows/scan.js`。

**制作插件的工程入口**：用官方 `plugin-creator` 插件（`.zcode-plugin/plugin.json` + 本地市场 `<workspace>/plugins/marketplace.json` → UI "Plugin Marketplace → Add" 安装），脚手架选项只有 `--with-skills/--with-mcp/--with-hooks/--with-commands`，没有 workflows——印证了贡献点缺失，走路径 A/B 时随包分发的主体是 skills。

---

## 6. 关键源码索引

| 主题 | 文件 |
| --- | --- |
| 官方 spec | `docs/dynamic-workflow/authoring.md`（创作规范）、`concurrency.md`、`launch.md`、`presentation.md`、`transcript-and-notifications.md` |
| 工具 schema | `apps/zcode-cli/packages/contracts/src/tools/create-workflow.ts`、`saved-workflow.ts` 等 |
| 编译/降级/facade | `apps/zcode-cli/packages/dynamic-workflow/src/{compiler/compile.ts, lowering/lower.ts, facade/dts.ts, engine/}` |
| 沙箱运行时 | `apps/zcode-cli/packages/dynamic-workflow-runtime/src/harness.ts`、`child-source.ts` |
| 宿主服务 | `apps/zcode-cli/packages/bootstrap/src/app/dynamic-workflow-run-service.ts`、`workflow-driver.ts`、`workflow-world-read.ts`、`script-workflow-tool-port.ts`（含 `BUILTIN_WORKFLOW_ALLOWLIST`） |
| saved workflow 存储 | `apps/zcode-cli/packages/core/src/tool/handlers/saved-workflows/store.ts` |
| 插件系统 | `apps/zcode-cli/packages/adapters/src/plugins/index.ts`（manifest 解析）、`contracts/src/plugins/index.ts`（类型）、`core/src/tool/handlers/workflow-path-source.ts`（path 源不拦外部路径） |
| 内置命令与技能 | `apps/zcode-cli/packages/bootstrap/src/builtin-workflow-command.ts`、`apps/zcode-cli/packages/bundled-skills/skills/dynamic-workflows/` |
