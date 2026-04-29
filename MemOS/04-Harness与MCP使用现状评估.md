## TL;DR

- Harness：当前项目未使用。
- MCP：当前项目深度使用，且实现质量较高，可直接借鉴。

## 1. Harness 现状判断

### 1.1 结论

未发现 Harness 相关 runtime、SDK、模块目录或依赖。

### 1.2 证据来源

1. 依赖层：`go.mod` 无 Harness 依赖。
2. 代码层：全仓库检索无 Harness 关键接入代码。
3. 架构层：执行路径是 API/MCP -> APIV1Service/Store，不存在独立 harness orchestration。

## 2. MCP 使用深度（可复用价值很高）

### 2.1 协议与能力

`server/router/mcp/mcp.go`：
- tools/resources/prompts/logging 全能力启用。
- 支持 `/mcp`、`/mcp/readonly`、`/mcp/x/:toolsets` 路由。

### 2.2 工具治理

`tool_metadata.go` + `mcp.go`：
- `toolset` 分组
- `readonly` 模式
- `include/exclude` 精细控制
- 服务端二次 `enforceToolAccess`

这套治理非常适合你做 skills 网关。

### 2.3 安全控制

`access.go`：
- Origin 校验（同源或 instance-url）
- memo/attachment 可见性与归属校验
- 匿名与登录用户差异化访问

### 2.4 事件联动

`mcp_test.go` 验证了：
- MCP mutation 会触发 SSE 事件（memo/reaction 等）
- 说明 MCP 与主业务事件链路是连通的

## 3. 如果你要“接 Harness”，建议接在哪

推荐插入位置：

`MCP handler` -> `Harness 执行层` -> `APIV1Service/Store`

Harness 执行层负责：
1. 任务排队和并发控制
2. 超时/重试/熔断
3. 成本与配额
4. 审计与回放

这样做的优点：
- 不破坏现有 API/MCP 兼容性
- 能快速获得可控执行能力

## 4. 你的实践建议

1. 先保持 MemOS MCP 不动，新增 `executor` 中间层。
2. 先托管高风险 skill（代码执行、外网调用）。
3. 把每次执行结果挂 trace-id 回写到审计库。

一句话：MemOS 在“AI 接口层”已经很强，缺的是“可控执行层”；Harness 正好补这个缺口。
