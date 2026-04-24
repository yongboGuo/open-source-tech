# Harness 与 Atropos 使用现状评估

## 1. 先给结论

1. Hermes 核心运行时（CLI/Gateway/cron）不依赖 `lm-evaluation-harness` 作为主执行内核。
2. Hermes 在 RL/评测方向明确依赖 Atropos 体系（`atroposlib` + `environments/`）。
3. `lm-evaluation-harness` 在本仓库里主要以“skill 形式”存在，不是核心后端基础设施。
4. 仓库还引入了 `tinker-atropos` 子模块（RL 训练工具链的一部分）。

## 2. 源码证据
- Atropos 环境总览：`environments/README.md`
- 开发文档：`website/docs/developer-guide/environments.md`
- RL 工具：`tools/rl_training_tool.py`
- 依赖声明：`pyproject.toml`（`rl` extra 包含 `atroposlib`）
- 子模块：`.gitmodules`（`tinker-atropos`）
- Harness skill：`skills/mlops/evaluation/lm-evaluation-harness/SKILL.md`

## 3. Hermes 与 Atropos 的关系

### 3.1 集成方式
Hermes 在 `environments/` 下实现了对 Atropos 的整合层：

- 以 `HermesAgentBaseEnv` 承接 Atropos 的 `BaseEnv`。
- 复用 Hermes 工具调用能力跑多轮 agent rollout。
- 支持 reward 计算、benchmark eval、SFT 数据生成。

### 3.2 使用场景
- RL 训练（含 serve/process/evaluate 流程）。
- 基准评测（TerminalBench2、TBLite、YC-Bench）。
- 数据生成与模型能力验证。

### 3.3 与主产品链路的关系
主产品链路仍是 `run_agent.py` + tool runtime。Atropos 主要用于训练/评测，不是用户日常聊天执行主链路。

## 4. Hermes 与 Harness 的关系

### 4.1 `lm-evaluation-harness` 现状
仓库存在一个技能：`evaluating-llms-harness`，内容是如何调用 `lm_eval` CLI 做基准评估。

这说明 Hermes 支持“通过 skill 引导使用 harness”，但不等价于“后端底层采用 harness 作为框架”。

### 4.2 是否使用了 Harness 工程
- 如果你问的是 EleutherAI 的 `lm-evaluation-harness`：不是 Hermes 核心后端工程依赖。
- 如果你问的是“agent harness（编排壳）”：Hermes 自己有一套完整 agent 编排运行时。

## 5. 借鉴建议（对你的项目）

### 5.1 什么时候借鉴 Atropos 思路
当你要做：
- 复杂 agent 任务训练
- benchmark 自动回归
- 可复现实验与 reward pipeline

建议引入 Atropos 风格环境层。

### 5.2 什么时候用 lm-evaluation-harness
当你要做：
- 学术 benchmark 对齐（MMLU/GSM8K/HellaSwag 等）
- 模型横向打分与报告

建议作为独立评测子系统，不要塞进主业务请求路径。

## 6. 风险与边界
- RL 与线上推理混用会拉高系统复杂度。
- 评测指标与线上业务指标不一致时，容易优化错方向。
- benchmark 工具链（含 harness）应该在离线流水线运行。

## 7. 分阶段落地

### Phase 1
- 线上仅保留推理与技能执行。
- 训练/评测放离线 worker。

### Phase 2
- 接入 Atropos 风格环境抽象，统一 reward 接口。
- 将评测结果回流到模型与技能质量系统。

### Phase 3
- 建立“线上行为 -> 离线评测 -> 策略迭代”的闭环。
