# dws CLI 底层实现分析

本文面向需要读代码、定位问题、扩展命令或排查运行时行为的开发者。重点不是罗列用户命令，而是说明 `dws` 这个 CLI 如何从入口启动、如何生成命令树、如何把一次命令转成 MCP `tools/call`，以及认证、缓存、输出、发布这些底层部件怎么配合。

## 技术栈概览

| 层面 | 技术/依赖 | 作用 |
|---|---|---|
| 主语言 | Go 1.25.8 | 单二进制 CLI，入口在 `cmd/main.go` |
| CLI 框架 | `github.com/spf13/cobra` / `pflag` | 根命令、子命令、flag、补全、错误处理 |
| MCP 传输 | `net/http` + JSON-RPC 2.0；本地插件走 stdio | HTTP MCP 服务调用、stdio 插件调用 |
| 认证存储 | 自研 `internal/auth` + `internal/keychain` | profile、OAuth/PAT token、跨平台安全存储 |
| 交互/TUI | Bubble Tea / Huh / Lipgloss | 登录、profile 选择、交互确认等终端 UI |
| 输出处理 | `encoding/json`、`gojq`、CSV/NDJSON 自研格式化 | `-f json/table/raw/pretty/ndjson/csv`、`--fields`、`--jq` |
| 动态发现 | Market registry + MCP `initialize`/`tools/list` | 从远端元数据构造本地命令树和工具 schema |
| 缓存 | `~/.dws/cache` 分区缓存 | registry、tools/list、detail metadata 的冷启动加速和离线降级 |
| 扩展机制 | edition hooks、plugin、embedded skills | 内部版/插件/Agent skills 复用同一核心 CLI |
| 构建发布 | GoReleaser、shell/PowerShell installer、npm wrapper、Homebrew 模板 | 多平台二进制、安装脚本、npm 包二进制转发 |

## 代码结构

核心目录如下：

| 路径 | 职责 |
|---|---|
| `cmd/main.go` | 最薄入口，只调用 `internal/app.Execute()` 并把返回值作为进程退出码 |
| `pkg/cli` | 对外公开的嵌入入口，允许 overlay main.go 设置版本并启动 CLI |
| `internal/app` | 根命令装配、全局 flag、认证命令、动态/静态命令挂载、runner、升级、skills 安装 |
| `internal/cli` | canonical MCP 命令树、schema 查询、flag 从 JSON Schema 的映射、参数合并和校验 |
| `internal/compat` / `internal/helpers` | 旧版/公开产品命令、helper-only 命令、与 envelope 命令合并 |
| `internal/discovery` / `internal/market` | 拉取 server registry，执行 MCP runtime discovery，并写入缓存 |
| `internal/ir` | 把 discovery 结果归一成产品/工具/flag/auth 的 canonical catalog |
| `internal/transport` | HTTP/stdio JSON-RPC 客户端，协议协商、重试、可信域、错误分类 |
| `internal/output` | 响应解包、格式化、字段筛选、jq 过滤 |
| `internal/auth` / `internal/keychain` | 登录态、profile、token 缓存和跨平台密钥存储 |
| `internal/plugin` | 外部 MCP 插件 manifest、加载、auth 配置 |
| `skills/` + `skills_embed.go` | Agent skill 文档树，编译时嵌入二进制 |
| `scripts/` / `build/` / `.goreleaser.yaml` | 本地构建、安装、发布和包管理器模板 |

## 启动链路

最短调用链：

```text
cmd/main.go
  -> internal/app.Execute()
    -> normalizeProcessProfileArgs()
    -> signal.NotifyContext()
    -> newPipelineEngine()
    -> NewRootCommandWithEngine()
    -> pipeline.RunPreParse()
    -> root.ExecuteC()
```

`Execute()` 做了几件底层兜底：

- 用 `defer recover` 把 panic 统一转成退出码 `5`。
- 预解析 `--profile`，包括 `--profile a, b` 这类被 shell 拆开的情况。
- 建立可被 SIGINT/SIGTERM 取消的 context。
- 初始化 timing collector；退出时停止 stdio 子进程、打印/写入性能报告。
- 在 Cobra 解析前跑 pipeline 的 PreParse，用来修正常见模型生成参数错误，例如 flag 命名或粘连参数。
- 统一改写 Cobra 错误、输出结构化错误，并按 `internal/errors` 的分类返回退出码。

## 根命令装配

`internal/app.NewRootCommandWithEngine()` 是命令树入口。它先构造全局状态：

- `GlobalFlags` 承载 `--format`、`--timeout`、`--profile`、`--dry-run`、`--mock`、`--debug` 等全局 flag。
- `cli.EnvironmentLoader` 负责后续 catalog/discovery 加载。
- `runtimeRunner` 负责把 CLI invocation 执行成真实 MCP 调用。

