# 其他 Feature 汇总

> 所有未在前面文档中 deep dive 的 feature，按重要性排序。

## Feature 一览表

| # | Feature | 位置 | 复杂度 | Feature Flag | 简介 |
|---|---------|------|--------|-------------|------|
| 1 | [Web 界面](#1-web-界面) | `web/` | 大 | 多个 | 完整的 Web UI（Next.js） |
| 2 | [IDE Bridge](#2-ide-bridge) | `src/bridge/` | 大 | `BRIDGE_MODE` | VS Code/JetBrains 集成协议 |
| 3 | [Voice 语音系统](#3-voice-语音系统) | `src/voice/` | 中 | `VOICE_MODE` | 语音输入/输出 |
| 4 | [Buddy 伴侣精灵](#4-buddy-伴侣精灵) | `src/buddy/` | 中 | `BUDDY` | 动画伴侣精灵 |
| 5 | [Vim 模式](#5-vim-模式) | `src/vim/` | 中 | 无 | 完整的 Vim 状态机 |
| 6 | [Keybinding 系统](#6-keybinding-系统) | `src/keybindings/` | 中 | 无 | 自定义键盘快捷键 |
| 7 | [x402 支付协议](#7-x402-支付协议) | `src/services/x402/` | 中 | 无 | USDC on Base 区块链支付 |
| 8 | [Output Styles](#8-output-styles) | `src/outputStyles/` | 小 | 无 | 自定义输出风格 |
| 9 | [Magic Docs](#9-magic-docs) | `src/services/MagicDocs/` | 中 | 无 | 自动更新文档 |
| 10 | [Auto Dream](#10-auto-dream) | `src/services/autoDream/` | 中 | `KAIROS_DREAM` | 后台记忆整理 |
| 11 | [Prompt Suggestion](#11-prompt-suggestion--speculation) | `src/services/PromptSuggestion/` | 中 | `PROACTIVE` | 推测执行 |
| 12 | [Extract Memories](#12-extract-memories) | `src/services/extractMemories/` | 中 | `EXTRACT_MEMORIES` | 自动提取记忆 |
| 13 | [Agent Summary](#13-agent-summary) | `src/services/AgentSummary/` | 小 | 无 | Agent 进度摘要 |
| 14 | [Session Memory](#14-session-memory) | `src/services/SessionMemory/` | 中 | 无 | 会话级记忆 |
| 15 | [Team Memory Sync](#15-team-memory-sync) | `src/services/teamMemorySync/` | 中 | `TEAMMEM` | 团队记忆同步 |
| 16 | [Tips 系统](#16-tips-系统) | `src/services/tips/` | 小 | 无 | 上下文提示 |
| 17 | [Notification Hooks](#17-notification-hooks) | `src/hooks/notifs/` | 中 | 多个 | 16 个专用通知 hook |
| 18 | [Upstream Proxy](#18-upstream-proxy) | `src/upstreamproxy/` | 中 | CCR env | CCR 容器代理 |
| 19 | [Remote Managed Settings](#19-remote-managed-settings) | `src/services/remoteManagedSettings/` | 中 | 多个 | 远程配置同步 |
| 20 | [Policy Limits](#20-policy-limits) | `src/services/policyLimits/` | 小 | 无 | 组织策略限制 |
| 21 | [Migrations](#21-migrations) | `src/migrations/` | 小 | 多个 | 配置 schema 迁移 |
| 22 | [Native TS 模块](#22-native-ts-模块) | `src/native-ts/` | 小 | 无 | 原生性能模块 |
| 23 | [Bootstrap 状态](#23-bootstrap-状态) | `src/bootstrap/` | 大 | `FILE_PERSISTENCE` | 会话/项目状态管理 |
| 24 | [MCP Server](#24-mcp-server) | `mcp-server/` | 中 | 多个 | 独立 MCP Server |
| 25 | [Docker 支持](#25-docker-支持) | `docker/` | 小 | 无 | 容器化部署 |
| 26 | [Bundled Skills](#26-bundled-skills) | `src/skills/bundled/` | 中 | 多个 | 17 个内置 Skill |
| 27 | [Bundled Plugins](#27-bundled-plugins) | `src/plugins/bundled/` | 小 | 无 | 内置插件 |

---

## 1. Web 界面

**位置**：`web/`（143 个 TSX 文件，~600KB）

基于 Next.js 的完整 Web UI，是 claude.ai/code 的前端。包含：
- 聊天界面
- 设置面板
- 多用户协作
- 命令面板
- 移动端支持
- 文件查看器
- 工具管理
- 导出功能

**感兴趣的话**：可以 deep dive 它的组件架构、状态管理、与 CLI 的通信协议。

---

## 2. IDE Bridge

**位置**：`src/bridge/`（34 个文件，~500KB+）

Claude Code 与 VS Code / JetBrains IDE 之间的双向通信协议。

**核心协议**：
- `WorkSecret`：包含 session token、API base URL、git/auth 源、MCP 配置、环境变量
- `BridgeWorkerType`：`claude_code` 或 `claude_code_assistant`
- `SpawnMode`：`single-session` / `worktree` / `same-dir`
- 认证：session ingress token + 环境注册
- Session 活动追踪：tool_start, text, result, error

**关键文件**：
| 文件 | 大小 | 功能 |
|------|------|------|
| `bridgeMain.ts` | 115KB | 主桥接逻辑 |
| `replBridge.ts` | 100KB+ | REPL 桥接编排 |
| `remoteBridgeCore.ts` | 39KB | 远程桥接核心 |
| `initReplBridge.ts` | 23KB | 初始化 |
| `bridgeApi.ts` | 18KB | API 层 |
| `codeSessionApi.ts` | ~10KB | 代码 session 管理 |
| `jwtUtils.ts` | ~5KB | JWT 认证 |
| `trustedDevice.ts` | ~5KB | 设备信任 |

**Feature flags**: `BRIDGE_MODE`, `DIRECT_CONNECT`, `CCR_MIRROR`, `CCR_REMOTE_SETUP`, `CCR_AUTO_CONNECT`

**感兴趣的话**：可以 deep dive 协议消息格式、session 管理、权限代理机制。

---

## 3. Voice 语音系统

**位置**：`src/voice/`、`src/services/voice.ts`、`src/services/voiceStreamSTT.ts`

语音输入/输出支持，允许免手操作。

**组件**：
- Voice service：核心语音处理
- STT streaming：语音转文字 streaming
- Key terms：领域词汇表
- Hooks：`useVoice`, `useVoiceEnabled`, `useVoiceIntegration`
- 命令：`/voice`

**条件**：需要 Anthropic OAuth（不支持 API key / Bedrock / Vertex），GrowthBook gate `tengu_amber_quartz_disabled`。

**Feature flag**: `VOICE_MODE`

---

## 4. Buddy 伴侣精灵

**位置**：`src/buddy/`（6 个文件，~100KB）

用户输入框旁边的动画伴侣精灵，会偶尔评论。

**精灵系统**：
- **18 种物种**：duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk
- **5 种稀有度**：common (60%), uncommon (25%), rare (10%), epic (4%), legendary (1%)
- **6 种眼型**：`·, ✦, ×, ◉, @, °`
- **8 种帽子**：none, crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **5 种属性**：DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK

**设计**：外观由 `hash(userId)` 确定性生成，用户无法手动编辑到 legendary。首次"孵化"后由模型生成个性，存储在配置中。

**Feature flag**: `BUDDY`

---

## 5. Vim 模式

**位置**：`src/vim/`（5 个文件）

完整的 Vim 模式状态机，用于输入框编辑。

**能力**：
- INSERT / NORMAL 模式切换
- 移动：h/j/k/l, w/b/e, 行首/行尾, find (f/F/t/T)
- 操作符：delete, change, yank
- 文本对象：words, quotes, brackets, braces
- Dot-repeat 通过 RecordedChange 追踪
- Register 支持（yank/paste）

---

## 6. Keybinding 系统

**位置**：`src/keybindings/`（14 个文件）

可自定义的键盘快捷键系统。

**能力**：
- 通过 `~/.claude/keybindings.json` 自定义
- 支持 Ctrl+, Cmd+, Alt+, Shift+, chord 组合
- 保留快捷键保护（防止覆盖系统快捷键）
- React Context 分发
- 验证和显示格式化

---

## 7. x402 支付协议

**位置**：`src/services/x402/`（6 个文件）

处理 HTTP 402 Payment Required 响应，通过 USDC on Base 区块链支付。

**能力**：
- 私钥生成 + 安全存储
- 支付需求解析/验证
- 每 session 消费限制 + 成本追踪
- 每次支付上限
- 网络选择（Base mainnet / testnet）
- Coinbase x402 协议支持

---

## 8. Output Styles

**位置**：`src/outputStyles/loadOutputStylesDir.ts`

自定义输出风格，通过 `.claude/output-styles/` 中的 markdown 文件定义。

**Frontmatter**：
```yaml
name: concise
description: Brief, direct responses
keep-coding-instructions: true
force-for-plugin: false
```

---

## 9. Magic Docs

**位置**：`src/services/MagicDocs/`（2 个文件）

自动更新的文档。检测带有 `# MAGIC DOC: [title]` 标题的 markdown 文件，在后台通过 forked subagent 自动更新内容。

---

## 10. Auto Dream

**位置**：`src/services/autoDream/`（4 个文件）

后台记忆整理。将完整的对话日志压缩为摘要块，使用 mtime 锁防止并发。

**Feature flag**: `KAIROS_DREAM`

---

## 11. Prompt Suggestion / Speculation

**位置**：`src/services/PromptSuggestion/`

基于对话上下文预测用户的下一步操作，生成建议 prompt 加速工作流。

**Feature flag**: `PROACTIVE`

---

## 12. Extract Memories

**位置**：`src/services/extractMemories/`（2 个文件）

在 query loop 结束时自动从对话中提取持久化记忆。通过 forked agent 执行，写入 `~/.claude/projects/<path>/memory/`。包含密钥扫描防止泄露。

**Feature flag**: `EXTRACT_MEMORIES`, `TEAMMEM`

---

## 13. Agent Summary

**位置**：`src/services/AgentSummary/`（1 个文件）

Coordinator 模式下每 ~30 秒为子 Agent 生成 1-2 句进度摘要，用于 UI 展示。

---

## 14. Session Memory

**位置**：`src/services/SessionMemory/`（3 个文件）

会话级别的上下文持久化，跨 turn 保持。与 `/remember` 命令和 task 系统集成。

---

## 15. Team Memory Sync

**位置**：`src/services/teamMemorySync/`（5 个文件）

团队成员之间同步记忆文件。包含文件系统 watcher、密钥扫描器、加密同步。

**Feature flag**: `TEAMMEM`

---

## 16. Tips 系统

**位置**：`src/services/tips/`（3 个文件）

定时向用户展示上下文相关的使用提示。有历史追踪防止重复。

---

## 17. Notification Hooks

**位置**：`src/hooks/notifs/`（16 个文件）

专门的通知 hook，各司其职：

| Hook | 功能 |
|------|------|
| `useAutoModeUnavailableNotification` | Auto mode 不可用通知 |
| `useDeprecationWarningNotification` | 废弃警告 |
| `useFastModeNotification` | Fast mode 状态 |
| `useIDEStatusIndicator` | IDE 连接状态（21KB） |
| `useLspInitializationNotification` | LSP 初始化状态 |
| `useMcpConnectivityStatus` | MCP server 连接监控 |
| `useModelMigrationNotifications` | 模型迁移提示 |
| `usePluginAutoupdateNotification` | 插件更新 |
| `usePluginInstallationStatus` | 插件安装状态 |
| `useRateLimitWarningNotification` | 速率限制警告 |
| `useSettingsErrors` | 设置错误 |
| `useStartupNotification` | 启动通知 |
| `useTeammateShutdownNotification` | Teammate 关闭提醒 |

---

## 18. Upstream Proxy

**位置**：`src/upstreamproxy/`（2 个文件，~25KB）

CCR（Claude Cloud Runtime）容器内的代理配置：
- Session token 从 `/run/ccr/session_token` 读取
- `prctl(PR_SET_DUMPABLE, 0)` 防止 ptrace
- CA 证书下载 + 系统 bundle 拼接
- CONNECT→WebSocket relay
- NO_PROXY 配置（localhost、RFC1918、IMDS、anthropic.com）

---

## 19. Remote Managed Settings

**位置**：`src/services/remoteManagedSettings/`（4 个文件）

从服务器同步配置，支持离线缓存和状态追踪。

**Feature flags**: `DOWNLOAD_USER_SETTINGS`, `UPLOAD_USER_SETTINGS`

---

## 20. Policy Limits

**位置**：`src/services/policyLimits/`（2 个文件）

组织级别的使用限制（token 预算、工具使用等），服务端配置，强制执行。

---

## 21. Migrations

**位置**：`src/migrations/`（11 个文件）

配置 schema 迁移，处理版本升级：
- 模型版本升级：Fennec→Opus, Sonnet 1M→4.5→4.6
- 功能废弃
- 设置合并（auto-updates、permissions、MCP）
- 定价/功能变更后重置默认值

---

## 22. Native TS 模块

**位置**：`src/native-ts/`

性能关键的原生模块：
- `color-diff/` — 颜色差异计算
- `file-index/` — 文件索引
- `yoga-layout/` — Yoga 布局引擎（终端 flexbox）

---

## 23. Bootstrap 状态

**位置**：`src/bootstrap/state.ts`（1,760 行）

管理 session 和项目状态：
- Session ID 生成
- 项目根目录检测
- CWD 追踪
- Git root 检测
- Worktree 状态
- 终端备份恢复

**Feature flag**: `FILE_PERSISTENCE`

---

## 24. MCP Server

**位置**：`mcp-server/src/`（3 个文件，~38KB）

独立的 MCP server，让其他 MCP 客户端（如 AI 助手）可以通过 MCP 协议访问这个代码库。

---

## 25. Docker 支持

**位置**：`docker/`

标准的容器化部署支持：Dockerfile + docker-compose。

---

## 26. Bundled Skills

**位置**：`src/skills/bundled/`（17 个文件，92KB）

| Skill | 大小 | 功能 |
|-------|------|------|
| `scheduleRemoteAgents.ts` | 19KB | Agent 定时调度 |
| `skillify.ts` | 17KB | 从工作流创建新 Skill |
| `updateConfig.ts` | 17KB | 编程式配置管理 |
| `keybindings.ts` | 10KB | 键盘快捷键管理 |
| `batch.ts` | ~5KB | 批量文件操作 |
| `claudeApi.ts` | ~5KB | Claude API 访问 |
| `loop.ts` | ~5KB | 循环/定时执行 |
| `simplify.ts` | ~3KB | 代码简化 |
| `debug.ts` | ~3KB | 调试工作流 |
| `stuck.ts` | ~3KB | 卡住状态恢复 |
| `verify.ts` | ~3KB | 代码验证 |
| `remember.ts` | ~2KB | 记忆持久化 |
| `loremIpsum.ts` | ~1KB | 占位文本生成 |
| `claudeInChrome.ts` | ~3KB | Chrome 扩展集成 |

---

## 27. Bundled Plugins

**位置**：`src/plugins/bundled/`

内置插件代码，随 Claude Code 打包发布。

---

## 全部 Feature Flags（87 个）

<details>
<summary>点击展开完整 Feature Flag 列表</summary>

| Flag | 功能 |
|------|------|
| `ABLATION_BASELINE` | A/B 测试基线 |
| `AGENT_MEMORY_SNAPSHOT` | Agent 记忆快照 |
| `AGENT_TRIGGERS_REMOTE` | 远程 Agent 触发 |
| `AGENT_TRIGGERS` | 本地 Agent 触发 |
| `ALLOW_TEST_VERSIONS` | 测试模型版本 |
| `ANTI_DISTILLATION_CC` | 反蒸馏（Chrome） |
| `AUTO_THEME` | 自动主题 |
| `AWAY_SUMMARY` | 离开状态摘要 |
| `BASH_CLASSIFIER` | Bash 命令分类器 |
| `BG_SESSIONS` | 后台 session |
| `BREAK_CACHE_COMMAND` | 缓存破坏命令 |
| `BRIDGE_MODE` | IDE 桥接 |
| `BUDDY` | 伴侣精灵 |
| `BUILDING_CLAUDE_APPS` | Claude Apps 构建模式 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置探索/规划 Agent |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC 运行器 |
| `CACHED_MICROCOMPACT` | 缓存微压缩 |
| `CCR_AUTO_CONNECT` | CCR 自动连接 |
| `CCR_MIRROR` | CCR 镜像 |
| `CCR_REMOTE_SETUP` | CCR 远程设置 |
| `CHICAGO_MCP` | Chicago MCP server |
| `COMMIT_ATTRIBUTION` | Commit 归属追踪 |
| `COMPACTION_REMINDERS` | 压缩提醒 |
| `CONNECTOR_TEXT` | 文本连接器 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `COORDINATOR_MODE` | 多 Agent 协调 |
| `COWORKER_TYPE_TELEMETRY` | 协作者类型遥测 |
| `DAEMON` | 守护进程模式 |
| `DIRECT_CONNECT` | IDE 直连 |
| `DOWNLOAD_USER_SETTINGS` | 下载用户设置 |
| `DUMP_SYSTEM_PROMPT` | 调试：导出 system prompt |
| `ENHANCED_TELEMETRY_BETA` | 增强分析 |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性 Skill 搜索 |
| `EXTRACT_MEMORIES` | 自动记忆提取 |
| `FILE_PERSISTENCE` | 文件状态持久化 |
| `FORK_SUBAGENT` | 子 Agent fork |
| `HARD_FAIL` | 硬失败模式 |
| `HISTORY_PICKER` | 对话历史选择器 |
| `HISTORY_SNIP` | 历史裁剪 |
| `HOOK_PROMPTS` | Hook 提示 |
| `IS_LIBC_GLIBC` | glibc 检测 |
| `IS_LIBC_MUSL` | musl 检测 |
| `KAIROS_BRIEF` | Kairos 简报 |
| `KAIROS_CHANNELS` | Kairos 频道 |
| `KAIROS_DREAM` | Kairos 记忆整理 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook 集成 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 |
| `KAIROS` | Kairos 基础 |
| `LODESTONE` | Lodestone（内部） |
| `MCP_RICH_OUTPUT` | MCP 富输出 |
| `MCP_SKILLS` | MCP Skills |
| `MEMORY_SHAPE_TELEMETRY` | 记忆形态遥测 |
| `MESSAGE_ACTIONS` | 消息操作 |
| `MONITOR_TOOL` | 监控工具 |
| `NATIVE_CLIENT_ATTESTATION` | 原生客户端认证 |
| `NATIVE_CLIPBOARD_IMAGE` | 原生剪贴板图片 |
| `NEW_INIT` | 新初始化流程 |
| `OVERFLOW_TEST_TOOL` | 溢出测试工具 |
| `PERFETTO_TRACING` | 性能追踪 |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式 |
| `PROACTIVE` | 主动建议 |
| `PROMPT_CACHE_BREAK_DETECTION` | 缓存破坏检测 |
| `QUICK_SEARCH` | 快速搜索 UI |
| `REACTIVE_COMPACT` | 响应式压缩 |
| `REVIEW_ARTIFACT` | Artifact 审查 |
| `RUN_SKILL_GENERATOR` | Skill 生成器 |
| `SELF_HOSTED_RUNNER` | 自托管运行器 |
| `SHOT_STATS` | 截图统计 |
| `SKILL_IMPROVEMENT` | Skill 改进 |
| `SLOW_OPERATION_LOGGING` | 慢操作日志 |
| `SSH_REMOTE` | SSH 远程执行 |
| `STREAMLINED_OUTPUT` | 精简输出 |
| `TEAMMEM` | 团队记忆 |
| `TEMPLATES` | 模板系统 |
| `TERMINAL_PANEL` | 终端面板 |
| `TOKEN_BUDGET` | Token 预算 |
| `TORCH` | Torch（内部） |
| `TRANSCRIPT_CLASSIFIER` | Transcript 分类器 |
| `TREE_SITTER_BASH_SHADOW` | Tree-sitter Bash shadow |
| `TREE_SITTER_BASH` | Tree-sitter Bash |
| `UDS_INBOX` | Unix Domain Socket 消息 |
| `ULTRAPLAN` | 超级规划 |
| `ULTRATHINK` | 超级思考 |
| `UNATTENDED_RETRY` | 无人值守重试 |
| `UPLOAD_USER_SETTINGS` | 上传用户设置 |
| `VERIFICATION_AGENT` | 验证 Agent |
| `VOICE_MODE` | 语音模式 |
| `WEB_BROWSER_TOOL` | Web 浏览器工具 |
| `WORKFLOW_SCRIPTS` | 工作流脚本 |

</details>

---

## Kairos 系统（值得单独关注）

Kairos 是一个横跨多个 feature flag 的子系统：

| Flag | 功能 |
|------|------|
| `KAIROS` | 基础：后台 Agent 模式 |
| `KAIROS_DREAM` | 记忆整理/梦境模式 |
| `KAIROS_BRIEF` | 简报生成 |
| `KAIROS_CHANNELS` | 频道系统 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook 集成 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 |

这似乎是 Claude Code 的"后台自主 Agent"系统，但大部分功能处于 feature flag 后面。

---

## 有趣的 `src/utils/` 子目录

| 目录 | 功能 |
|------|------|
| `computerUse/` | 计算机使用工具支持 |
| `sandbox/` | 沙箱执行 |
| `secureStorage/` | 安全凭据存储 |
| `claudeInChrome/` | Chrome 扩展集成 |
| `teleport/` | 远程执行工具 |
| `deepLink/` | 深度链接 |
| `ultraplan/` | 超级规划工具 |
| `suggestions/` | 建议系统 |
| `github/` | GitHub 集成 |
| `dxt/` | DXT 调试工具 |
| `nativeInstaller/` | 原生组件安装器 |
| `processUserInput/` | 用户输入处理管道 |
