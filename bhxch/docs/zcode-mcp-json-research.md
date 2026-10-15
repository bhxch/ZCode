# ZCode 项目级 `.mcp.json` 支持情况调研

- 调研日期：2026-10-15
- 基线版本：`main @ c19b4f13`（docs: add ZCode workflow research notes）
- 调研方式：源码走读（`packages/desktop`、`packages/services`、`apps/zcode-cli`）
- 路径约定：下文相对路径均相对仓库根 `F:\github\ZCode`

## TL;DR

| 问题 | 结论 |
| --- | --- |
| 项目根目录 `.mcp.json` 会被自动加载吗？ | **不会**。ZCode 不支持 Claude Code 式的"项目根 `.mcp.json` 自动注册 MCP"，全仓库没有项目根 `.mcp.json` 的运行时消费点。 |
| `.mcp.json` 在 ZCode 里是什么角色？ | 仅两类：① **插件根目录**下的 `.mcp.json`（随插件装载）；② **设置同步**里作为 Claude Code 外部配置的扫描来源（手动导入，非自动）。 |
| 项目级 MCP 该写在哪？ | `<workspace>/.agents/mcp.json`（键 `mcpServers`）或 `<workspace>/.zcode/config.json`（键 `mcp.servers`，强优先）。 |
| 现有 Claude Code `.mcp.json` 能复用吗？ | 能。`.agents/mcp.json` 用的就是相同的 `mcpServers` JSON 格式（settings-sync 中标记为 `mcpServersJson`），复制/软链即可，无需逐字段转换。 |

---

## 1. 运行时实际消费的 MCP 注册路径

桌面端 Host 按"目录约定"发现 MCP server，加载顺序先 workspace 再 user，同一作用域内 `.zcode` 是强优先来源（读到即不再看 `.agents`），见 `packages/desktop/src/main/mcpUserDirectory/index.ts:37-60`（descriptor 定义）与 `index.ts:327-337`（`readDirectoryServersFromPreferredSources`）：

| 作用域 | 文件 | 配置键 | 优先级 |
| --- | --- | --- | --- |
| 项目级 | `<workspace>/.zcode/config.json` | `mcp.servers` | 强优先 |
| 项目级 | `<workspace>/.agents/mcp.json` | `mcpServers` | fallback |
| 用户级 | `~/.zcode/cli/config.json` | `mcp.servers` | 强优先 |
| 用户级 | `~/.agents/mcp.json` | `mcpServers` | fallback |
| 插件 | `<插件根>/.mcp.json` | `mcpServers` | 由 `.zcode-plugin/plugin.json` 的 `mcpServers` 字段指向 |

补充细节：

- 停用状态写在 server 配置对象内的 `enabled: false`；历史遗留字段 `enable` 读取时就地折叠迁移（`index.ts:156-189`）。
- 配置解析/权限失败会向上抛错而不是按空列表处理，避免 renderer 把空配置当显式输入断开运行中的 server（`index.ts:312-324`）。

## 2. CLI 运行时不读任何 MCP 配置文件

CLI（`apps/zcode-cli`）自身不发现配置，MCP server 由 Host 通过 ZCode Protocol 的 `mcpServers` 全量下发，再转换为 runtime 配置：

- 协议转换：`apps/zcode-cli/packages/bootstrap/src/zcode-protocol/protocol-mcp-config.ts:19`，注释明确"ZCode Protocol 的 mcpServers 是 session/runtime 和状态查询的完整覆盖配置"。
- 因此即使项目根存在 `.mcp.json`，CLI 侧也没有任何代码路径会读取它。

## 3. `.mcp.json` 在本仓库的两类出现场景（都不是自动注册）

### 3.1 插件机制

`apps/zcode-cli/packages/adapters/src/plugins/mcp.ts:26` 只从**插件根目录**读 `.mcp.json`，且必须随插件被装载；官方 auth 字段（`auth: { type: "zcode_official" }`）的 provenance 由宿主运行时生成，写在 `.mcp.json` 里会被覆盖（`mcp.ts:245`、`mcp-official-auth.ts`）。插件脚手架同样按此约定生成 `manifest.mcpServers = "./.mcp.json"`。

### 3.2 设置同步 / 外部 Agent 配置导入

`packages/services/src/settings-sync/settingsSyncService.ts:332` 把项目根 `.mcp.json` 列为外部 agent `claudeCode` 的扫描来源：

```ts
projectFiles: [[".claude", "settings.json"], [".mcp.json"]],
globalFiles: [[".claude", "settings.json"]],
format: "mcpServersJson",
```

`collectMcpImportCandidates`（`settingsSyncService.ts:1590`）扫描后生成导入候选，仅在设置同步 UI 里由用户**手动选择导入**（`importMcpServersForAgent`，`settingsSyncService.ts:2072`），导入目标是写入 ZCode 自己的 config（`addMcpServerToZcodeConfig`）。它是迁移通道，不是运行时来源。

## 4. 实用建议

- 想复用现有 Claude Code `.mcp.json`：复制（或软链）为 `<workspace>/.agents/mcp.json` 即可生效，格式同为 `mcpServers` 对象。
- 需要项目级配置但不与 Claude 格式互操作时，优先 `<workspace>/.zcode/config.json` 的 `mcp.servers` 键（强优先，避免 `.agents` 来源歧义）。
- 临时项目若已放好 `.mcp.json`，也可走设置同步界面从 `claudeCode` 来源手动导入一次。