然后挂载三类命令：

1. 固定工具命令：`auth`、`profile`、`api`、`skill`、`cache`、`catalog`、`config`、`doctor`、`completion`、`recovery`、`upgrade`、`version`、`plugin`、`schema`。
2. 运行时产品命令：`newLegacyPublicCommands()` 会从 registry/envelope 构造公开产品命令，并与 helper 命令合并。
3. 扩展命令：`loadPlugins()` 加载插件，`pat.RegisterCommands()` 加 PAT 授权命令，edition hook 可继续注册内部版命令。

根命令的 `PersistentPreRunE` 是所有子命令执行前的公共入口：设置 runtime profile、应用 client-id/client-secret 覆盖、配置日志级别、处理 `--output` 输出文件，并运行 edition 的 hook。

## 命令树如何生成

项目里有两条相互补充的命令生成路径。

### 1. Public/legacy 命令树

`internal/app/legacy.go` 的 `loadDynamicCommands()` 是公开命令树的主要来源：

```text
cache.LoadRegistry(partition)
  -> cache hit: 直接用缓存，必要时后台 revalidate
  -> cache miss: fetchRegistryServers()
  -> editionmerge.MergeSupplement()
  -> SetDynamicServers()
  -> loadCachedDetailsFast() + loadCachedToolNames()
  -> compat.BuildDynamicCommands()
```

这条路径强调启动速度：先用本地 registry cache 构造命令树，过期时后台刷新；冷启动或无缓存时才同步请求 registry。工具详情和 tools/list 也尽量从缓存读取，避免每次启动都对所有 MCP 服务握手。

如果缓存数据导致构建 panic，`buildEnvelopeCommandsSafe()` 会先隔离当前分区缓存并重试；仍失败时降级到 helper 命令，避免连 `dws cache refresh` 都无法执行。

### 2. Canonical MCP 命令树

`internal/cli.NewMCPCommand()` 从 `ir.Catalog` 生成 `dws mcp ...` 下的 canonical 命令。每个工具命令的 flag 来自 MCP input schema：

- string/integer/number/boolean 映射为对应 pflag 类型。
- array 类型用 `StringSlice` 接收，再转成目标类型数组。
- object 或复杂字段走 `--json` / `--params`。
- schema 参数名中的 `_` 会转成 CLI flag 的 `-`。
- `flag_hints` 可提供 alias 和 shorthand；冲突或保留名会被跳过，避免 pflag panic。

一次工具命令执行时，参数来源按优先级合并：

```text
--json
  + --params
  + 单个 flag override
  + 显式 @file / @-
  + 隐式 stdin pipe fallback
```

合并后经过 pipeline PostParse、JSON Schema 校验、敏感工具确认、pipeline PreRequest，最后生成 `executor.Invocation` 交给 runner。

## Discovery 与 IR

`internal/cli.EnvironmentLoader.Load()` 负责把当前环境变成 `ir.Catalog`：

1. 如果设置 `DWS_CATALOG_FIXTURE`，直接读取本地 JSON fixture。
2. 确定 discovery base URL：测试覆盖 > edition DiscoveryURL > 开源默认 URL。
3. 读取 `DWS_CACHE_DIR` 对应的缓存。
4. 如果已有可用 catalog cache，优先返回，缩短启动时间。
5. 有 token 时请求 Market registry，然后并发对需要刷新的 server 做 MCP runtime discovery。
6. discovery 结果进入 `ir.BuildCatalog()`，得到稳定的产品、工具、schema、auth、flag overlay、lifecycle 信息。

`internal/discovery.Service` 对单个 server 的 runtime discovery 顺序是：

```text
transport.Initialize()
  -> transport.NotifyInitialized()  # best effort
  -> transport.ListTools()
  -> optional Detail API merge
  -> cache.SaveTools()
```

并发发现使用每 server 默认 2 秒超时；超时但本地有 tools cache 时，会退回缓存结果。

## 一次命令如何变成 MCP tools/call

核心执行器是 `internal/app.runtimeRunner`。单 profile 的路径如下：

```text
runtimeRunner.Run()
  -> runSingle()
    -> resolve endpoint
      -> mock endpoint / direct runtime / catalog product endpoint / tool endpoint correction
    -> executeInvocation()
      -> resolve auth token
      -> dry-run/mock/auth preflight
      -> preflightDocDownload()
      -> transport.Client.CallTool()
      -> PAT/business error classification
      -> content scan
      -> success envelope
```

endpoint 解析有几个优先级：

