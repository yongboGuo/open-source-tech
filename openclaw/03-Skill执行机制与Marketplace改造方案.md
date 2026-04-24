## 4. Skill 执行机制研究

### 4.1 Skill 的装载与优先级

Skills 来源优先级（高到低）大意：
- workspace skills
- project `.agents/skills`
- `~/.agents/skills`
- `~/.openclaw/skills`
- bundled
- extra dirs

并有 gating：`requires.bins/env/config/os`。

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/docs/tools/skills.md`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/skills/workspace.ts`

### 4.2 Skill 命令的执行链路

用户 `/skill` 或 `/某技能命令` 后：

1. 解析 skill command spec（包含 dispatch 策略）
2. 若 `command-dispatch: tool`，直接调用指定 tool（绕过模型）
3. 否则改写 prompt，让模型按 skill 执行

关键代码：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/auto-reply/reply/get-reply-inline-actions.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/skills/command-specs.ts`

这个“可直达 tool”的能力对你提到的场景很关键：
- 代码执行
- WebSearch API 调用
- 文件整理
- 数据分析
- 商科数据处理

都可以封装成 skill -> tool dispatch，不必每次让模型自由生成全部执行细节。

### 4.3 Skills 的安装/更新/远程检索

网关已内置：
- `skills.status`
- `skills.bins`
- `skills.search`
- `skills.detail`
- `skills.install`
- `skills.update`

并接入 ClawHub（搜索、详情、安装、更新、origin/lock 管理）。

关键代码：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/gateway/server-methods/skills.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/agents/skills-clawhub.ts`

### 4.4 安全机制（安装与内容扫描）

- 安装流程有安全扫描入口（install security scan）。
- skill-workshop 生成内容也会扫描，critical 直接 quarantine。

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/src/plugins/install-security-scan.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/src/scanner.ts`

---

## 5. Skill 进化机制研究（是否可演化成 marketplace）

### 5.1 已有“进化”链路（skill-workshop）

skill-workshop 具备完整 proposal 生命周期：
- capture（启发式或 LLM reviewer）
- pending / applied / rejected / quarantined
- safe write 到 `<workspace>/skills/...`
- bump snapshot 使新 skill 生效

参考：
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/index.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/src/workshop.ts`
- `/Users/guoyongbo/Myproject/openclaw/openclaw-src/extensions/skill-workshop/src/store.ts`

### 5.2 能否改造成 skills marketplace

结论：**基础设施已具备 60%+。**

已经有：
- 注册中心接口（search/detail/install/update）
- 版本与来源跟踪（origin/lock）
- 命令级执行链路
- 安全扫描与隔离

还需要补：
- 发布者身份与签名体系（包签名/证书）
- 评分、下载量、质量信号
- 权限声明标准（skill capability contract）
- 兼容性矩阵（runtime/version/tool deps）
- 商业化能力（计费、结算、许可证）

---

