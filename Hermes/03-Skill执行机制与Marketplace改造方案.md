# Skill 执行机制与 Marketplace 改造方案

## 1. 现状结论
Hermes 的 skill 体系已经具备 marketplace 的核心雏形：

- 执行面：`skills_list` + `skill_view` + `/skill-name` 注入执行。
- 管理面：`skill_manage` 支持创建/编辑/补丁/文件写入。
- 安全面：`skills_guard` 风险扫描 + trust policy。
- 供给面：`skills_hub` 多源检索、隔离安装、审计与锁文件。

如果你要做“skills marketplace”，Hermes 现有机制基本可直接迁移。

## 2. 源码证据
- Skill 读取与渐进加载：`tools/skills_tool.py`
- Slash 命令注入：`agent/skill_commands.py`
- CLI 命令转发：`cli.py`、`hermes_cli/commands.py`
- Gateway 侧命令分发：`gateway/run.py`
- 定时任务绑定 skills：`cron/scheduler.py`
- Skill 自进化：`tools/skill_manager_tool.py`
- 安全扫描策略：`tools/skills_guard.py`
- Hub 多源与安装：`tools/skills_hub.py`、`hermes_cli/skills_hub.py`

## 3. 执行链路拆解

### 3.1 发现与加载
`tools/skills_tool.py` 负责：

- 扫描 `SKILL.md` 并读取 frontmatter。
- `skills_list` 仅返回 metadata（节省 token）。
- `skill_view` 按需加载正文与 linked files（references/templates/scripts/assets）。
- 校验平台兼容、disabled 状态、路径安全、环境依赖。

### 3.2 命令注入
`agent/skill_commands.py` 负责：

- 将 skill 映射成 `/skill-name`。
- 加载 skill 后拼接“系统激活块 + skill 正文 + 目录/文件提示”。
- 可选模板变量替换与 inline shell 扩展。

本质：skill 不是独立 runtime，而是对 Agent 行为的“动态操作手册注入”。

### 3.3 入口路由
- CLI：`cli.py` 对 `/skills` 和 `/skill-name` 分别处理。
- Gateway：`gateway/run.py` 同样识别并注入 skill 文本后进入主对话循环。
- Cron：`cron/scheduler.py` 将 job 绑定的 skills 前置注入 prompt。

### 3.4 自进化能力
`skill_manager_tool.py` 支持 agent 在运行中沉淀新技能：

- create/edit/patch/delete
- write_file/remove_file
- 可选结合 guard 做 agent-created 扫描

这部分是“技能自动沉淀”能力，非常适合你做企业知识流程化。

## 4. Marketplace 现状能力

### 4.1 多源供给
`tools/skills_hub.py` 已支持多来源路由：

- `official`
- `hermes-index`
- `skills-sh`
- `well-known`
- `github`
- `clawhub`
- `claude-marketplace`
- `lobehub`

### 4.2 安装安全链
安装流程已经是标准供应链流程：

1. fetch bundle
2. quarantine
3. static scan
4. trust policy 判定
5. confirm + install
6. lock file 记录 + audit log

### 4.3 可信策略
`skills_guard.py` 用 trust level + verdict 组合策略：

- source trust：builtin / trusted / community / agent-created
- verdict：safe / caution / dangerous
- policy：不同来源有不同允许矩阵

## 5. 改造成你项目 Skills Marketplace 的建议

## 5.1 目标能力
- 发布与版本：可发布、可升级、可回滚。
- 评分与推荐：安装量、成功率、用户评分。
- 组织治理：租户级白名单/黑名单、审批流。
- 商业化：私有技能、团队技能、付费技能。

### 5.2 建议服务拆分
- `skill-registry-service`：索引、检索、元数据。
- `skill-package-service`：打包、签名、分发。
- `skill-security-service`：扫描与策略判定。
- `skill-install-service`：租户安装、回滚与锁文件。
- `skill-telemetry-service`：运行质量与使用分析。

### 5.3 数据模型建议
- `skills`（全局技能元信息）
- `skill_versions`（版本、hash、签名、兼容矩阵）
- `tenant_skill_installs`（租户安装状态）
- `skill_exec_metrics`（成功率、时延、错误率）
- `skill_reviews`（评分与评论）

## 6. 面向你当前 E2B 资源的落地重点
你已经有 E2B 沙箱，建议优先做这三件事：

1. Skill 执行统一走 `Skill Runner -> E2B Session`，不在主机直接执行脚本。
2. 每次执行绑定 `tenant_id + user_id + run_id`，做全链路审计。
3. 技能安装与执行解耦：安装只管文件与元数据，执行按需拉起沙箱。

## 7. 风险与边界
- 技能供应链风险：第三方 skill 可能夹带恶意指令。
- 执行成本失控：并发沙箱 + 外部 API 消耗不可控。
- 兼容性风险：技能依赖命令/环境变量在不同运行环境不一致。

## 8. 分阶段落地

### MVP（2 周）
- 仅支持私有技能仓（团队内部）。
- 支持搜索、安装、执行、回滚。
- 基础扫描 + 白名单。

### Beta（3-6 周）
- 多源接入与可信分级。
- 评分与指标采集。
- 运行沙箱化（E2B）全面接入。

### Production（持续）
- 引入签名校验与发布审批。
- 引入计费与配额。
- 建立技能质量排行榜与自动降权机制。