- `--mock` 使用 mock endpoint。
- `shouldUseDirectRuntime()` 命中的命令直接走动态 server endpoint 映射。
- 否则加载 catalog，按产品和工具查找 endpoint。
- 如果一个 CLI 命令背后存在多 server 工具归属，`directRuntimeToolEndpoint()` 会用工具级 endpoint 修正产品级 endpoint。

`transport.Client.CallTool()` 最终发送的 JSON-RPC 形状是：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "tool_name",
    "arguments": {}
  }
}
```

成功后 runner 会把 MCP content 包成：

```json
{
  "endpoint": "redacted endpoint",
  "content": {
    "success": true
  }
}
```

compat/helper 命令在输出层会解包 `content`，所以用户看到的通常是业务 payload，而不是完整 `executor.Result`。

## HTTP 与 stdio 传输

HTTP 客户端在 `internal/transport/client.go`：

- 默认总超时 30 秒。
- Dial timeout 3 秒、TLS handshake 10 秒、response header 20 秒。
- 使用 `http.ProxyFromEnvironment`，支持代理环境。
- TLS 最低版本 TLS 1.2。
- 最多重试 1 次，主要处理可重试 HTTP 状态。
- redirect 超过 10 次失败；跨 host redirect 会剥离 `Authorization` 和 `x-user-access-token`。
- bearer token 只会发给可信 endpoint，默认可信域是 `*.dingtalk.com`，可用 `DWS_TRUSTED_DOMAINS` 覆盖。

MCP 协议协商按版本降序尝试：

```text
2025-03-26
2024-11-05
2024-06-18
```

只有 JSON-RPC 协议层错误才会尝试下一个版本；HTTP/dial 失败会直接返回，避免不可达 endpoint 被三次 dial 放大耗时。

stdio 插件使用 `internal/transport/stdio.go`，路径与 HTTP 类似，也是 `initialize`、`tools/list`、`tools/call`，但请求通过本地子进程的标准输入输出传输。

## 认证、profile 与安全边界

认证相关主要分三层：

- `internal/app/auth_command.go`：用户可见的 `dws auth ...` 命令。
- `internal/auth`：profile、client id/secret、runtime token、PAT/host-owned PAT 判定。
- `internal/keychain`：跨平台 secret 存储抽象。

Keychain 策略：

- macOS 默认用系统 Keychain 保存 DEK，业务数据用 AES-256-GCM 加密。
- Linux 使用文件型 DEK + AES-256-GCM。
- Windows 使用 DPAPI + Registry。
- `DWS_DISABLE_KEYCHAIN=1` 可让 macOS 在沙盒环境退回文件型 DEK，但安全性弱于系统 Keychain。

profile 的本地身份键是 `profileId`，新登录默认由 `corpId:userId` 组成。这样同一台机器可以同时保存同一个钉钉组织下的多个账号：

```text
profiles.json
  primaryProfile/currentProfile/previousProfile = profileId
  profiles[].profileId = corpId:userId

keychain
  dingtalk-workspace-profile-<profileId> = token material
