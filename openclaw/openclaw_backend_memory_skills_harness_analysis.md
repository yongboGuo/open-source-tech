# OpenClaw 后端框架、记忆机制（长期记忆重点）、Skill 执行/进化、Harness 机制研究报告

- 分析日期：2026-04-24
- 代码仓库：`/Users/guoyongbo/Myproject/openclaw/openclaw-src`
- 分析方式：静态代码与文档分析（未启动服务）

---

## 0. 项目更新结果（已完成）

本地仓库已从：
- `c6684af682bcb59bd1ee85cb49f72d4074007e2c`

更新到：
- `f0b6c65e3b0982ec7633b10c7eedb9eafd0242fc`

当前状态：
- 分支：`main`
- 与远端：`0 behind / 0 ahead`
- 未启动任何运行时进程，仅完成拉取与代码研究。

---

## 1. 后端框架全景（OpenClaw 是什么架构）

### 1.1 总体形态：Gateway + Agent Runtime + Plugin Registry

OpenClaw 的后端核心不是“单一 Agent 脚本”，而是一个**网关化控制平面**：

1. **Gateway 守护进程**
- 统一接入 WS 控制面（CLI、macOS App、Web UI、自动化）。
- 对外暴露 typed methods（如 `agent`、`skills.*`、`sessions.*`）。
- 统一事件流与协议验证。
- 参考：
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/concepts/architecture.md`

2. **Agent Loop 执行面**
- 每个 session 串行执行，带队列与 session 写锁。
- 支持 hook 注入（before_prompt_build、tool hooks、agent_end 等）。
- 参考：
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/concepts/agent-loop.md`

3. **Plugin Registry 能力面**
- manifest-first 装载 + runtime register，能力通过统一 registry 暴露。
- 能力类型包括 provider/channel/tool/CLI/service/gateway method/harness/memory capability。
- 参考：
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/plugins/architecture-internals.md`
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/plugins/registry.ts`

### 1.2 Manifest-first 的意义（对你后端可借鉴）

OpenClaw 的插件加载流程先看 manifest，再决定是否真正加载代码模块。意义是：

- 配置校验、可观测性、治理（allow/deny/slots）可以先完成。
- 降低“加载即执行”的安全风险。
- UI/CLI 能在不执行插件逻辑的前提下展示元信息。

这对 C 端产品特别关键：你可以把“插件/技能市场”治理前置，降低供应链风险。

### 1.3 插件独占槽位（slot）机制

`plugins.slots.memory` 决定谁拥有 memory 主能力（exclusive owner）。

- 配置定义：`plugins.slots.memory`
- 注册约束：非 memory 插件不能注册 memory capability；dual-kind 插件若未被选中会被跳过。
- 参考：
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/config/schema.base.generated.ts`
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/plugins/registry.ts`

这套 slot 机制非常适合你后端的“主能力冲突仲裁”（如同时安装多个记忆引擎时谁生效）。

### 1.4 权限与方法作用域

Gateway methods 是按 scope 分级的，`skills.status/search/detail` 为读权限，`skills.install/update` 为 admin 权限。

- 参考：
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/gateway/method-scopes.ts`

这可以直接借鉴成你系统的 RBAC/ABAC 方法级鉴权层。

### 1.5 任务运行时分层

任务 runtime 明确区分：`subagent` / `acp` / `cli` / `cron`。

- 参考：
  - `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/tasks/task-registry.types.ts`

这种 runtime 分层对“同一任务不同执行器”的扩展很有价值。

---

## 2. 记忆机制研究（重点：长期记忆）

## 2.1 记忆能力不是“隐式黑盒”，而是“文件 + 索引 +插件能力”

OpenClaw 明确把记忆做成可审计资产：

- 长期记忆：`MEMORY.md`
- 日记忆：`memory/YYYY-MM-DD.md`
- 梦境/整合日志：`DREAMS.md`

核心说明文档：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/concepts/memory.md`

### 2.2 默认长期记忆实现：memory-core

`memory-core` 在 register 阶段挂载：

- `registerMemoryCapability(...)`
- tools：`memory_search` / `memory_get`
- CLI：`openclaw memory ...`

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/memory-core/index.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/plugins/memory-state.ts`

### 2.3 内置记忆引擎（builtin）关键能力

内置引擎底层是 SQLite，支持：

- FTS5（关键词）
- 向量检索（sqlite-vec 或降级）
- Hybrid merge（vector + BM25）
- 目录监听（watch）
- session transcript 增量同步（可选）
- 全量重建时的**原子 reindex swap**（临时库构建后替换）

关键实现：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/memory-core/src/memory/manager.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/memory-core/src/memory/manager-sync-ops.ts`

