# Claude Code 源码学习文档

> 深入理解 Claude Code 的内部实现。

## 文档结构

| # | 文档 | 内容 | 复杂度 |
|---|------|------|--------|
| 00 | [架构总览](00-architecture-overview.md) | 高层架构、目录结构、模块关系、设计模式 | 入门 |
| 01 | [核心管道](01-core-pipeline.md) | QueryEngine、Query Loop、API 调用、重试、上下文管理 | 核心 |
| 02 | [工具系统](02-tool-system.md) | 工具定义、注册、执行、权限、42 个工具分类 | 核心 |
| 03 | [命令系统](03-command-system.md) | Slash 命令类型、注册、加载、100+ 命令分类 | 基础 |
| 04 | [UI 和状态管理](04-ui-and-state.md) | React/Ink、AppState、Hooks、渲染优化 | 基础 |
| 05 | [任务系统](05-task-system.md) | 后台任务、Shell/Agent/Remote 任务、GC 策略 | 进阶 |
| 06a | [Sub-agent 系统](06a-subagent.md) | AgentTool、单 Agent 派生、Fork、工具过滤 | 进阶 |
| 06b | [Agent Team 系统](06b-agent-team.md) | **Coordinator、Swarm、三种后端、消息协议** | 高级 |
| 07 | [服务层和基础设施](07-services-and-infra.md) | API、MCP、OAuth、插件、记忆概览、远程执行 | 进阶 |
| 08 | [其他 Feature 汇总](08-other-features.md) | 未 deep dive 的 feature 索引 + 87 个 feature flags | 索引 |
| 09 | [记忆系统全景](09-memory-systems.md) | **Auto Dream、Extract Memories、Session Memory、Team Sync** | 高级 |
| 10 | [通知与提示系统](10-notifications-and-tips.md) | 16 个通知 Hook、优先级队列、65 个旋转 Tips | 进阶 |
| 11 | [Voice & Buddy](11-voice-and-buddy.md) | 语音 STT 管道、伴侣精灵生成与动画 | 进阶 |
| 12 | [基础设施](12-infrastructure.md) | Bootstrap 状态、Remote Settings、MCP Server、Docker | 进阶 |
| 13 | [远程 Agent 执行](13-remote-execution.md) | **CCR 云端执行、Triggers 定时调度、Ultraplan、BYOC** | 高级 |
| 14 | [Feature Flags 专题](14-feature-flags.md) | **87 个 flags 分类、功能组合分析** | 索引 |
| 15 | [KAIROS 系统](15-kairos.md) | **后台自主 Agent：Brief、Dream、Channels、Cron、Daemon** | 高级 |

## 建议阅读顺序

**快速了解**：00 → 01（前半部分）→ 06b

**全面学习**：00 → 01 → 02 → 03 → 04 → 05 → 06a → 06b → 07 → 09 → 10 → 11 → 12 → 08

**专题深入**：
- Agent 系统：06a → 06b → 05
- 记忆系统：09（四层记忆架构）
- 工具开发：02
- UI 开发：04 → 10（通知）
- API 集成：01 + 07
- 语音 & 伴侣：11
- 部署：12（Bootstrap + Docker + MCP Server）
- Feature 发现：08（全部 feature 索引）

## 关键文件速查

| 功能 | 入口文件 |
|------|---------|
| CLI 入口 | `src/entrypoints/cli.tsx` → `src/main.tsx` |
| 核心引擎 | `src/QueryEngine.ts` + `src/query.ts` |
| 工具定义 | `src/Tool.ts` + `src/tools.ts` |
| 工具实现 | `src/tools/{ToolName}/{ToolName}.ts` |
| 命令注册 | `src/commands.ts` |
| 主 UI | `src/screens/REPL.tsx` |
| 全局状态 | `src/state/AppStateStore.ts` |
| Sub-agent | `src/tools/AgentTool/AgentTool.ts` |
| Agent Team | `src/coordinator/coordinatorMode.ts` |
| Team 工具 | `src/tools/TeamCreateTool/` + `SendMessageTool/` |
| Swarm 后端 | `src/utils/swarm/backends/` |
| 消息传递 | `src/tools/SendMessageTool/SendMessageTool.ts` |
| API 客户端 | `src/services/api/claude.ts` |
| MCP 客户端 | `src/services/mcp/client.ts` |
| 记忆核心 | `src/memdir/memdir.ts` |
| 记忆提取 | `src/services/extractMemories/` |
| 记忆整理 | `src/services/autoDream/` |
| Session 记忆 | `src/services/SessionMemory/` |
| 团队记忆 | `src/services/teamMemorySync/` |
| 通知系统 | `src/context/notifications.tsx` + `src/hooks/notifs/` |
| Tips 系统 | `src/services/tips/` |
| Voice | `src/services/voiceStreamSTT.ts` + `src/hooks/useVoice.ts` |
| Buddy | `src/buddy/CompanionSprite.tsx` + `src/buddy/companion.ts` |
| Bootstrap | `src/bootstrap/state.ts` |
| Remote Settings | `src/services/remoteManagedSettings/` |
| MCP Server | `mcp-server/src/server.ts` |
| Feature gating | `src/utils/agentSwarmsEnabled.ts` |