```

`corpId` 仍可作为兼容选择器，但只有在本地唯一匹配一个 profile 时才生效；同一 `corpId` 下有多个账号时，`profile use`、`auth logout --profile`、`--profile` 等路径应使用 `profileId` 或唯一 profile 名。旧的单组织 token 和 corp-scoped keychain slot 只作为迁移/兜底读取路径，保存新 token 时会写入 profile-scoped slot。

runner 执行时会先解析当前 profile；如果 `--profile` 是逗号分隔并且不能被解析成单个 profile，会进入 multi-profile 模式，逐个 profile 执行同一 invocation，并合并每个组织的结果和错误。

账号相关的运行时缓存也按 `profileId` 隔离。全局 registry 仍按 edition 分区共享；`tools/list`、Detail API 元数据和 runtime refresh 使用：

```text
<edition-partition>/profile/<profileId>
```

底层 cache store 会把 `/`、`\`、`:`、空格等字符替换成 `_` 后落盘，所以 `corpId:userId` 不会直接变成路径分隔符。这样同组织多账号不会复用彼此的工具授权状态或 detail 缓存。

## 输出与错误模型

输出入口在 `internal/output`：

- `WriteCommandPayload()` 按 `--format` 选择 json/table/raw/pretty/ndjson/csv。
- `--fields` 在输出前做字段投影。
- `--jq` 使用 gojq 直接过滤 JSON payload。
- compat/helper invocation 成功时会自动解包 `response.content`，让输出更接近业务结果。

错误分类在 `internal/errors`，退出码对外稳定：

| 退出码 | 类别 |
|---|---|
| 0 | success |
| 1 | API/upstream |
| 2 | auth |
| 3 | validation |
| 4 | PAT |
| 5 | internal |
| 6 | discovery |

`-f json` 时错误会输出结构化字段；普通模式下走 human-readable 错误。`--debug` / `--verbose` 会提高日志和错误细节。

## Skills 与 Agent 集成

`skills_embed.go` 使用 Go embed 把 `skills/` 整棵树编译进二进制。`dws skill setup` 默认从当前二进制内嵌的 skill 释放到临时目录，再复用目录安装逻辑写入 Agent skill 目录。

这样做的效果是：升级 `dws` 二进制后，重新执行 `dws skill setup` 安装的是这个二进制随带的 skill 版本，而不是本地 checkout 里可能过期的 `skills/`。

安装模式分两类：

- `mono`：一个 `dws` skill，默认稳定模式。
- `multi`：按产品拆分多个 skill，目前是 preview/experimental。

## 插件与 edition 扩展

底层扩展点有三类：

- `pkg/cli` 暴露 `SetVersion()`、`Execute()`、`MCPIdentityHeaders()`，方便外部 overlay 嵌入。
- `pkg/edition` 提供 edition hooks，例如 static/fallback/supplement servers、额外命令注册、认证错误处理、结果分类。
- `internal/plugin` 读取插件 manifest，把第三方 HTTP/stdio MCP server 合入命令树；插件可以带自己的 token、headers 和 trusted domains。

这也是项目里“开源核心”和“内部/插件能力”能复用同一个 runner、transport、output、auth 框架的原因。

## 构建与发布

本地最小构建：

```bash
go build -buildmode=pie -trimpath -ldflags="-s -w" -o dws ./cmd
```

对应脚本是 `scripts/dev/build.sh`，`make build` 会调用它。

发布由 `.goreleaser.yaml` 管理：

- main package 是 `./cmd`。
- 输出二进制名 `dws`。
- 目标平台：darwin/linux/windows × amd64/arm64。
- 通过 ldflags 注入 `internal/app.version`、`gitCommit`、`buildTime`。
- 产物为 tar.gz，Windows 为 zip。
- SHA256 校验和文件名是 `checksums.txt`。

npm 包不是 Node 实现 CLI，而是 wrapper：

- `build/npm/bin/dws.js` 查找 `vendor/dws` 或 `vendor/dws.exe`。
- 用 `child_process.spawnSync()` 透传用户参数和 stdio。
- `install.js` 从发布归档解出对应平台二进制，并安装/缓存 skills。

shell/PowerShell 安装脚本直接下载 GitHub Releases 产物；`scripts/install.sh` 还支持 Gitee 镜像和国内自动 fallback。

## 调试入口

常用定位点：

| 问题 | 先看哪里 |
|---|---|
| 命令不存在或 help 缺叶子 | `internal/app/legacy.go`、`internal/compat`、`dws cache refresh`、`DWS_CATALOG_FIXTURE` |
| flag 解析或参数合并异常 | `internal/cli/canonical.go`、`internal/executor/invocation.go` |
| endpoint 解析错误 | `internal/app/direct_runtime.go`、`internal/app/runner.go`、`internal/cli/loader.go` |
| MCP 请求失败 | `internal/transport/client.go`、`~/.dws/logs/dws.log`、recovery snapshot |
| 登录态或 profile 问题 | `internal/app/auth_command.go`、`internal/auth`、`internal/keychain` |
| 输出格式不对 | `internal/output/formatter.go`、`filter.go`、`pretty.go`、`csv.go`、`ndjson.go` |
| skill 安装版本不对 | `skills_embed.go`、`internal/app/skill_setup*.go`、`DWS_SKILL_SOURCE` |
| 插件启动或 schema 问题 | `internal/plugin`、`internal/app/plugin_*.go`、`internal/transport/stdio.go` |

建议排查顺序：

1. 用 `dws version` 确认当前二进制版本和 edition。
2. 用 `dws cache status` / `dws cache refresh` 看 discovery 缓存是否可用。
3. 用 `dws schema <path> -f pretty` 看工具 schema、flag overlay 和授权元数据。
4. 用 `--dry-run -f json` 看最终 `tools/call` payload。
5. 用 `--debug` 查看 stderr 和 `~/.dws/logs/dws.log`。

## 核心结论

`dws` 的底层不是一组手写静态命令，而是“缓存优先的 MCP 动态聚合 CLI”：

- Cobra 只负责承载命令树和 flag。
- 命令树主要来自 registry/envelope/tools-list/detail metadata。
- 真正执行统一收敛到 `executor.Invocation -> runtimeRunner -> transport.Client.CallTool()`。
- 安全边界放在 token 解析、可信域、redirect 清洗、参数校验、PAT/业务错误分类和输出通道隔离上。
- skills、npm、安装脚本、edition、plugin 都围绕同一个 Go 二进制核心做包装，不重复实现 CLI 逻辑。
