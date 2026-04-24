# EverOS 后端框架、记忆机制、Skill 机制与迁移改造分析

> 分析对象：`/Users/guoyongbo/Myproject/everOS/EverOS/methods/evermemos`
> 
> 产出目的：为你的后端项目做技术迁移参考，重点覆盖：长期记忆、Skill 执行机制、Marketplace 改造、Harness 使用情况。

---

## 1. 项目更新状态（已完成）

### 1.1 本地仓库更新结果
- 分支：`main`
- 当前提交：`d40b8703c1ccea7e07d0bfc5c70cc4085c9ad0fc`
- 更新方式：Fast-forward（非破坏）
- 本地未跟踪文件保留：`AGENTS.md`（未被改动）

### 1.2 从 `581b26f` 到 `d40b870` 的变化特征
- 近期提交含 use-case、文档、检索策略优化相关变更。
- 与你关注点直接相关的后端能力仍集中在 `methods/evermemos` 的记忆抽取、检索与多租户隔离链路。

---

## 2. 后端框架全景（可迁移视角）

## 2.1 启动与装配
- 启动入口：`src/run.py`
- 核心启动流程：
  1. 加载环境与参数
  2. `setup_all()` 加载 Addons + DI 扫描
  3. 执行 Mongo migration
  4. 启动 FastAPI (`src/app.py`)
- `LifespanFactory` 会自动组装所有 `LifespanProvider`，并支持 `LIFESPAN_DISABLED` 按名称禁用。

## 2.2 架构分层（实际代码映射）
- `infra_layer`: API 控制器、持久化文档、ES/Milvus/Mongo 适配器
- `biz_layer`: 记忆业务编排（`mem_memorize.py`）
- `memory_layer`: 抽取器、聚类、画像、LLM Prompt
- `agentic_layer`: 统一 MemoryManager、检索与 rerank、vectorize
- `core`: DI、生命周期、多租户、锁、观测、上下文
- `service`: RawMessage/Session/Group/Sender 等业务服务

## 2.3 扩展机制（Addons）
- 通过 `pyproject.toml` 的 `project.entry-points."memsys.addons"` 注入扩展。
- `ADDONS_REGISTRY.load_entrypoints()` 运行时加载插件并合并 DI/异步任务扫描路径。
- 支持 `ADDON_PRIORITY` 设定 Bean 优先级（为企业版覆盖开源实现准备）。

---

## 3. 记忆机制深析（长期记忆原理）

## 3.1 写入链路（从消息到长期记忆）

入口 API：
- `POST /api/v1/memories`
- `POST /api/v1/memories/group`
- `POST /api/v1/memories/flush`
- `POST /api/v1/memories/group/flush`

核心流程（`MemoryController -> agentic_layer.MemoryManager.memorize -> biz_layer.mem_memorize.memorize`）：
1. 写入 RawMessage（待处理消息池）
2. Boundary detection 抽取 MemCell（会话片段）
3. 从 MemCell 抽取：
   - `EpisodicMemory`
   - `AtomicFact`
   - `Foresight`
   - `AgentCase`（agent 场景）
4. Mongo 落库（真值层）
5. 同步 ES/Milvus（检索索引层）
6. 异步触发聚类、画像、AgentSkill 增量抽取

## 3.2 长期记忆原理（设计核心）

这套系统的长期记忆不是“单库存储”，而是**双平面架构**：

- 平面 A（Source of Truth）：Mongo 文档库
  - `v1_memcells`
  - `v1_episodic_memories`
  - `v1_atomic_fact_records`
  - `v1_foresight_records`
  - `v1_agent_cases`
  - `v1_agent_skills`
  - `v1_user_profiles`
  - `v1_mem_scenes`

- 平面 B（Retrieval Index）：ES + Milvus
  - ES：关键词召回 / BM25 / 过滤
  - Milvus：向量召回 / 语义相似
  - Hybrid：ES + Milvus + Rerank

关键点：
- Mongo 负责“完整事实与审计可追踪”。
- ES/Milvus 负责“检索性能与语义匹配”。
- 检索结果经常会“回填 Mongo”取完整字段，避免索引层信息丢失。

## 3.3 增量更新与一致性策略

### 聚类状态（MemScene）
- `MemSceneState` 持续维护：`memcell -> cluster`、cluster centroid、最后时间戳、计数。
- 每条新 MemCell 做增量 cluster assignment，不是离线全量重算。

### 分布式锁（Redis）
- 锁 1：`trigger_clustering:{group_id}`
  - 保护 cluster state 与 profile 抽取过程，避免并发丢更新。
- 锁 2：`trigger_agent_skill:{group_id}:{cluster_id}`
  - 保护同一 cluster 的 skill 增量合并。

