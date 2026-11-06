# ZCode Git Worktree 支持调研

> 调研日期：2026-11-06。基于 main 分支源码（HEAD 16dc79ba），所有结论附 `file:line`，行号以当日检出代码为准。
> 三个核心问题：① 本工具有无 worktree 相关功能/工具；② 要求"创建并进入 worktree"后，插件/MCP 等用哪个项目的；③ Bash 工具 cd 后目录是否保持。

## 结论速览

| 问题 | 结论 |
|---|---|
| 有无 worktree 原生管理功能 | **没有**。无 `/worktree` 命令、无 EnterWorktree/ExitWorktree 工具（仅排序白名单字符串）、无 UI 入口。但"感知类"兼容功能完善（识别、审批、只读白名单、checkpoint 兼容等） |
| workflow `isolation: 'worktree'` | **未实现**。schema 接受但运行时直接抛错；注意工具描述以已实现口吻宣称，与实现不符 |
| cd 进 worktree 后插件/MCP 归属 | **仍用主仓库（会话启动工作区）的**。MCP、插件、hooks、AGENTS.md、skill 清单、斜杠命令全部锚定会话启动时的工作区，启动后冻结，不跟随 cd。唯一例外：Skill 工具加载正文时跟随当前 cwd（见 §3.2） |
| Bash cd 后目录保持吗 | **工作区内保持，工作区外重置**。无持久 shell 进程；命令成功后捕获 pwd，在 workspaceRoot 边界内则写回保持，出界则重置回 workspaceRoot 并在 stderr 提示 |

---

## 1. Worktree 功能与工具全景

### 1.1 没有原生"创建/进入/切换 worktree"入口

- **无工具**：`EnterWorktree`/`ExitWorktree` 在全仓库非文档代码中只出现于 `apps/zcode-cli/packages/core/src/tool/provider-visible-order.ts:13,15` 的工具名排序白名单（决定 provider 可见工具列表的排序），没有任何 handler、schema 或注册代码。设计文档明确声明不注册：`docs/plan/trajectory/embedded-search-provider-visible.md:74`（"ZCode 暂不新增 `Cron*`、`EnterWorktree`、`ExitWorktree`……默认注册能力"）。
- **无命令**：内置/项目斜杠命令发现里没有任何 worktree 命令。
- **无 UI 入口**：packages/ui、packages/desktop、packages/web 无 worktree 菜单/按钮/标识；services 层也没有任何 `git worktree add` 子进程调用。

### 1.2 workflow 的 `isolation: 'worktree'` —— 未实现，且描述与实现不符

- Schema 接受该值：`apps/zcode-cli/packages/contracts/src/workflow/script.ts:42`（`WorkflowAgentIsolationSchema = z.enum(["worktree"])`）。
- 运行时直接抛错：`apps/zcode-cli/packages/bootstrap/src/app/script-workflow-runtime.ts:347-349`：

```ts
if (input.opts?.isolation === "worktree") {
  throw new Error("workflow agent isolation 'worktree' is not implemented yet.");
}
```

- **注意**：模型可见的 Workflow 工具描述（`apps/zcode-cli/packages/core/src/tool/handlers/workflow-description.ts:3`）以已实现口吻宣称 "`opts.isolation: 'worktree'` runs the agent in a fresh git worktree …"，实际一调用就抛错。让 agent 写 workflow 脚本时要避开这个选项。

### 1.3 已实现的"感知类"功能（不管理，只识别/拦截/兼容）

