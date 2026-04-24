# MemOS 后端/记忆/Skill/Harness 研究总览

- 分析时间：2026-04-25
- 分析代码：`/Users/guoyongbo/Myproject/MemOS/memos`
- 代码版本：`ee1799851e88674a6920c7a56d93428fcf95e662`

## 一页结论

1. MemOS 是成熟的笔记后端，不是 agent orchestration 框架。
2. 记忆机制是“持久化 memo + payload 派生字段 + relation 关联”，可作为长期记忆底座。
3. 当前没有语义向量长期记忆能力，需要额外扩展 embedding/hybrid recall。
4. skill 执行机制当前主要是 MCP 固定工具，不是动态 skills runtime。
5. 未发现 Harness 工程接入。
6. 可借鉴价值很高：双协议 API、CEL 过滤引擎、MCP 工具治理、认证体系、迁移体系。

## 详见拆分文档

1. `01-后端框架全景与执行链路.md`
2. `02-记忆机制拆解与C端记忆后端改造方案.md`
3. `03-Skill执行机制与Marketplace改造方案.md`
4. `04-Harness与MCP使用现状评估.md`
5. `05-可借鉴到自研后端的技术清单.md`
6. `06-分阶段落地路线图与风险清单.md`
