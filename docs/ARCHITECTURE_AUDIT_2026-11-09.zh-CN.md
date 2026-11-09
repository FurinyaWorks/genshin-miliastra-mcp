# MCP 架构基线与轻量治理计划（2026-11-09）

> 审查对象：FurinyaWorks/genshin-miliastra-mcp。目标是让 Codex 等 Agent **低上下文开销、可靠且安全地使用**这套 MCP；保留已有正确架构，不为重构而重构。
>
> 基线：Fork `main@59c576e` 已合入 7.2 资产 PR #1；逐文件 blob SHA 比对与上游 `8itlab/main@bcc7579` 的 **3,596 个文件全部相同**。这是 GitHub 仓库内容对齐，**不等于 Windows 本地 MCP 进程已经安装/重启**。

## 1. 现状与优势

```text
Agent (Codex / Other MCP Host)
  └─ src/server.mjs — stdio MCP + tool/resource registration
      └─ src/host/core-client.mjs — local RPC
          └─ src/host/daemon.mjs + src/core.mjs
              ├─ src/core/*.ts — projects / transactions / recovery
              ├─ src/scene-save.mjs — scene, prefabs, UI, economy, GIL writes
              ├─ src/scene/* — assets / images / effects / protobuf helpers
              ├─ src/playtest*.mjs + Python UI automation
              └─ workspace (working copies, traces and published kit)
```

- 目前 42 个 `src/` 下 JS/TS 模块、约 92 条静态本地 import，**0 个静态循环依赖 SCC**。检查只覆盖静态 JS/TS import，不包含动态 import、Python 或外部包；不要把这一结论扩展成整个运行时无环。
- 现有保护值得保留：默认复制存档、事务恢复与回滚、Core/Host 隔离、本地 Core 身份令牌与 workspace 管理、结构化错误、Zod 参数校验。
- `search_knowledge` / `read_knowledge` 和 `miliastra://workflow` / `miliastra://platform-rules` 等 Resources 已能按需给出资料；9 个面向底层编译步骤的工具默认隐藏。这些均是有价值的上下文工程实践。
- 资产目录中大多数文件是图片/JSON/知识材料，不等于同等数量的逻辑模块；不要据此新增空架构层。

## 2. 已核实的问题

| 优先级 | 项目 | 证据和影响 |
| --- | --- | --- |
| P0 | MCP 工具风险注解不一致 | `src/server.mjs` 将 `destructiveHint` 一律设置为 `!readOnly`；`publish_kit` 在 `src/tool-registry.mjs` 中设 `readOnly=true`，但 `src/publish.mjs` 会在 `workspace/publish/<name>` 创建文件、复制视频。只读标签失真；“写入新副本”也不一定是破坏性操作。 |
| P1 | 工具定义上下文过大 | 注册工具 96 个，默认暴露 87 个；可见工具描述文本约 **106,010 字符**，**不含 Zod 转换的 Schema**。13 个描述长于 2,000 字符；`scene_add_components` 约 13,001 字符，`playtest_start` 约 9,293。若 Host 全量注入，会吃掉大量模型上下文。 |
| P1 | 没有 GitHub 自动检查门禁 | 截至基线不存在 `.github/workflows/`，PR #1 的 check-runs = 0。已有核心、host、asset 等离线测试脚本，不能把“有测试代码”当成“每次 PR 自动验证”。 |
| P1 | 7.2 UI 图片覆盖未完成 | `src/scene/image-library/index.json` 仍是 **15 类 / 1,590 项**。造物、四动作与 26 特效已补，但 UI 图差量、其他素材尚未完整覆盖；不能因 Fork 与上游一致就声称素材完整。 |
| P2 | 高维护密度模块 | `src/scene-save.mjs` 约 **3,051 行 / 334,543 字符 / 100+ 导出**，含场景存档、造物、UI、经济等不同责任；`src/tool-registry.mjs` 约 **687 行 / 177,783 字符**。文件长度并不等于函数级 CC，需 AST 实测。 |

## 3. 渐进式披露的正确边界