| 功能 | 位置 | 说明 |
|---|---|---|
| `git worktree list` 只读放行 | `apps/zcode-cli/packages/core/src/tool/handlers/bash-readonly-policy-git-subcommands-history.ts:221-226` | 只读模式下 agent 可查看 worktree 列表 |
| `git worktree remove --force` 审批拦截 | `apps/zcode-cli/packages/core/src/tool/handlers/guarded/git.ts:73-87` | 命中守卫规则 `git-worktree-force-remove`，需用户确认 |
| CLI 侧 `.git` 文件指针识别仓库 | `apps/zcode-cli/packages/adapters/src/context/git-snapshot.ts:33-45` | `.git` 是文件（worktree/submodule 的 gitdir 指针）也算仓库标记，不启动 git 即可判定 |
| 技能/命令发现止步于 worktree 根 | `apps/zcode-cli/packages/adapters/src/skills/roots.ts:74-96`、`adapters/src/commands/roots.ts:74-96`（`findWorktreeRoot`） | 从 cwd 向上找最近 `.git`（文件或目录）即停，不会窜到主仓库去收集 `.agents/skills` 等 |
| 桌面技能发现同款逻辑 | `packages/services/src/skills/skillsService.ts:125-165` | 注释明确"对齐 apps/zcode-cli/packages/adapters/src/skills/roots.ts" |
| linked-worktree 识别（服务端） | `packages/services/src/git/repo/gitCliRepo.ts:789-852`（核心 `:814-837`） | 读 `.git` 文件的 `gitdir:` 指针，命中 `/.git/worktrees/` 判为 `linked-worktree`。**但目前无 UI 消费方**（仅 web 空桩引用），属于挂空能力 |
| Git 元数据监控兼容 worktree | `packages/services/src/git/repo/gitCliRepo.ts:142-165` | 用 `git rev-parse --git-common-dir` 确定监控边界（元数据可能在 worktree 场景位于 repoRoot 之外） |
| 分支被其他 worktree 占用的报错 | `packages/services/src/git/repo/gitCliHelpers.ts:124-132` → `packages/ui/src/git-branch-switcher/display.ts:102-103` → i18n `zh-CN.ts:1900` | 错误码 `branch-in-other-worktree`，UI 有本地化提示"目标分支已在其他 worktree 中被检出"，但不提供跳转动作 |
| checkpoint 兼容 worktree index | `packages/services/src/git/repo/gitCheckpointRepo.ts:271,433` | 用 `rev-parse --git-path index` 兼容 worktree/submodule 的 index 路径 |
| workspace hook 配置向上收集兼容 | `packages/shared/src/workspace-hook-config.ts:316-324,352-361`（`hasWorktreeMarker`） | `.git` 目录**或文件**都算边界标记 |
| Claude 会话导入过滤 | `packages/services/src/session/claude-native/claudeNativeSessionImportRepo.ts:14,21-31,169-234` | 按路径过滤 `~/.claude/worktrees` 下的短生命周期 worktree transcript，不当成可迁移项目 |

命名易混淆项：`packages/ui` 里的 `gitWorktreeChangeSummary` 等指 **working tree**（工作副本的行级变更汇总），与 git linked worktree 无关。

### 1.4 实际可用的两条路径

1. **superpowers 插件的 `using-git-worktrees` 技能**（`apps/zcode-cli/packages/superpowers-plugin/skills/using-git-worktrees/SKILL.md`）：半自动流程——检测已有 worktree → 用户同意 → 优先在项目内 `.worktrees/<分支>` 执行 `git worktree add` → 装依赖 → 基线测试。技能文本会先找"原生工具"（`EnterWorktree`、`/worktree` 命令、`--worktree` flag），ZCode 里都没有，所以走手动 git 分支。
2. **手动 `git worktree add` 后把该目录作为新工作区打开**（侧边栏独立项目条目）。

---

## 2. workspaceRoot：一切锚定的原点

所有扩展资源锚定的"工作区根"在**会话启动时一次性确定**，之后不随 Bash cd 漂移：

- CLI 启动：`apps/zcode-cli/packages/cli/src/run.ts:539-549` 用启动 cwd（或 `--cwd` 参数）确定，`resolveCliCwd`（`cli/src/cwd.ts:11-34`）只做 resolve，**不做 git 根向上查找**。
- App/runtime 创建：`apps/zcode-cli/packages/bootstrap/src/app/create-app.ts:177`（`resolve(options.runtimeConfig?.workingDirectory ?? process.cwd())`）；协议宿主每个会话由客户端传 workspacePath（`bootstrap/src/zcode-protocol/server-operations.ts:3607-3610`）。
- runtime 内两个字段分工（`apps/zcode-cli/packages/core/src/runtime/agent-runtime.ts:346,363` 与 `runtime/methods/context.ts:76-78`）：
  - `workingDirectory`：会随 Bash cd 变化（边界内）；
  - `workspaceRoot`：恒为会话初始工作区，是身份与资源发现的锚。官方注释："Bash cd 之后 workingDirectory 会变化，但 workspaceRoot 仍表示会话初始工作区边界"。
- 身份明确不漂移：`runtime/methods/config.ts:108-116`——`getProjectId` 用 `workspaceRoot`（注释："project identity 不能随工具内 cwd 漂移"），`setWorkingDirectory` 只改 `workingDirectory`（注释："Bash cwd 持久化只应影响当前 runtime 会话，不能改变工作区身份"）。

## 3. 问题二：cd 进 worktree 后，插件/MCP 用哪个项目的？

场景：会话工作区为主仓库 `F:\repo`，agent 执行 `git worktree add` 后 `cd F:\repo\.worktrees\feat-x` 继续干活。

### 3.1 锚定矩阵（全部主仓库，启动后冻结）

