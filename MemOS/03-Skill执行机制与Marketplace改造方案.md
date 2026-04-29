## TL;DR

1. MemOS 现在没有通用 skill runtime。
2. 当前“skill 执行”本质是 MCP 固定 tools 调业务方法。
3. 你可直接借鉴它的工具治理，再补执行沙箱和 marketplace。

## 1. 现状：MemOS 的 skill 能力边界

| 模块 | 现状 | 含义 |
|---|---|---|
| MCP tools | 固定注册在 `tools_*.go` | 不是动态插件 |
| MCP prompts | 模板化提示词 `prompts.go` | 引导调用，不负责执行隔离 |
| AIService | 主要 `Transcribe` | 不等于通用 skill 引擎 |

## 2. 你的场景映射（代码执行/搜索/分析）

你需要的是“可执行 skill 平台”，建议拆为四层：

1. `Skill Registry`
- 记录技能元信息：版本、输入输出 schema、权限、计费标签。

2. `Skill Runtime`
- `code`：容器/WASM 沙箱
- `search`：统一 WebSearch Adapter
- `file`：受控文件系统
- `data`：SQL/Python 分析执行器

3. `Policy & Security`
- 资源配额、网络白名单、敏感操作审批。

4. `Audit & Observability`
- 全量调用日志 + 成本 + 时延 + 失败原因。

## 3. 直接借鉴 MemOS 的部分

1. MCP 工具治理：`readonly`、`toolsets`、`include/exclude`。
2. 鉴权模型：PAT/JWT + 可见性校验。
3. 业务复用策略：MCP handler 调主业务层，减少双实现。

## 4. 建议的 Skill 执行契约

```json
{
  "name": "biz_analytics_skill",
  "version": "1.0.0",
  "input_schema": {
    "type": "object",
    "properties": {
      "dataset": {"type": "string"},
      "metrics": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["dataset", "metrics"]
  },
  "permissions": {
    "network": false,
    "filesystem": "scoped",
    "max_cpu_sec": 30,
    "max_memory_mb": 512
  }
}
```

## 5. Marketplace 最小版本（MVP）

### 5.1 需要的最小能力

1. 发布：上传 skill 包 + manifest 校验。
2. 安装：签名校验 + 安全扫描 + 入库启用。
3. 调用：编排器选 skill -> policy 校验 -> runtime 执行。
4. 运营：版本管理 + 调用统计 + 下线机制。

### 5.2 先不要做的

1. 复杂分润和商业计费。
2. 过早支持太多 runtime 类型。
3. 无审计的任意代码执行。

## 6. 你的“拿来即用”实施顺序

1. 先复制 MemOS MCP 工具治理模式到你的 Skill Gateway。
2. 优先上线 `code_exec_skill` 和 `web_search_skill` 两种 runtime。
3. 再做 `data_analysis_skill` 与 `biz_analytics_skill` 模板化。
4. 最后再做 marketplace 的上架/审核/评分。

一句话：MemOS 是很好的“skill 网关骨架”，但执行引擎和市场体系要你自己补。