### Skill 增量抽取
- 仅用“新 AgentCase + 已有 Skill”做 add/update/none 操作。
- 支持 retire（confidence 低于阈值后从检索层移除，Mongo 仍保留）。
- maturity_score 用于控制“可检索可用性”。

### Profile 增量抽取
- 按 `last_updated_ts` 与 `cluster_last_ts` 筛选新数据。
- Profile Milvus 索引采用 delete-then-insert + 分布式锁，确保索引与画像一致。

## 3.4 检索机制（读路径）
- 统一入口：`POST /api/v1/memories/search`
- 支持：`keyword / vector / hybrid / agentic`
- `agentic` 检索是两轮：
  1. Round1 Hybrid + rerank
  2. LLM 充分性判断（是否信息足够）
  3. 不足则多查询扩展（multi-query）再融合 rerank

这意味着：
- 不是“简单 topK 检索”，而是“有策略的检索编排层”。
- 该能力可直接迁移到你后端做智能检索增强。

---

## 4. 能否改造成 C 端产品记忆后端？结论：**可以，但要补产品级治理**

## 4.1 为什么技术上可行
- 已具备多种记忆粒度（memcell/episode/fact/foresight/profile/agent_skill）。
- 已具备持续更新能力（增量抽取 + 聚类 + 索引回写）。
- 已具备多租户隔离骨架（Mongo/ES/Milvus 全链路 tenant 注入）。
- 已具备混合检索与 agentic 检索能力。

## 4.2 你要补齐的 C 端关键能力

### 数据治理
- 用户级 TTL 与数据分层（热/温/冷）
- 可解释删除（软删 + 物理擦除作业）
- GDPR/隐私法要求：导出、删除、撤回同意

### 成本治理
- LLM 抽取频率节流（按价值触发）
- Embedding 批处理与重试队列
- 低价值会话降级到轻量提要

### 风险治理
- PII 分类与脱敏（写入前 + 检索后）
- Prompt 注入过滤（尤其 agentic 检索）
- 多租户越权检测与审计日志

## 4.3 推荐迁移蓝图（面向你现有后端）

1. 保留“Mongo 真值层 + ES/Milvus 索引层”双平面
2. 先迁移 `episodic_memory + profile` 两个高价值类型
3. 后续再加 `atomic_fact/foresight/agent_skill`
4. 统一事件总线：写库成功后异步索引
5. 对索引写入做幂等（按 doc_id + version）

---

## 5. Skill 执行机制研究与 Marketplace 改造

## 5.1 当前 Skill 机制的真实边界
当前的 `AgentSkill` 本质是：
- 通过 LLM 从 AgentCase 中抽取“可复用方法文本（SOP）”
- 存 Mongo + ES + Milvus
- 检索时作为“知识/策略”返回给上层

它**不是**完整的“可执行技能 runtime”：
- 无独立执行沙箱
- 无 skill 包签名
- 无权限模型（例如读写外部资源）
- 无版本分发/回滚/灰度/审核流

## 5.2 能否改造成 Skills Marketplace？结论：**可以，且已有 30% 基础**

已有基础：
- Addons Entry Points（扩展注入）
- Skill 文本抽取与检索
- 多租户隔离骨架

缺失核心（你需要新增）：
1. Skill Package 规范
   - manifest（name/version/capabilities/runtime）
   - input/output schema
   - 权限声明（网络/文件/secret）
2. Registry 服务
   - 发布、审核、签名、版本、兼容性
3. 执行平面
   - 沙箱执行（容器/微 VM）
   - 资源限额（CPU/内存/时长）
4. 交易与治理
   - 评分、下载、风控、下架、计费
5. 运行时可观测
   - success rate、latency、failure code、tenant 成本

## 5.3 建议演进路线
- Phase A：先做“私有 marketplace”（团队内部）
- Phase B：引入 skill 包签名 + 审核流
- Phase C：开放第三方上架（公有 marketplace）

---

## 6. Harness 工程使用结论

## 6.1 主后端（evermemos）
- 未发现 Harness 作为运行时依赖。
- `methods/evermemos/src` 内未检出 swebench harness 调用。

## 6.2 仓库其他子项目
- `benchmarks/EvoAgentBench` 中明确使用了 `swebench.harness`：
  - `benchmarks/EvoAgentBench/src/domains/software_engineering/swebench.py`
- 该部分属于评测/benchmark 体系，不是主后端在线链路依赖。

结论：
- 你的后端迁移可以不引入 Harness；仅在评测自动化场景再接入。

---

## 7. 可借鉴技术清单（按优先级）

## P0（建议立刻借鉴）
1. 双平面记忆存储（Mongo 真值 + ES/Milvus 检索）
2. 增量聚类 + 画像更新（基于 `last_updated_ts`）
3. 分布式锁保护关键写链路（group/cluster 级）
4. 混合检索 + rerank + agentic 二轮检索