分清两个问题：
1. **MCP Server**：把 `tools/list` 的每个工具说明控制在“做什么、必须参数、风险、副作用、失败处理”；长工作流、示例和领域资料迁入现有可搜索的 knowledge/resources。保留工具名、Schema、返回结构、错误码。**先试点 3 个最长工具**，以真实 Agent 任务证明不回退。
2. **MCP Host / Agent**：确认是否能只按需要将工具定义提供给模型。支持原生 tool search 时使用原生功能；否则在 Host 层考虑 Catalog → Inspect → Execute。**不要直接在 Server 端自造 87 工具动态启停框架**，以免引入连接状态和兼容性问题。

参考：<https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/docs/2026-07-28/develop/clients/client-best-practices.mdx>。

## 4. 推进次序：小 PR、实测、可回退

### PR A（P0）工具注解修复
- 纠正 `publish_kit` readOnly 错误；审计所有只读、仅新增工作区副本、改变编辑器状态、覆盖/删除、外部世界操作工具的注解。
- 将 read-only 与 destructive 分开表达，不用 `!readOnly` 代替 destructive 的实际含义。注解仅是提示，操作授权依旧由硬性守卫实现。
- 增加 registry / tools-list 行为测试：保护正确标注、名称、Schema、工具调用不回退。
- 参考：<https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/>。

### PR B（P1）最快的离线 CI
- PR 上验证 `npm ci`、`npm run build:core`，以及**先在 runner 证明可执行**的 core/host/asset 等无界面测试。
- Windows 原神客户端的 UI/试玩集成留到相关功能改变后的独立验收，不作为普通代码 PR 的阻塞门禁。
- 记录真实 CI 墙钟时长、失败原因，保留最少但足够的检查，不引入昂贵 scheduled matrix。

### PR C（P1）工具 Token 预算
- 先记录实际 `tools/list` 大小、Host 是否全量注入和模型调用基线。
- 只对 `scene_add_components`、`playtest_start`、`scene_add_client_ui` 试点将长例子/背景迁到按需文档；工具本身仍能正确被选中并填好参数。
- 对 3 条典型 Agent 任务做前后 A/B：找物件→搭场景、写图→试玩、UI 创建→预览；测调用成功、误选/重试、Token。新方案不得依靠删掉关键约束来“压缩”。

### PR D（P2）函数级 CC 与模块边界
- 使用成熟 AST 工具或项目已有 TypeScript 依赖获取**函数级**圈复杂度 CC、静态 SCC、变更频度基线；集中处理真正复杂且频繁变更的函数。
- `scene-save` 优先按场景、UI、造物、经济等责任做**局部抽离**，保持公开函数兼容；不要一次改写全部 3,000 行。
- 回归硬约束：原 GIL 未知 protobuf 字段保留、GUID/索引不漂移、场景输出可读、事务恢复与字节精确回滚不退化；失败能回退原模块。

### 独立资产工作（P1）
- 官方 7.2 UI 图片目录按**增量**补录，查明与现有 1,590 张的差量；只加入有官方截图、ID 与来源记录的条目，不猜测 ID、不重写旧图片。
- 独立于架构优化交付，不能为等待圈复杂度整改而延误资产可用性。

## 5. 验收与非目标

**必须提供实测证据**：PR 前后真实 tool-list Token/字节数、工具成功率、metadata 注解测试、CI 时长、函数级 CC 热点、SCC 结果、存档与资产回归。不能只宣称“架构已经优化”。

**不做**：推倒重写、微服务化、新引入复杂编排层、无意义统一抽象、将所有函数 CC 强行压到 0、全仓格式化大 diff、要求真机试玩阻塞所有小 PR。

## 6. Agent 使用注意

- 使用 Fork 的**独立本地副本**和独立 `MILIASTRA_WORKSPACE`，默认 `MILIASTRA_ALLOW_LIVE_WRITE=0`；`git pull` / 安装后重启 MCP 会话。
- 开始任务按需读 `miliastra://workflow`、`miliastra://platform-rules`，再检索目标 knowledge；不要在每轮对话全文加载全部资源和 87 个工具的说明。
- 合并任何行为代码前，至少通过对应离线回归。涉及导入、UI 或试玩链路的改动，补模拟/真机证据并明确未测部分。