几个工程上很强的点：

1. **首次搜索前 bootstrap sync**：避免冷启动时空结果。
2. **watch + debounce + interval 组合**：兼顾实时性与成本。
3. **只读损坏恢复与 provider fallback**：提高线上稳态。
4. **safe reindex 原子替换**：防止重建期间索引半成品污染。

### 2.4 长期记忆“晋升”机制（Dreaming + 短期证据库）

这部分是 OpenClaw 的亮点，真正回答“长期记忆如何不断进化”。

#### 数据层

- 短期证据库：`memory/.dreams/short-term-recall.json`
- 相位信号：`memory/.dreams/phase-signals.json`
- 锁文件：`memory/.dreams/short-term-promotion.lock`

核心实现：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/memory-core/src/short-term-promotion.ts`

#### 评分逻辑（深度阶段 Deep）

默认阈值：
- `minScore = 0.75`
- `minRecallCount = 3`
- `minUniqueQueries = 2`

默认权重：
- frequency 0.24
- relevance 0.30
- diversity 0.15
- recency 0.15
- consolidation 0.10
- conceptual 0.06

再叠加 light/rem phase boost，最后筛选能否晋升到 `MEMORY.md`。

#### 写入安全

- 进程内锁 + 文件锁。
- 临时文件写入后 rename（原子写）。
- promotion marker 避免重复落库。

这套机制比“把所有历史直接喂模型”更工程化，适合产品化长期记忆。

### 2.5 Active Memory（回复前阻塞检索）

`active-memory` 插件在 `before_prompt_build` 钩子触发，满足条件时先执行“记忆子代理”检索，再把摘要注入隐藏上下文。

核心特性：
- 两道门：插件启用 + 会话类型/agent 资格。
- 允许 session 级 on/off toggle。
- 可持久化 recall 子流程 transcript。

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/active-memory/index.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/concepts/active-memory.md`

---

## 3. 能不能做成 C 端产品的记忆后端，并把数据保存在数据库里持续更新？

结论：**可以，而且 OpenClaw 已经给了两条可行路径。**

### 路径 A：沿用 memory-core，外置数据库作为“主事实库”

做法：
- 保留 `MEMORY.md` 作为人类可审计层。
- 每次 promotion 同步写入你的 DB（Postgres/MySQL）。
- 检索时走 vector DB + 结构化过滤，再回填引用。

优点：
- 与 OpenClaw 当前生态兼容度高。
- 迁移成本低。

### 路径 B：直接换 slot 为 DB 型 memory 插件

OpenClaw 已有 `memory-lancedb`（向量数据库插件）示例，可作为模板。

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/memory-lancedb/index.ts`

你可以实现自己的 `memory-yourdb`，通过 `registerMemoryCapability` 提供：
- 检索 runtime
- prompt section
- flush/promotion 协议

### 建议的 C 端记忆后端分层

1. **事实层（relational）**
- 用户偏好、长期事实、标签、时间线、来源。

2. **向量层（vector DB）**
- chunk embedding、ANN 检索、相似度召回。

3. **证据层（short-term evidence）**
- recall hit、query 多样性、跨天复现度。

4. **晋升层（promotion engine）**
- 类似 OpenClaw 的阈值+权重+去重+锁。

5. **治理层（compliance/security）**
- PII 脱敏、TTL、可删除、可导出、审计日志。

---

## 4. Skill 执行机制研究

### 4.1 Skill 的装载与优先级

Skills 来源优先级（高到低）大意：
- workspace skills
- project `.agents/skills`
- `~/.agents/skills`
- `~/.openclaw/skills`
- bundled
- extra dirs

并有 gating：`requires.bins/env/config/os`。

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/tools/skills.md`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/skills/workspace.ts`

### 4.2 Skill 命令的执行链路

用户 `/skill` 或 `/某技能命令` 后：

1. 解析 skill command spec（包含 dispatch 策略）
2. 若 `command-dispatch: tool`，直接调用指定 tool（绕过模型）
3. 否则改写 prompt，让模型按 skill 执行

关键代码：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/auto-reply/reply/get-reply-inline-actions.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/skills/command-specs.ts`

这个“可直达 tool”的能力对你提到的场景很关键：
- 代码执行
- WebSearch API 调用
- 文件整理
- 数据分析
- 商科数据处理

都可以封装成 skill -> tool dispatch，不必每次让模型自由生成全部执行细节。

### 4.3 Skills 的安装/更新/远程检索

网关已内置：
- `skills.status`
- `skills.bins`
- `skills.search`
- `skills.detail`
- `skills.install`
- `skills.update`

并接入 ClawHub（搜索、详情、安装、更新、origin/lock 管理）。