| 资源 | 项目级来源（锚定主仓库） | 加载时机 | 跟随 cd？ |
|---|---|---|---|
| 项目配置 | `<ws>/zcode.json`、`<ws>/.zcode/config.json`，从 ws 向上到最近 `.git`（`packages/shared/src/workspace-hook-config.ts:211-225,339-350`） | App 创建一次 | 否 |
| **MCP servers** | 项目配置 `mcp.servers`（user 覆盖 project，`adapters/src/config/config-factory.ts:372-398`）+ `<ws>/.agents/mcp.json` 信任回退（`bootstrap/src/workspace-mcp-trust/index.ts:86-95`） | 启动解析并连接一次，`runtime/methods/mcp.ts:57-97` 有 `mcpInitialized` 门闩；协议宿主注释明确"MCP 是 runtime 启动期配置……一次性写入" | **否** |
| **插件** | 安装全在用户级 `~/.zcode/cli/plugins`（`bootstrap/src/plugins.ts:1032-1054`）；workspace 只存 enabled/options 覆盖；项目配置**不能**引入插件目录（`adapters/src/config/project-config.adapter.ts:132-136`） | App 创建时冻结（`create-app.ts:333-336` 注释："已有 Session 不热加载新 Plugin"），无 watcher | **否** |
| hooks | 项目配置 `hooks` 字段，启动时快照+信任（`config-factory.ts:212-228`） | 启动一次 | 否 |
| AGENTS.md | `~/.zcode/AGENTS.md`（user）+ cwd→git 根链上的 AGENTS.md（`adapters/src/context/index.ts:123-158`） | context 初始化一次，`context.ts:48` 门闩 | 否 |
| skill 清单（系统提示） | `<dir>/.zcode/skills`、`.agents/skills`（cwd→git 根链 + home） | context 初始化一次，cd 后清单不刷新 | 否 |
| 斜杠命令 | `.zcode/commands`、`.agents/commands`，锚定 CLI 启动目录闭包（`cli/src/run.ts:553`）或协议 workspacePath | 列举时执行但目录固定 | 否 |
| 记忆 | 键 = `workspaceIdentity \|\| workspacePath` 的哈希，runtime 构造时固定为 workspaceRoot（`agent-runtime.ts:317-333`） | — | 否（仍读写主仓库的记忆） |

**直接回答：worktree 里干活时，注册的插件、MCP、hooks、AGENTS.md、技能清单全是主仓库（会话启动工作区）的那一套**。worktree 里自带的 `.zcode/config.json`、`.agents/mcp.json`、AGENTS.md 不会被本会话加载——除非把 worktree 目录作为新工作区另开。

### 3.2 唯一例外：Skill 工具加载正文跟随当前 cwd

`apps/zcode-cli/packages/core/src/tool/handlers/skill.ts:55-58`：

```ts
const loaded = await skillPort.loadSkill({
  name: skill,
  workingDirectory: context.workingDirectory,  // 工具调用时刻的 cwd，非 workspaceRoot
```

每次调用 Skill 工具都会以当前 `runtime.workingDirectory` 重新解析项目级 skill 根（`adapters/src/skills/index.ts:95-107` 无缓存重跑 `resolveDefaultSkillRoots`）。而 `findWorktreeRoot` 从 cwd 向上找到最近的 `.git` 就停——cd 进 `F:\repo\.worktrees\feat-x` 后最近 `.git` 是 worktree 自己的标记文件，根链只剩 feat-x，**不再包含主仓库**。后果：

- cd 后 Skill 工具只看得见 worktree 自带 + 用户级的 skills，主仓库的项目级 skills 加载不到；
- 但系统提示里的清单仍来自主仓库 → 两边可能不一致：按清单点名主仓库的 skill 会报 not found；worktree 的 skill 不在清单里却可以被点名加载。

（此"跟随"仍受 §4 的 cwd 边界策略约束：worktree 建在主仓库目录内（如 `.worktrees/`）才 cd 得住；建在外面会被重置。）

---

## 4. 问题三：Bash 工具 cd 后目录保持吗？

### 4.1 机制：无持久 shell，靠"捕获→边界判定→写回"

每次 Bash 调用新起一个 shell 子进程；cwd 的延续由四步实现：

1. **启动**：用 runtime 记住的 `workingDirectory` 作为 spawn cwd（`apps/zcode-cli/packages/core/src/tool/handlers/bash.ts:438-498`，经 `tool/path-policy.ts:42-59` 解析）。
2. **捕获**：非后台命令被包装为 `<原命令>; __zcode_status=$?; if [ 0 ]; then pwd -P > tmpfile; fi`，成功时把最终物理目录写入临时文件（`adapters/src/exec/cwd-capture.ts:17-49`；cmd 方言用 `cd > tmpfile`，`:60-67`）。
3. **判定写回**：`bash.ts:243-254` 调 `decideBashCwdPolicy`，若返回 `nextWorkingDirectory` 则 `setWorkingDirectory` 写回 runtime。
4. **下一次**回到第 1 步。

### 4.2 判定规则（`core/src/tool/handlers/bash-cwd-policy.ts:36-64`，已逐行核对）

