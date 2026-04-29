# MemOS 后端/记忆/Skill/Harness 实用版总览

- 更新时间：2026-04-29
- 分析代码：`/Users/guoyongbo/Myproject/MemOS/memos`
- 代码版本：`ee1799851e88674a6920c7a56d93428fcf95e662`

## 一页结论

1. `memos` 是稳定的内容后端，适合作为记忆主库。
2. 长期记忆能力已具备“存储+结构化索引+关系”，缺“语义召回+治理”。
3. skill 现状是 MCP 固定工具，不是动态 skill runtime。
4. Harness 未接入；MCP 接入非常完整，可直接借鉴到你的平台。

## 推荐你立刻做的 3 件事

1. 先做 Postgres + pgvector 的 memory 扩展。
2. 在写链路后接异步 embedding 和 hybrid search。
3. 复用 MCP 工具治理模型，搭建自己的 skill gateway。

## 阅读入口

1. 架构与执行链路：`01-后端框架全景与执行链路.md`
2. 长期记忆改造：`02-记忆机制拆解与C端记忆后端改造方案.md`
3. skill 与 marketplace：`03-Skill执行机制与Marketplace改造方案.md`
4. Harness/MCP 评估：`04-Harness与MCP使用现状评估.md`
5. 可借鉴技术：`05-可借鉴到自研后端的技术清单.md`
6. 分阶段落地：`06-分阶段落地路线图与风险清单.md`