## P1（1-2 个迭代引入）
1. 多租户注入式隔离（Mongo/ES/Milvus 拦截器）
2. Addons EntryPoints 扩展体系
3. 统一检索接口（method 可插拔 + memory type 可扩展）

## P2（规模化阶段）
1. Skill Marketplace（包规范 + 注册中心 + 沙箱）
2. 数据生命周期治理与合规工具链
3. 成本感知策略（抽取/索引/检索全链路）

---

## 8. 给你后端项目的迁移落地方案（90 天）

## 第 0-30 天：MVP 记忆后端
- 迁移对象：`memcell + episodic_memory + profile`
- 建立写入链路：RawMessage -> MemCell -> Episode/Profile
- 建立检索链路：keyword/vector/hybrid
- 数据库：Mongo（真值）+ Milvus（向量）+ ES（关键词）

验收标准：
- 写入成功率 > 99%
- 检索 P95 < 500ms（非 agentic）
- 用户画像可持续更新

## 第 31-60 天：质量提升
- 加入 cluster state（MemScene）
- 加入 agentic 检索两轮策略
- 引入分布式锁与幂等索引
- 加入审计日志与指标看板

验收标准：
- 重复写入不产生脏数据
- 画像/技能更新无并发冲突
- 检索结果稳定性提升

## 第 61-90 天：Marketplace 雏形
- 先做内部 Skill Registry
- 支持 skill 版本与发布审核
- 增加执行沙箱（最小能力）
- 接入权限与计费打点

验收标准：
- 技能可发布/回滚
- 运行有隔离、有审计
- 可统计技能价值（调用、成功率、成本）

---

## 9. 风险与规避建议

1. 索引一致性风险
- 风险：Mongo 成功、ES/Milvus 失败导致召回缺失。
- 建议：Outbox + retry + DLQ + 定时 reconciliation。

2. 多租户越权风险
- 风险：查询漏注入 tenant 过滤。
- 建议：保留双层拦截（注入 + guard 校验）并加集成测试。

3. C 端隐私风险
- 风险：敏感信息被长期存储与错误召回。
- 建议：PII 分类、脱敏、分级加密、删除权全链路。

4. 成本失控风险
- 风险：LLM 抽取和向量化成本飙升。
- 建议：事件价值打分 + 采样策略 + 分层存储。

---

## 10. 关键代码证据（便于二次深挖）

- 启动与装配：
  - `methods/evermemos/src/run.py`
  - `methods/evermemos/src/application_startup.py`
  - `methods/evermemos/src/app.py`

- 核心记忆编排：
  - `methods/evermemos/src/biz_layer/mem_memorize.py`
  - `methods/evermemos/src/memory_layer/memory_manager.py`
  - `methods/evermemos/src/agentic_layer/memory_manager.py`

- Skill 机制：
  - `methods/evermemos/src/memory_layer/memory_extractor/agent_skill_extractor.py`
  - `methods/evermemos/src/infra_layer/adapters/out/persistence/document/memory/agent_skill.py`
  - `methods/evermemos/src/infra_layer/adapters/out/persistence/repository/agent_skill_raw_repository.py`

- 聚类与画像：
  - `methods/evermemos/src/memory_layer/cluster_manager/manager.py`
  - `methods/evermemos/src/memory_layer/profile_manager/manager.py`
  - `methods/evermemos/src/memory_layer/profile_indexer/profile_indexer.py`
  - `methods/evermemos/src/infra_layer/adapters/out/persistence/document/memory/mem_scene.py`

- 检索：
  - `methods/evermemos/src/agentic_layer/search_mem_service.py`
  - `methods/evermemos/src/agentic_layer/retrieval_utils.py`
  - `methods/evermemos/src/biz_layer/retrieve_constants.py`

- 多租户：
  - `methods/evermemos/src/core/tenants/tenant_contextvar.py`
  - `methods/evermemos/src/core/tenants/tenantize/oxm/mongo/tenant_aware_document.py`
  - `methods/evermemos/src/core/tenants/tenantize/oxm/es/tenant_aware_async_document.py`
  - `methods/evermemos/src/core/tenants/tenantize/oxm/milvus/tenant_aware_collection_with_suffix.py`

- 扩展机制：
  - `methods/evermemos/src/core/addons/addons_registry.py`
  - `methods/evermemos/src/addon.py`
  - `methods/evermemos/src/core/addons/addonize/addon_bean_order_strategy.py`

- Harness（评测子项目）：
  - `benchmarks/EvoAgentBench/src/domains/software_engineering/swebench.py`

---

## 11. 对你项目的一句话建议

如果你目标是“C 端长期记忆后端 + 可演进到技能市场”，建议先复制 EverOS 的**双平面记忆架构 + 增量更新策略**，再在第二阶段引入**技能执行平面与市场治理层**，不要一开始把“记忆”和“市场”一起做重。