前置门槛，任一不满足直接返回 `{}`（本次 cd 被忽略，目录不变）：

- `status !== "completed"`（超时/取消）→ 不更新；
- `exitCode !== 0` → 不更新（`cd X && failing-cmd` 之后 X 不会被记住）;
- 无 `resolvedCwd`（捕获失败或 `run_in_background: true`，后台命令从不捕获，`bash.ts:474`）→ 不更新；
- `runtimeScope !== "main"`（**子代理内的 Bash 永不写回 cwd**，`:40`）→ 不更新。

边界判断：

- **界内（resolvedCwd 在 workspaceRoot 之内或相同）→ 保持**：下一次 Bash 在 cd 后的目录执行（`:46-57`，写回统一用 workspaceRoot 的原拼写拼相对路径，避免 symlink/盘符大小写写法差异引起误判）。
- **界外 → 重置**：`nextWorkingDirectory = workspaceRoot`，并在 stderr 追加 `Shell cwd was reset to <workspaceRoot>`（`:60-63`）。注意命令本身仍能在外部目录执行完，只是执行完会话 cwd 被拉回。
- 等价性归一化（`:73-130` + `tool/path-normalization.ts`）：比较用"原始写法对 + `realpathSync.native` 物理对"两套边界，任一满足包含即算界内——覆盖 symlink 工作区、macOS `/private` 别名、Windows 盘符大小写、Git Bash `/c/...` 写法、`\\?\` 扩展前缀。

### 4.3 对 worktree 场景的直接推论

| worktree 位置 | cd 后行为 |
|---|---|
| 主仓库目录内（如 `F:\repo\.worktrees\feat-x`，superpowers 技能默认） | **cwd 保持**，同会话可直接在里面干活，每条命令不用带 `cd` 前缀 |
| 工作区外（如全局回退 `~/.config/superpowers/worktrees/`） | **每条命令后被重置回 `F:\repo`**，stderr 出现 reset 提示；只能 `cd <路径> && ...` 前缀硬干，建议另开工作区 |

### 4.4 行为速查表

| 场景 | 下一次 Bash 执行目录 |
|---|---|
| 主线程 cd 到工作区内目录，exit 0 | cd 后的目录（保持） |
| 主线程 cd 到工作区外，exit 0 | 重置回 workspaceRoot + stderr 提示 |
| 命令失败/非零退出/超时/取消 | 不变（cd 被忽略） |
| `run_in_background: true` | 不变（不捕获 cwd） |
| 子代理内任何 Bash | 恒为子代理创建时继承的目录（父发起 Agent 调用瞬间的 cwd） |

相关文案：工具描述宣称 "Working directory persists between calls"（`core/src/tool/handlers/bash-prompt.ts:13`）；子代理提示宣称 "cwd reset between bash calls, use absolute paths"（`core/src/subagent/system-prompt.ts:9`）。

---

## 5. 同会话 cd 进 worktree 的代价与建议

即使 worktree 建在项目内（cwd 保持可用），以下能力仍锚定主仓库，同会话内"失明"：

1. **环境上下文一次性初始化**：AGENTS.md、env git 快照、技能/命令发现固定为主仓库版本（`runtime/methods/context.ts:48` 门闩），cd 后不重建，模型环境信息里的分支/状态仍是主树的。
2. **checkpoint 安全网失明**：checkpoint 基于 git index 快照主树 tracked 文件；`.worktrees/` 通常已被 gitignore，worktree 内改动不进 checkpoint、回滚不作用到它。
3. **UI Git 面板看不到** worktree 内改动；分支显示、变更摘要全程主树。
4. **会话重启后 cwd 不持久**：runtime 重建时 `workingDirectory` 归位为会话工作区路径，复活会话若未先 cd 可能误写主树。

**建议**：临时小改动可在同会话 cd 进 `.worktrees/` 干（进后先 `pwd && git branch` 确认，尽早 commit）；要完整体验（Git 面板、独立配置/MCP/技能、独立记忆）就把 worktree 目录作为新工作区打开——本地目录不生成 `workspaceIdentity`，身份回落为路径（记忆按路径哈希隔离，不与主仓库共用）。

## 6. 调查方法备注

- 全仓库 `rg -il worktree`（排除 node_modules/dist/.temp）命中约 150 文件，多数为设计文档字符串提及；产品代码逐文件归类。
- 关键结论（cwd 策略、workflow 抛错、Skill 工具跟随 cwd、排序白名单）已直接读源码二次核对。
- 桌面 `getWorkspaceRepositoryInfo` 的 linked-worktree 识别（`gitCliRepo.ts:789-852`）目前无 UI 消费方，属挂空能力；若未来 UI 接线，本文"UI 无 worktree 标识"结论需复查。
