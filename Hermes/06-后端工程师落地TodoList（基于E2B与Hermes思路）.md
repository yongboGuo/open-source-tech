# 后端工程师落地 Todo List（基于 E2B 与 Hermes 思路）

## 0. 目标定义
在你现有项目中落地一套：
- 可执行 skills（含代码执行 / webSearch / 文件处理 / 数据分析）
- 在服务器集中编排、在 E2B 沙箱隔离执行
- 支持长期记忆、跨会话召回、技能市场化扩展

## 1. 第 1 周（架构与基础协议）
- [ ] 定义 `AgentRuntime` 接口：`run(input, session_ctx)`。
- [ ] 定义 `ToolCall`/`ToolResult` 数据结构（JSON Schema）。
- [ ] 建立 `ToolRegistry`（注册、解析、分发、权限校验）。
- [ ] 抽离 `SkillRuntime`（list/load/invoke）。
- [ ] 定义统一 `ExecutionContext`（tenant_id, user_id, session_id, run_id）。

验收标准：
- [ ] 能用一条 API 跑通“模型 -> 工具调用 -> 工具结果回注入 -> 最终回答”。

## 2. 第 2 周（E2B 执行闭环）
- [ ] 接入 E2B sandbox manager（创建、复用、销毁）。
- [ ] 工具执行统一改为在 E2B 内运行。
- [ ] 实现 artifacts 回传（文件、日志、截图、统计）。
- [ ] 加入超时、取消、重试机制。
- [ ] 记录执行审计日志（命令、耗时、资源、结果摘要）。

验收标准：
- [ ] 并发 20 路稳定运行，无跨会话污染。

## 3. 第 3 周（Skills 体系最小上线）
- [ ] 实现 `skills_list`（metadata）与 `skill_view`（正文+附件）。
- [ ] 实现 `/skill-name` 注入执行链路。
- [ ] 支持 skill 依赖声明（env vars、credential files）。
- [ ] 支持 skill disabled 配置（全局 + 平台级）。

验收标准：
- [ ] 10 个示例 skills 可稳定调用（代码、搜索、文件、数据处理）。

## 4. 第 4 周（安全与治理）
- [ ] 引入 skill scan（注入、外传、破坏性命令检测）。
- [ ] 建立 trust policy（official/trusted/community）。
- [ ] 建立 quarantine 安装流程。
- [ ] 高风险动作增加审批网关（approve/deny）。

验收标准：
- [ ] 能阻断高风险 skill 安装与执行。

## 5. 第 5-6 周（记忆与召回）
- [ ] 实现 DB MemoryProvider：`prefetch` + `sync_turn`。
- [ ] 会话数据入库（消息、工具调用、结果摘要）。
- [ ] 实现 `session_search`（全文检索 + 摘要）。
- [ ] 加入用户纠错入口（replace/remove 事实）。

验收标准：
- [ ] 用户跨会话追问能找回历史方案与关键上下文。

## 6. 第 7-8 周（Marketplace 雏形）
- [ ] skill registry（发布、版本、安装、回滚）。
- [ ] lock file + audit log。
- [ ] 基础 ranking（安装量、成功率、最近活跃）。
- [ ] 租户级技能策略（允许/禁用清单）。

验收标准：
- [ ] 两个租户可独立管理技能与执行策略。

## 7. 非功能与运维清单（并行推进）
- [ ] 指标：QPS、tool latency、error rate、cost/session。
- [ ] 日志：run_id 全链路串联。
- [ ] 限流：按租户/用户/技能维度。
- [ ] 配额：E2B 并发上限与预算控制。
- [ ] 灰度：新技能先小流量再全量。

## 8. 你现在就可以下发给团队的任务包
- [ ] Backend A：ToolRegistry + Runtime 编排。
- [ ] Backend B：E2B sandbox 执行器 + artifact 通道。
- [ ] Backend C：SkillRuntime + scan + install。
- [ ] Backend D：MemoryProvider + session_search。
- [ ] SRE：观测、告警、成本面板、限流与配额。

## 9. 最低上线门槛（Go/No-Go）
- [ ] 高风险命令有审批链。
- [ ] 每个 run 都可审计可追踪。
- [ ] 记忆支持删除与纠错。
- [ ] E2B 并发压测通过并设定硬阈值。
- [ ] 失败可降级（禁用某技能、回滚某版本）。

## 10. 并发与资源容量估算模板（给后端/SRE）

### 10.1 估算公式
- 峰值并发沙箱数 `S = QPS * 平均执行时长(秒) * 沙箱使用率`
- 峰值 CPU 核需求 `C = S * 单沙箱平均CPU核占用`
- 峰值内存需求 `M = S * 单沙箱平均内存(GB)`

### 10.2 建议先做 3 档压测
- 轻任务档：文件读写、简单搜索、无长命令。
- 中任务档：代码执行 + 多次 webSearch + 数据处理。
- 重任务档：长脚本、依赖安装、批量分析。

### 10.3 监控硬阈值
- [ ] E2B 活跃会话数
- [ ] 会话平均时长与 P95 时长
- [ ] 每租户并发与失败率
- [ ] 超时率与重试率
- [ ] 单技能成本与超预算比例

### 10.4 保护策略
- [ ] 每租户并发上限
- [ ] 每用户速率限制
- [ ] 超时自动熔断与排队降级
- [ ] 高成本技能分级执行（普通/高优先）
