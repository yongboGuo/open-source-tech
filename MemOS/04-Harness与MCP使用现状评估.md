## 1. 结论先行

- Harness：当前 MemOS 代码中未发现 Harness 工程接入。
- MCP：有完整内嵌实现，并且是 AI 外部接入的核心扩展接口。

## 2. Harness 是否存在

扫描结果：
- 代码与依赖中未发现 Harness 关键词或对应 runtime/SDK。
- `go.mod` 未包含 Harness 相关依赖。
- 目录结构中无 Harness 模块。

判断：
- 当前项目不基于 Harness。

## 3. MCP 使用深度

### 3.1 协议接入与能力声明

`server/router/mcp/mcp.go` 中：
- 创建 MCP server，启用 tools/resources/prompts/logging capabilities。
- 注册 `/mcp`、`/mcp/readonly`、`/mcp/x/:toolsets` 等路由。

### 3.2 工具面治理

`tool_metadata.go` + `mcp.go`：
- 工具集合按 toolset 管理；
- readOnly/mutation 工具可区分；
- 支持 include/exclude/toolset 组合过滤；
- 服务端执行时再做一次 `enforceToolAccess`（双保险）。

### 3.3 安全控制

`access.go`：
- Origin 校验（同源或 instance-url）；
- memo/attachment 的可见性和归属校验；
- 匿名与登录用户的权限路径区分。

### 3.4 与业务事件联动

`mcp_test.go` 显示：
- MCP mutation 会触发 SSE 事件（memo created/updated/deleted、reaction 等）。
- 说明 MCP 与主业务事件总线是贯通的。

## 4. MCP 与 Harness 的关系（从工程角度）

可这样理解：

- MCP：对外协议层，负责“让 AI 客户端能调你的能力”。
- Harness：通常是内部执行与编排层，负责“如何安全稳定地跑任务/技能”。

MemOS 已有 MCP（北向接口），没有 Harness（南向执行引擎）。

## 5. 如果你要引入 Harness，建议位置

建议在 `MCP tool handler -> APIV1Service` 之间插入执行层：

1. MCP 负责接收/鉴权/工具选择。
2. Harness 层负责任务生命周期（排队、超时、重试、配额、观测）。
3. 业务层负责读写 memos 和记忆库。

这样改造后：
- 你既保留 MemOS 的 API/MCP 兼容性，
- 又能获得可控的 skill 执行运行时。

## 6. 一句话结论

MemOS 在“AI 接口层”做得很好（MCP 完整），在“通用执行引擎层”是留白；这正好给你做 Harness 化改造留出空间。
