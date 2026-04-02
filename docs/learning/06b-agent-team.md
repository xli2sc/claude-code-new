# Agent Team 系统 — 多 Agent 协调

> Coordinator 模式：一个主 Agent 指挥多个 Worker Agent 并行协作。

## 概览

Agent Team 是 Claude Code 最复杂的子系统。它允许一个 **Coordinator (Leader)** 创建一个 **Team**，团队中的 **Worker (Teammate)** 可以并行执行不同的任务，通过**文件邮箱**异步通信。

```mermaid
graph TB
    subgraph "Coordinator"
        LEADER[Leader Agent<br/>统一编排]
    end

    subgraph "Team Workers"
        W1[Worker 1<br/>Researcher]
        W2[Worker 2<br/>Implementer]
        W3[Worker 3<br/>Tester]
    end

    subgraph "执行后端"
        IP[InProcess<br/>同进程]
        TMUX[Tmux<br/>终端分屏]
        ITERM[iTerm2<br/>原生分栏]
    end

    subgraph "通信"
        MAIL[文件邮箱<br/>异步投递]
    end

    LEADER -->|AgentTool| W1
    LEADER -->|AgentTool| W2
    LEADER -->|AgentTool| W3
    LEADER <--> MAIL
    W1 <--> MAIL
    W2 <--> MAIL
    W3 <--> MAIL
    W1 --- IP
    W2 --- TMUX
    W3 --- ITERM
```

## 与 Sub-agent 的区别

| 特性 | Sub-agent (06a) | Agent Team (本文档) |
|------|-----------------|-------------------|
| 关系 | 1:1 父子 | 1:N Coordinator → Workers |
| 通信 | 直接返回结果 | 文件邮箱异步通信 |
| 生命周期 | 单次任务 | 持续协作直到 TeamDelete |
| Worker 间通信 | 不支持 | SendMessage 广播 |
| 关闭协议 | 直接终止 | 优雅关闭（请求→确认） |
| 执行后端 | InProcess 或后台 task | InProcess / Tmux / iTerm2 |
| Feature gate | 无（核心功能） | `COORDINATOR_MODE` + `isAgentSwarmsEnabled()` |

## Feature Gating

Agent Team 有**两层门控**：

### 层 1: Agent Swarms 启用 (`src/utils/agentSwarmsEnabled.ts`)

```typescript
function isAgentSwarmsEnabled(): boolean {
  // Anthropic 内部: 始终启用
  if (process.env.USER_TYPE === 'ant') return true

  // 外部: 需要显式 opt-in
  if (!isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS)
      && !isAgentTeamsFlagSet()) return false

  // GrowthBook killswitch
  if (!getFeatureValue('tengu_amber_flint', true)) return false

  return true
}
```

### 层 2: Coordinator 模式 (`src/coordinator/coordinatorMode.ts`)

```typescript
function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

两者的关系：Agent Swarms 控制 team 工具是否可用，Coordinator Mode 控制是否使用专门的 Coordinator system prompt。

## 架构详解

```mermaid
graph TB
    subgraph "入口层"
        GATE1[isAgentSwarmsEnabled<br/>工具可用性]
        GATE2[isCoordinatorMode<br/>Coordinator prompt]
    end

    subgraph "工具层"
        TCT[TeamCreateTool<br/>创建团队]
        TDT[TeamDeleteTool<br/>删除团队]
        SMT[SendMessageTool<br/>650 行，完整协议]
        AT[AgentTool<br/>生成 Worker]
    end

    subgraph "执行后端"
        REG[Backend Registry<br/>自动检测可用后端]
        IPB[InProcessBackend<br/>同进程 + AsyncLocalStorage]
        TMB[TmuxBackend<br/>tmux 分屏 21.5KB]
        ITB[ITermBackend<br/>iTerm2 分栏 12.9KB]
    end

    subgraph "Swarm 基础设施"
        RUNNER[inProcessRunner.ts<br/>核心执行循环 53KB]
        HELPERS[teamHelpers.ts<br/>团队文件 I/O 21.4KB]
        PERM_SYNC[permissionSync.ts<br/>权限同步 26.5KB]
        LAYOUT[teammateLayoutManager.ts<br/>颜色分配]
        RECON[reconnection.ts<br/>断线恢复]
    end

    subgraph "状态层"
        TC[AppState.teamContext]
        INBOX[AppState.inbox]
        TASKS[AppState.tasks]
        TF[TeamFile<br/>磁盘持久化]
        MB[Mailbox Files<br/>消息文件]
    end

    GATE1 --> TCT
    GATE1 --> TDT
    GATE1 --> SMT
    GATE2 --> AT

    TCT --> HELPERS
    HELPERS --> TF
    TCT --> TC

    AT --> REG
    REG --> IPB
    REG --> TMB
    REG --> ITB

    IPB --> RUNNER
    SMT --> MB
    SMT --> INBOX

    PERM_SYNC --> TC
