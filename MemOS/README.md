# MemOS 实用化研究索引

- 更新时间：2026-04-29
- 源仓库：`/Users/guoyongbo/Myproject/MemOS/memos`
- 代码版本：`ee1799851e88674a6920c7a56d93428fcf95e662`
- 同步状态：本地 `HEAD` 与 `usememos/memos@main` 一致

## 30 秒结论

1. `memos` 是成熟的内容后端，不是 Agent 编排引擎。
2. 它非常适合做 C 端长期记忆底座（结构化存储 + 关系 + MCP）。
3. 真正的 AI 长期记忆还需补：embedding/vector/hybrid recall/记忆治理。
4. skill 目前是 MCP 固定工具，不是动态 marketplace。

## 3 分钟阅读顺序

1. 先看 [02-记忆机制拆解与C端记忆后端改造方案](./02-记忆机制拆解与C端记忆后端改造方案.md)
2. 再看 [03-Skill执行机制与Marketplace改造方案](./03-Skill执行机制与Marketplace改造方案.md)
3. 然后看 [06-分阶段落地路线图与风险清单](./06-分阶段落地路线图与风险清单.md)
4. 最后按需补看架构和技术细节

## 文档目录（按用途）

1. [01-后端框架全景与执行链路](./01-后端框架全景与执行链路.md)
用途：快速理解系统结构、请求链路、扩展点。

2. [02-记忆机制拆解与C端记忆后端改造方案](./02-记忆机制拆解与C端记忆后端改造方案.md)
用途：评估能否承载长期记忆并给出最小改造方案。

3. [03-Skill执行机制与Marketplace改造方案](./03-Skill执行机制与Marketplace改造方案.md)
用途：从 MCP 工具走向可执行 skills 与 marketplace。

4. [04-Harness与MCP使用现状评估](./04-Harness与MCP使用现状评估.md)
用途：明确 Harness 是否存在，MCP 可复用深度有多高。

5. [05-可借鉴到自研后端的技术清单](./05-可借鉴到自研后端的技术清单.md)
用途：挑选可直接迁移到你项目的工程能力。

6. [06-分阶段落地路线图与风险清单](./06-分阶段落地路线图与风险清单.md)
用途：按阶段推进，控制风险与成本。

## 一页总览

- [memos_backend_memory_skills_harness_analysis.md](./memos_backend_memory_skills_harness_analysis.md)

## 立刻开工清单

1. 先把 `Postgres` 作为记忆主库（保留现有 memo 模型）。
2. 新增 `memory_chunk` + `memory_embedding` 两张表。
3. 在 `CreateMemo/UpdateMemo` 后挂异步 embedding 任务。
4. 增加 `hybrid search` 接口（keyword + vector + relation）。
5. 把 MCP 现有工具治理（readonly/toolset/include/exclude）复制到你的 skill 网关。
