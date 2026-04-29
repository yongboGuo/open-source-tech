# Open Source Tech Notes

开源 AI 工程项目的后端研究与落地仓库。  
目标是把“看源码”变成“可直接复用的工程方案”。

## 30 秒总览

| 项目 | 研究重点 | 推荐入口 |
|---|---|---|
| EverOS | 后端架构、记忆、skill、harness | `EverOS/EverOS_Backend_Memory_Skill_Harness_Analysis.md` |
| Hermes | 架构、长期记忆、skills marketplace、落地 Todo | `Hermes/01-后端框架全景与执行链路.md` |
| OpenClaw | 架构、记忆、skill 执行、ACP/Harness评估 | `openclaw/README.md` |
| MemOS | 记忆后端改造、MCP、skill/runtime 改造路线 | `MemOS/README.md` |

## 按目标阅读

1. 我想做“C 端长期记忆后端”
- 先读：`MemOS/02-记忆机制拆解与C端记忆后端改造方案.md`
- 再读：`MemOS/06-分阶段落地路线图与风险清单.md`
- 对照：`项目横向对比-记忆机制_Skill执行_自我进化.md`

2. 我想做“skills 执行与 marketplace”
- 先读：`MemOS/03-Skill执行机制与Marketplace改造方案.md`
- 再读：`openclaw/03-Skill执行机制与Marketplace改造方案.md`
- 补充：`Hermes/03-Skill执行机制与Marketplace改造方案.md`

3. 我想快速借鉴后端工程能力
- 先读：`MemOS/05-可借鉴到自研后端的技术清单.md`
- 再读：`openclaw/05-可借鉴到自研后端的技术清单.md`
- 最后读：`Hermes/05-可借鉴到自研后端的技术清单.md`

## 仓库结构

```text
open source tech/
├── EverOS/
├── Hermes/
├── openclaw/
├── MemOS/
└── 项目横向对比-记忆机制_Skill执行_自我进化.md
```

## 文档索引

### EverOS
- `EverOS/EverOS_Backend_Memory_Skill_Harness_Analysis.md`

### Hermes
- `Hermes/01-后端框架全景与执行链路.md`
- `Hermes/02-记忆机制拆解与C端记忆后端改造方案.md`
- `Hermes/03-Skill执行机制与Marketplace改造方案.md`
- `Hermes/04-Harness与Atropos使用现状评估.md`
- `Hermes/05-可借鉴到自研后端的技术清单.md`
- `Hermes/06-后端工程师落地TodoList（基于E2B与Hermes思路）.md`

### OpenClaw
- `openclaw/README.md`（分主题索引）
- `openclaw/openclaw_backend_memory_skills_harness_analysis.md`（总览）

### MemOS
- `MemOS/README.md`（实用化索引）
- `MemOS/memos_backend_memory_skills_harness_analysis.md`（总览）

### 横向对比
- `项目横向对比-记忆机制_Skill执行_自我进化.md`

## 如何维护（统一标准）

新增项目研究时，建议保持同样结构：

1. `01` 架构与执行链路
2. `02` 记忆机制与 C 端改造
3. `03` skill 执行与 marketplace
4. `04` Harness/协议评估
5. `05` 可借鉴技术清单
6. `06` 路线图与风险
7. `README.md`（该目录索引）+ 总览单文件