```

## 三种执行后端

### 后端检测 (`src/utils/swarm/backends/registry.ts`)

```mermaid
flowchart TB
    START[检测可用后端] --> TMUX_IN{在 tmux 内?}
    TMUX_IN -->|是| USE_TMUX[使用 Tmux Backend]
    TMUX_IN -->|否| ITERM{在 iTerm2 内?}

    ITERM -->|是| IT2_CLI{it2 CLI 可用?}
    IT2_CLI -->|是| USE_ITERM[使用 iTerm2 Backend]
    IT2_CLI -->|否| SETUP_NEEDED[提示安装 it2 CLI]

    ITERM -->|否| TMUX_AVAIL{tmux 已安装?}
    TMUX_AVAIL -->|是| USE_TMUX_EXT[使用 Tmux Backend<br/>外部 socket 模式]
    TMUX_AVAIL -->|否| FALLBACK[InProcess Backend<br/>始终可用]
```

### InProcess Backend（同进程）

**位置**：`src/utils/swarm/backends/InProcessBackend.ts`（10.5KB）

```mermaid
graph TB
    subgraph "Node.js 进程"
        LEADER[Leader Agent<br/>主线程]
        ALS1[AsyncLocalStorage<br/>Worker 1 上下文]
        ALS2[AsyncLocalStorage<br/>Worker 2 上下文]
        SHARED[共享资源<br/>API Client / MCP 连接]
    end

    LEADER --> ALS1
    LEADER --> ALS2
    ALS1 --> SHARED
    ALS2 --> SHARED
```

**特点**：
- 与 Leader **共享进程** — 零进程开销
- 通过 `AsyncLocalStorage` 实现上下文隔离
- 共享 API 客户端和 MCP 连接
- 通过 `AbortController` 终止
- 消息通过文件邮箱（与其他后端一致）
- **始终可用**，无系统依赖

### Tmux Backend（终端分屏）

**位置**：`src/utils/swarm/backends/TmuxBackend.ts`（21.5KB）

```
┌──────────────────┬──────────────────┐
│                  │  Worker 1        │
│                  │  (红色边框)       │
│  Leader          ├──────────────────┤
│  (30% 宽度)      │  Worker 2        │
│                  │  (蓝色边框)       │
│                  ├──────────────────┤
│                  │  Worker 3        │
│                  │  (绿色边框)       │
└──────────────────┴──────────────────┘
```

**特点**：
- Leader 占左侧 30%，Workers 占右侧 70%
- 每个 Worker 是独立的 tmux pane（独立进程）
- 彩色边框区分不同 Worker
- Pane 标题显示 Worker 名称
- 支持隐藏/显示 pane
- Pane 创建使用锁防止并行生成的竞争条件
- 可以在 tmux 外运行（使用外部 socket 创建新 session）

### iTerm2 Backend（原生分栏）

**位置**：`src/utils/swarm/backends/ITermBackend.ts`（12.9KB）

**特点**：
- 使用 `it2` CLI 进行原生分屏
- 第一个 Worker：从 Leader 垂直分割(-v)
- 后续 Worker：从上一个 Worker 水平分割
- 无法隐藏/显示 pane
- 有 dead pane 恢复机制（用户关闭 pane 后重试）

## 团队生命周期

### 创建团队 (TeamCreateTool)

```mermaid
sequenceDiagram
    participant L as Leader
    participant TCT as TeamCreateTool
    participant DISK as 磁盘
    participant STATE as AppState

    L->>TCT: TeamCreate({ team_name: "refactor-team" })
    TCT->>TCT: 1. 校验名称唯一
    TCT->>TCT: 2. 生成 leadAgentId = "team-lead@refactor-team"
    TCT->>DISK: 3. 创建 TeamFile → ~/.claude/teams/refactor-team/config.json
    TCT->>DISK: 4. 创建 task 目录 → ~/.claude/tasks/refactor-team/
    TCT->>STATE: 5. 注册 teamContext
    TCT->>TCT: 6. 注册 session 清理 hook
    TCT->>L: { team_name, team_file_path, lead_agent_id }
