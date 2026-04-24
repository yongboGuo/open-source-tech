## 6. Harness 工程研究

### 6.1 OpenClaw 中的 Harness 是什么

这里的 Harness 主要是两类：

1. **Agent Harness（嵌入式执行器）**
- 通过 `registerAgentHarness` 注册。
- selection 策略支持 `auto` / `pi` / 指定插件 harness。
- 支持 fallback 策略。

关键代码/文档：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/harness/selection.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/plugins/sdk-agent-harness.md`

2. **ACP Harness Runtime（acpx）**
- ACP 会话运行时后端，`acpx` 默认启用（`enabledByDefault: true`）。
- task runtime 中明确包含 `acp`。

关键代码/文档：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/acpx/openclaw.plugin.json`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/tools/acp-agents.md`

### 6.2 是否使用了 Harness.io 工程

在仓库中未发现 Harness.io 相关配置或依赖（未检索到 `harness.io` 等标识）。

本项目里的 “harness” 语义主要是：
- Agent 运行时执行器（runtime harness）
- ACP 运行时后端
- 测试辅助 harness（test harness）

---