关键代码：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/gateway/server-methods/skills.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/skills-clawhub.ts`

### 4.4 安全机制（安装与内容扫描）

- 安装流程有安全扫描入口（install security scan）。
- skill-workshop 生成内容也会扫描，critical 直接 quarantine。

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/plugins/install-security-scan.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/src/scanner.ts`

---

## 5. Skill 进化机制研究（是否可演化成 marketplace）

### 5.1 已有“进化”链路（skill-workshop）

skill-workshop 具备完整 proposal 生命周期：
- capture（启发式或 LLM reviewer）
- pending / applied / rejected / quarantined
- safe write 到 `<workspace>/skills/...`
- bump snapshot 使新 skill 生效

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/index.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/src/workshop.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/src/store.ts`

### 5.2 能否改造成 skills marketplace

结论：**基础设施已具备 60%+。**

已经有：
- 注册中心接口（search/detail/install/update）
- 版本与来源跟踪（origin/lock）
- 命令级执行链路
- 安全扫描与隔离

还需要补：
- 发布者身份与签名体系（包签名/证书）
- 评分、下载量、质量信号
- 权限声明标准（skill capability contract）
- 兼容性矩阵（runtime/version/tool deps）
- 商业化能力（计费、结算、许可证）

---

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

## 7. 对你自己后端项目的可借鉴技术（高价值）

1. **Manifest-first 插件架构**：先治理后执行。
2. **Slot 独占机制**：主能力冲突可控。
3. **方法级 scope 鉴权**：API 不同级别权限分层。
4. **会话串行 + 文件锁**：避免并发写脏数据。
5. **Atomic reindex swap**：索引重建期间保持可用。
6. **多信号长期记忆晋升**：避免“全量沉淀垃圾记忆”。
7. **short-term evidence store**：让长期记忆可解释。
8. **主动记忆前置注入（active-memory）**：提升交互体验。
9. **Skill 命令直达 Tool dispatch**：减少不确定性。
10. **技能安装/内容安全扫描**：降低供应链风险。
11. **proposal/quarantine 生命周期**：支持技能自动进化。
12. **runtime 抽象（subagent/acp/cli/cron）**：同一编排协议多执行后端。

---

## 8. 你的场景落地建议（代码执行 + WebSearch + 文件整理 + 数据分析）

### 8.1 Skill 执行架构建议

建议采用 `Skill -> Tool Contract -> Executor` 三层：

1. **Skill 层（说明书）**
- 只定义何时用、输入输出约束、流程规则。

2. **Tool Contract 层（强类型）**
- 每个能力定义 JSON Schema 入参（避免模型自由拼装危险命令）。
- 例：`code_exec`, `web_search`, `file_ops`, `data_pipeline`。

3. **Executor 层（可替换运行器）**
- code_exec 在 sandbox/container。
- web_search 通过 provider adapter。
- data_pipeline 支持 Python/SQL/Spark backend。

### 8.2 最小可用实现（建议）

Phase 1（2-3 周）：
- 先做 4 个 tool：`exec_code`、`web_search`、`file_organize`、`biz_analysis`。
- skill 只允许 `command-dispatch: tool`。
- 所有高危操作必须 approval。

Phase 2（3-6 周）：
- 引入短期证据库 + promotion 引擎。
- 长期记忆写入 DB，并支持可审计回溯。

Phase 3（6-10 周）：
- 上线 marketplace alpha：安装、更新、签名、评分、回滚。
- 引入 skill-workshop 类机制，先 pending 后 auto。

---

## 9. 风险与边界

1. **Prompt/Skill 注入风险**：必须保留扫描、审批、沙箱、最小权限。
2. **隐私与合规风险**：长期记忆必须支持删除、导出、TTL、脱敏。
3. **成本风险**：active-memory 与 dreaming 会增加推理与向量成本。
4. **一致性风险**：多写路径（文件+DB）需事件总线或 outbox 保证最终一致。
5. **可解释性风险**：记忆晋升必须保留证据链与评分明细。

---

## 10. 总结结论

- OpenClaw 已经不是“仅对话机器人”，而是可扩展的**后端编排框架**。
- 记忆方面，已经具备从短期证据到长期记忆的完整机制，且可数据库化改造。
- Skill 方面，已具备 marketplace 雏形（检索、安装、更新、扫描、演化）。
- Harness 方面，已形成 agent harness + ACP(acpx) 双体系，不是 Harness.io CI/CD。
- 对你的项目，最佳策略不是照搬 UI，而是复用其：
  - slot + plugin registry
  - promotion memory engine
  - skill dispatch + scanning
  - runtime 抽象分层