```

### TeamFile 结构（磁盘持久化）

```typescript
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  hiddenPaneIds?: string[]           // tmux/iTerm2 隐藏的 pane
  teamAllowedPaths?: TeamAllowedPath[]  // 跨团队编辑权限
  members: Array<{
    agentId: string                  // "researcher@refactor-team"
    name: string                     // "researcher"
    agentType?: string
    model?: string
    prompt?: string
    color?: string                   // 颜色：red/blue/green/yellow/purple/orange/pink/cyan
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: 'tmux' | 'iterm2' | 'in-process'
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

### 生成 Worker

Leader 通过 AgentTool 生成 Worker：

```mermaid
sequenceDiagram
    participant L as Leader
    participant AT as AgentTool
    participant REG as Backend Registry
    participant BE as 执行后端

    L->>AT: Agent({ prompt: "研究 auth 模块", team_name: "refactor-team" })
    AT->>REG: 检测可用后端
    REG->>REG: tmux? iTerm2? 或 InProcess
    REG->>BE: spawn({ name, prompt, color, ... })
    BE->>BE: 创建独立执行环境
    BE->>L: { agentId, taskId, abortController }
```

### 删除团队 (TeamDeleteTool)

**安全检查**：不能删除有活跃 Worker 的团队。

```mermaid
flowchart TB
    DELETE[TeamDelete] --> CHECK{有活跃 Worker?}
    CHECK -->|是| REJECT[拒绝删除<br/>提示先优雅关闭 Worker]
    CHECK -->|否| CLEANUP[执行清理]
    
    CLEANUP --> C1[清理 team 目录]
    CLEANUP --> C2[清理 worktree]
    CLEANUP --> C3[清理颜色分配]
    CLEANUP --> C4[清理 task 列表]
    CLEANUP --> C5[清除 AppState.teamContext]
    CLEANUP --> C6[清除 AppState.inbox]
```

## 消息传递 — SendMessageTool

**位置**：`src/tools/SendMessageTool/SendMessageTool.ts`（650+ 行）

### 消息类型

```mermaid
graph TB
    MSG[SendMessage] --> PLAIN[纯文本消息<br/>日常任务指令]
    MSG --> STRUCT[结构化消息<br/>协议控制]

    STRUCT --> SD_REQ[shutdown_request<br/>请求 Worker 关闭]
    STRUCT --> SD_RESP[shutdown_response<br/>Worker 确认/拒绝关闭]
    STRUCT --> PLAN_APP[plan_approval_response<br/>审批 Worker 计划]
```

### 输入

```typescript
{
  to: string,       // Worker 名称 | "*"（广播）| "bridge:<session-id>"（跨机器）
  message: string | StructuredMessage,
  summary?: string  // 纯文本消息时必填
}
```

### 路由

```mermaid
flowchart TB
    SEND[SendMessage] --> TO{to 目标?}

    TO -->|"researcher"| DIRECT[直接投递<br/>写入 researcher 的 mailbox]
    TO -->|"*"| BROADCAST[广播<br/>写入所有 Worker 的 mailbox<br/>排除发送者]
    TO -->|"bridge:xxx"| REMOTE[跨机器<br/>通过 ReplBridge POST]
    TO -->|跨 session| UDS[Unix Domain Socket<br/>进程间通信]

    DIRECT --> POLL[Worker 在 tool round 间<br/>轮询 mailbox 获取消息]
    BROADCAST --> POLL
```

**核心机制**：所有消息都通过**文件邮箱**投递，不管使用哪种执行后端。这是一个**最终一致**的异步通信模型。

### 关闭协议

```mermaid
sequenceDiagram
    participant L as Leader
    participant W as Worker

    L->>W: SendMessage({ type: "shutdown_request", reason: "任务完成" })

    alt Worker 同意关闭
        W->>L: SendMessage({ type: "shutdown_response", approve: true })
        W->>W: AbortController.abort()
        W->>W: 清理资源，退出
    else Worker 拒绝关闭
        W->>L: SendMessage({ type: "shutdown_response", approve: false, reason: "还在工作" })
        W->>W: 继续执行当前任务
    end
```

### 计划审批

当 Worker 处于 plan mode 时，需要 Leader 审批计划：

```mermaid
sequenceDiagram
    participant W as Worker (plan mode)
    participant L as Leader

    W->>W: EnterPlanMode()
    W->>W: 制定计划...
    W->>L: 等待审批

    alt Leader 批准
        L->>W: SendMessage({ type: "plan_approval_response", approve: true })
        W->>W: 继承 Leader 的权限模式
        W->>W: 开始执行计划
    else Leader 拒绝 + 反馈
        L->>W: SendMessage({ type: "plan_approval_response", approve: false, feedback: "请也考虑 X" })
        W->>W: 根据反馈修改计划
    end
```

## InProcess Teammate Task

### TeammateIdentity

```typescript
type TeammateIdentity = {
  agentId: string,           // "researcher@refactor-team"
  agentName: string,         // "researcher"
  teamName: string,
  color?: string,            // "red" | "blue" | ...
  planModeRequired: boolean,
  parentSessionId: string
}
```

### 完整状态

```typescript
type InProcessTeammateTaskState = TaskStateBase & {
  identity: TeammateIdentity
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition

  // 生命周期控制
  abortController: AbortController           // 终止整个 teammate
  currentWorkAbortController: AbortController // 终止当前 turn
  shutdownRequested: boolean

  // 执行状态
  awaitingPlanApproval: boolean
  permissionMode: PermissionMode             // 独立权限模式
  isIdle: boolean
  onIdleCallbacks: Function[]

  // 数据
  messages: Message[]                        // UI 展示用（上限 50）
  pendingUserMessages: string[]              // 待投递消息队列
  result?: AgentToolResult
}
```

### 内存管理

**`TEAMMATE_MESSAGES_UI_CAP = 50`**

来自生产经验：292 个 agents 的 swarm 会话消耗了 **36.8GB RAM**。UI 消息被严格限制为 50 条，完整历史持久化到磁盘。

### 执行循环 (`inProcessRunner.ts`, 53KB)

```mermaid
flowchart TB
    START[startInProcessTeammate] --> CTX[创建 AsyncLocalStorage 上下文]
    CTX --> AGENT[runAgent 进入 LLM 循环]

    AGENT --> TOOL_ROUND[执行工具]
    TOOL_ROUND --> CHECK_MAIL[检查 mailbox]
    CHECK_MAIL --> PROCESS_MSG[处理消息]
    PROCESS_MSG --> CHECK_SHUTDOWN{shutdown_requested?}
    CHECK_SHUTDOWN -->|否| AGENT
    CHECK_SHUTDOWN -->|是| CLEANUP[清理资源]
    CLEANUP --> IDLE[标记 idle<br/>通知 Leader]
```

## 权限同步 (`src/utils/swarm/permissionSync.ts`, 26.5KB)

Leader 和 Worker 之间需要同步权限状态：

```mermaid
graph TB
    subgraph "Leader"
        L_PERM[权限上下文]
        L_BRIDGE[leaderPermissionBridge.ts<br/>Leader 端桥接]
    end

    subgraph "Worker"
        W_PERM[独立权限模式]
        W_POLLER[useSwarmPermissionPoller<br/>权限轮询 hook]
    end

    L_BRIDGE <-->|权限请求/响应| W_POLLER
    L_PERM --> L_BRIDGE
    W_POLLER --> W_PERM
```

每个 Teammate 有**独立的权限模式**，可通过 Shift+Tab 切换。Leader 可以审批 Teammate 的计划，审批后 Teammate 继承 Leader 的权限级别。

## Coordinator System Prompt

当 Coordinator Mode 启用时，Leader 获得一个 **368 行**的专用 system prompt，核心原则：

### 工作流阶段

```mermaid
graph LR
    R[Research<br/>并行探索] --> S[Synthesis<br/>综合理解]
    S --> I[Implementation<br/>串行修改]
    I --> V[Verification<br/>验证结果]
    V -->|需要迭代| R
```

### 核心原则

1. **先综合再委派** — Leader 必须自己阅读和理解 Worker 的结果，然后才能指导下一步。**禁止**说 "based on your findings, fix the bug"
2. **并行即超能力** — 独立的研究任务必须并行启动
3. **写入必须串行** — 文件修改不能并行（会冲突）
4. **有效的 Worker prompt** — 必须包含具体的文件路径和行号，不能笼统地说"研究一下"

## 完整工作流示例

```mermaid
sequenceDiagram
    participant USER as 用户
    participant COORD as Coordinator
    participant W1 as auth-researcher
    participant W2 as api-researcher
    participant W3 as test-writer

    USER->>COORD: "重构 auth 模块"

    Note over COORD: Phase 1: 创建团队 + 并行研究
    COORD->>COORD: TeamCreate("auth-refactor")

    par 并行启动
        COORD->>W1: Agent("研究 src/auth/ 的实现细节")
        COORD->>W2: Agent("研究 src/api/ 中 auth 的使用方式")
    end

    W1-->>COORD: 结果: JWT + session token, 3 个中间件...
    W2-->>COORD: 结果: 15 个端点使用 auth...

    Note over COORD: Phase 2: 综合理解（Leader 自己读结果）
    COORD->>COORD: 阅读两个 Worker 的结果<br/>综合理解后制定修改计划

    Note over COORD: Phase 3: 串行实现
    COORD->>W1: SendMessage("修改 src/auth/middleware.ts:42, 具体改动：...")
    W1-->>COORD: 修改完成

    COORD->>W2: SendMessage("更新 src/api/routes.ts 适配新 auth...")
    W2-->>COORD: 更新完成

    Note over COORD: Phase 4: 验证
    COORD->>W3: Agent("运行 npm test -- --grep auth")
    W3-->>COORD: 所有测试通过

    Note over COORD: Phase 5: 清理
    COORD->>W1: SendMessage({ type: "shutdown_request" })
    COORD->>W2: SendMessage({ type: "shutdown_request" })
    W1-->>COORD: { type: "shutdown_response", approve: true }
    W2-->>COORD: { type: "shutdown_response", approve: true }
    COORD->>COORD: TeamDelete("auth-refactor")
    COORD->>USER: "auth 模块重构完成，所有测试通过"
```

## 关键文件一览

### Swarm 基础设施 (`src/utils/swarm/`)

| 文件 | 大小 | 功能 |
|------|------|------|
| `inProcessRunner.ts` | 53KB | 核心进程内 Agent 执行循环 |
| `permissionSync.ts` | 26.5KB | Leader-Worker 权限同步 |
| `teamHelpers.ts` | 21.4KB | 团队文件 I/O、名称清理、发现 |
| `spawnInProcess.ts` | ~5KB | 进程内 spawn 逻辑 |
| `spawnUtils.ts` | ~5KB | 通用 spawn 工具 |
| `constants.ts` | ~1KB | Swarm 常量 |
| `leaderPermissionBridge.ts` | ~5KB | Leader 端权限桥接 |
| `reconnection.ts` | ~3KB | 断线恢复 |
| `teammateLayoutManager.ts` | ~3KB | 颜色分配 |

### 执行后端 (`src/utils/swarm/backends/`)

| 文件 | 大小 | 功能 |
|------|------|------|
| `TmuxBackend.ts` | 21.5KB | Tmux 分屏管理 |
| `ITermBackend.ts` | 12.9KB | iTerm2 原生分栏 |
| `InProcessBackend.ts` | 10.5KB | 进程内执行 |
| `PaneBackendExecutor.ts` | ~5KB | Pane 后端封装 |
| `registry.ts` | 14.8KB | 后端检测和注册 |
| `detection.ts` | ~3KB | 系统能力检测 |
| `types.ts` | ~2KB | 后端接口定义 |

### 工具

| 文件 | 功能 |
|------|------|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 创建团队 |
| `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | 删除团队 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | 消息传递（650 行） |
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式 + system prompt |

### 状态

| 文件 | 功能 |
|------|------|
| `src/tasks/InProcessTeammateTask/types.ts` | Teammate 状态定义 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | 任务接口 |
| `src/state/AppStateStore.ts` | teamContext + inbox |
| `src/utils/agentSwarmsEnabled.ts` | Feature gating |

## 设计洞察

1. **三种后端，一种通信** — 不管用哪种后端（InProcess/Tmux/iTerm2），消息都通过文件邮箱。这简化了通信协议。

2. **InProcess 是默认最优** — 无外部依赖，共享资源，最低开销。Tmux/iTerm2 只在需要独立进程时使用。

3. **优雅关闭不可跳过** — Leader 不能直接 kill Worker，必须走 shutdown 协议。这防止 Worker 在中间状态被终止。

4. **AsyncLocalStorage 隔离** — InProcess Worker 共享进程但有独立的身份上下文。这是 Node.js 的"虚拟线程"模式。

5. **磁盘持久化** — TeamFile 持久化到磁盘（`~/.claude/teams/`），支持断线恢复和跨 session 恢复。

6. **颜色系统** — 8 种颜色（red/blue/green/yellow/purple/orange/pink/cyan），用于 tmux 边框和 UI 区分。

7. **内存管理** — 50 条消息上限 + 磁盘持久化，防止 292-agent 级别的内存爆炸。

8. **两层 Feature Gate** — `isAgentSwarmsEnabled()`（工具可用性）+ `isCoordinatorMode()`（专用 prompt），灵活控制发布。
