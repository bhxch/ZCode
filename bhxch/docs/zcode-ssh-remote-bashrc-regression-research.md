# ZCode SSH 远程 Bash 读不到 .bashrc 的回归调研（v3.15.1 → v3.16.2）

日期：2026-10-24。对比基线：`aac47556`（v3.15.1，Bash 工具可用远程机 ~/.bashrc 里的 alias/函数/env）→ `3c5b2276`（v3.16.2，读不到）。

## 结论

用户怀疑成立：**R3 架构把 SSH 远程从「每次连接经 ssh exec 起临时 zcode-server 进程」改为「连接常驻 Server daemon（systemd --user 服务）」，远程 Bash 工具的祖先进程 env 因此从 sshd 会话 env 变成 systemd user manager env，`.bashrc` 的内容从此不进入进程链。**

## 机制对比

| 环节 | v3.15.1（旧） | v3.16.2（新） |
| --- | --- | --- |
| 远程 server 进程 | `connectRemote` 每次 ssh exec 临时拉起 `zcode-server.cjs`，随 SSH 通道生灭（`packages/server/src/remote/connect.ts:204,376-410`） | `bin/zcode serve --daemon` 注册 systemd --user 服务（Linux unit：`ExecStart=<bin> serve --supervisor --service-entry ...`，无任何 shell 包装、无 `Environment=`，`packages/zcode-server-cli/src/platform/serviceManager.ts:68-76`），跨连接常驻复用 |
| 进程 env 来源 | sshd exec 会话：bash 被 sshd 调用时特例 source `~/.bashrc`（SSH_SOURCE_BASHRC 编译特性）+ sshd 注入 `SHELL/HOME/USER` | systemd user manager：无 `SHELL`、无 `.bashrc` 内容、PATH 为 systemd 默认 |
| env 回捕 | 不需要（env 直接来自 sshd 会话） | Core 启动时一次性 `$SHELL -ilc 'env -0'` 回捕（`packages/services/src/runtime-tools/runtimeLoginShellEnvCapture.ts:40,53-58`），但合并进 process.env 走键名白名单 `INHERITED_LOGIN_SHELL_ENV_KEY_PATTERNS`（`runtimeCommandEnv.ts:48-106`），**自定义 export 被丢弃**；`hasInitializedRuntimeCommandEnv` + 模块级缓存 = daemon 存活期内 `.bashrc` 改动永不生效 |
| shell 选择 | agent env 有 `SHELL=/bin/bash` → bash 优先 | `preferredPosixShellKinds`：`env.SHELL` 缺失或非 bash 时 **zsh 优先**（`apps/zcode-cli/packages/adapters/src/exec/bash-shell-provider.ts:301-306`）；env 回捕候选同理（`runtimeLoginShellEnvCapture.ts:40`）。装了 zsh 的远程机整体切到 zsh 链路 |
| alias/函数通道 | shell-init snapshot：agent env `SHELL=bash` → 快照创建显式 `source ~/.bashrc`（`shell-init-snapshot.ts:138-151,251-260`） | 同一机制但 shell 判定依赖 agent env：无 `SHELL` + 装有 zsh → 快照改读 `~/.zshrc`，`.bashrc` 的 alias/函数全丢 |

三个切断点（按因果排序）：

1. 【主因】常驻 daemon 取代 ssh exec 临时 server：systemd unit 不经任何 shell 启动 Supervisor → Core → agent，sshd 的 bash 特例与 `SHELL` 注入同时消失（`transportAdapter.ts:369-392` ensureDaemon；`cli.ts:90-91` 默认 startupMode=service）。
2. 【直接触发器】`SHELL` 缺失 → zsh 优先：影响 env 回捕 shell 选择与 Bash 工具 shell-init snapshot 的 rc 文件选择（`.zshrc` 替代 `.bashrc`）。
3. 【放大器】常驻 + 进程级一次性缓存：daemon 存活期内 `.bashrc` 的任何修改都不再反映。

注意：shell-init snapshot 只固化函数/alias/shopt/PATH（`shell-init-snapshot.ts:315-368`），**env 变量从来不在快照里**——`.bashrc` 的 env 只能靠进程 env 继承，正是被 daemon 化切断的那条。另外旧版行为依赖用户 `.bashrc` 顶部无非交互守卫（守卫 `case $- in *i*)` 会拦截非交互 source）；新版在白名单/守护链路上无论守卫与否都被过滤。

## 排除项（两版本均未变）

- `runtime-tools/`（回捕+白名单）、`shell-init-snapshot.ts`、`bash-shell-provider.ts`、`runtimeEnv.ts`、`zcodeAgentProcessManager.ts` spawn env 构建均逐字节相同；
- exec adapter 本次 diff 仅 Bash 输出临时目录/生命周期重构；`bash.ts` handler 仅 off-peak 后台策略；
- 旧 remote 主链代码仍在库内但桌面生产入口已不调用（`setupRemoteConnection` 调用数为 0，改走 `SourceServerConnectionManager`）。

## 验证方法（远程机上）

```sh
# 1. 确认 daemon 已注册为 systemd user 服务
systemctl --user list-unit-files | grep zcode   # com.zhipu.zcode.server.*.service

# 2. 看 Core/agent 进程 env：应无 SHELL、无 .bashrc 自定义导出
tr '\0' '\n' < /proc/$(pgrep -f server-core | head -1)/environ | grep -E '^SHELL=|<你的变量名>'

# 3. 对照实验：serviceRegistered=false（无 systemd user bus 走 detached fork）时
#    env 继承当次 ssh exec 会话，症状应减轻 —— 可佐证因果
```

## 缓解与修复方向

- 用户侧（不改代码）：远程机建 `~/.config/environment.d/*.conf` 写 `KEY=value`（systemd user manager 原生读取，daemon 会继承；可同时写 `SHELL=/bin/bash` 纠正 shell 选择），然后 `systemctl --user daemon-reexec` 并重启 zcode 服务。
- 代码侧候选：① systemd unit 注入用户登录 shell 的 `SHELL`；② `preferredPosixShellKinds` 在 `SHELL` 未设时 bash 优先（本就是 `posix-bash` profile）；③ session attach 时重新采集登录 env 而非进程级一次；④ 回捕白名单放行用户显式标记的变量。

相关档案：`docs/architecture/r3-source-server-remote-architecture.md`（R3 D8 删除临时 stdio 主链）、`packages/zcode-server-cli/scripts/verify-remote-ssh.mjs`（systemd 注册断言）。
