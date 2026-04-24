## 1. 先回答你的核心问题

如果你在自己的项目里要执行你写好的 skills（代码执行、WebSearch、文件整理、数据分析、商科数据处理），借鉴 MemOS 时要注意：

- MemOS 当前没有“通用 skill runtime”
- 它有的是 MCP tools（固定工具）+ prompts（提示模板）
- 所以可借鉴点是“工具接口治理模式”，不是“skill 引擎本体”

## 2. MemOS 当前的“Skill 执行”到底是什么

### 2.1 MCP tools = 固定后端能力

在 `server/router/mcp/tools_*.go` 里，工具是代码内注册：
- `list_memos/get_memo/create_memo/update_memo/delete_memo/search_memos/...`
- attachments/relations/reactions/tag 也同理

特点：
- 工具集合由后端发布，非动态安装；
- 执行本质是调用 APIV1Service / Store；
- 具备权限与可见性检查。

### 2.2 prompts = 工作流提示，不是可执行插件

`server/router/mcp/prompts.go` 定义了 `capture/review/daily_digest/organize`。

这些 prompt 本质是“引导模型如何调用 tool”的文本模板，不是独立运行时插件。

### 2.3 AIService 目前只做转写

`AIService` 只有 `Transcribe`，并通过 OpenAI/Gemini provider 实现音频转文字。

结论：
- 当前项目没有代码执行 skill、搜索 skill、数据分析 skill 的可插拔执行框架。

## 3. 借鉴 MemOS 时你该怎么做（实战方案）

### 3.1 可直接借鉴的骨架：MCP 工具治理

MemOS 已经有可复用的治理要素：
- 工具分组 toolsets（memos/tags/attachments/...）
- 只读模式（`X-MCP-Readonly`）
- include/exclude 精细控制
- PAT/JWT 认证与可见性鉴权

你可以把这套治理结构用于自己的 Skill Gateway。

### 3.2 你需要新增的核心：Skill Runtime

建议引入一个独立执行层：

1. `Skill Registry`
- 存技能元数据：名称、版本、输入输出 schema、权限、资源配额。

2. `Skill Executor`
- `code`：沙箱执行（容器/WASM）
- `search`：统一 WebSearch adapter
- `file`：受控文件系统能力
- `analysis`：数据处理（SQL/Pandas/指标模板）

3. `Policy Engine`
- 按用户、租户、场景限制 skill 调用。

4. `Audit/Trace`
- 记录每次 skill 调用的输入、输出、时长、成本、风险标记。

## 4. Marketplace 改造可行性

## 4.1 结论

可以改造成 skills marketplace，但这部分在 MemOS 当前代码中是“空白区”，需要你自己建设。

### 4.2 推荐最小可用版本（MVP）

1. 发布包规范
- `skill.yaml`（元信息 + 权限声明）
- `manifest.json`（entrypoint + runtime）
- `README.md`（使用示例）

2. 安装链路
- 上传包 -> 签名校验 -> 静态扫描 -> 入库 -> 启用

3. 运行链路
- 编排器选 skill -> 校验 policy -> executor 执行 -> 输出标准化

4. 商业链路
- 技能版本管理、兼容矩阵、评分反馈、调用计费（可后置）

### 4.3 安全红线

- 代码执行 skill 必须强沙箱（网络/文件/CPU/内存/超时）
- 外部 API skill 要做密钥托管与最小权限
- 所有 skill 输入输出都要审计可回放

## 5. 给你场景的具体映射

你关心的 skill 类型建议落地为：

1. `code_exec_skill`
- 用于代码执行和自动化脚本。

2. `web_search_skill`
- 统一搜索 API（可接 Google/Bing/自建搜索）。

3. `file_ops_skill`
- 文件归档、重命名、结构化整理。

4. `data_analysis_skill`
- CSV/Excel/SQL 分析，输出图表和结论。

5. `biz_analytics_skill`
- 商科指标（增长、留存、转化、CAC/LTV）模板化计算。

## 6. 一句话结论

MemOS 提供的是“工具化接入与治理框架”，不是“通用技能执行引擎”；你可以把它当作 skill 网关底盘，再补 runtime 和 marketplace。
