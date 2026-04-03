# 远程 Agent 执行 — CCR、Triggers、Ultraplan

> Claude Code 的分布式 Agent 系统：云端执行、定时调度、远程规划与代码审查。

## 概览

Claude Code 可以把 Agent 的执行从本地"传送"到云端（CCR），实现本地笔记本指挥、云端容器执行的模式。

```mermaid
graph TB
    subgraph "本地"
        LOCAL[Claude Code CLI<br/>Leader / 用户界面]
        POLL[Polling Loop<br/>每 1s 拉取事件]
        PERM_UI[权限 UI<br/>本地审批工具调用]
    end

    subgraph "Anthropic 云"
        CCR[CCR Container<br/>远程 Agent 执行]
        API[Sessions API<br/>创建/管理 session]
        WS[WebSocket<br/>事件推送]
        TRIGGERS[Triggers API<br/>定时调度]
    end

    subgraph "用户基础设施（BYOC）"
        SELF[Self-Hosted Runner<br/>用户自己的机器]
        ENV[Environment Runner<br/>用户容器]
    end

    LOCAL -->|创建 session| API
    API -->|启动| CCR
    CCR -->|事件流| WS
    WS -->|推送| LOCAL
    LOCAL -->|HTTP POST| CCR
    POLL -->|拉取事件| API
    CCR -->|权限请求| PERM_UI
    PERM_UI -->|批准/拒绝| CCR

    TRIGGERS -->|定时触发| CCR
    SELF -->|连接| API
    ENV -->|连接| API
```

## 五种远程任务类型

| 类型 | 触发方式 | 用途 |
|------|---------|------|
| `remote-agent` | 通用 | 泛用远程 Agent |
| `ultraplan` | `/ultraplan` 命令 | 远程规划，浏览器审批 |
| `ultrareview` | `/ultrareview` 命令 | 远程代码审查（bughunter） |
| `autofix-pr` | 自动触发 | PR 自动修复 |
| `background-pr` | 长运行 | PR 持续监控 |

## 远程 Session 生命周期

```mermaid
sequenceDiagram
    participant USER as 用户
    participant CLI as 本地 CLI
    participant API as Sessions API
    participant CCR as CCR Container

    USER->>CLI: /ultraplan "重构 auth"

    Note over CLI: 前置检查
    CLI->>CLI: 1. OAuth 登录？
    CLI->>CLI: 2. 远程环境可用？
    CLI->>CLI: 3. Git 仓库？GitHub App？

    CLI->>API: POST /v1/sessions<br/>{model, sources, tools, prompt}
    API->>CCR: 启动容器，clone 代码
    API-->>CLI: session_id

    CLI->>CLI: registerRemoteAgentTask()
    CLI->>CLI: 开始 polling loop

    loop 每 1 秒
        CLI->>API: GET /events?after={lastEventId}
        API-->>CLI: 新事件 (SDK messages)
        CLI->>CLI: 处理事件，更新 UI
    end

    CCR->>CCR: Agent 执行工具...
    
    alt 需要本地权限
        CCR-->>CLI: SDKControlPermissionRequest<br/>(via WebSocket)
        CLI->>USER: 显示权限对话框
        USER->>CLI: 批准
        CLI->>CCR: SDKControlResponse(allow)
    end

    CCR->>API: result(success)
    API-->>CLI: session archived
    CLI->>USER: 通知完成
```

## RemoteSessionManager — 远程连接管理

### 双通道通信

```mermaid
graph LR
    CCR_OUT[CCR 容器] -->|推送事件| WS[WebSocket<br/>wss://.../v1/sessions/ws/id/subscribe]
    WS --> LOCAL_IN[本地 CLI<br/>接收 assistant/user/result/control]

    LOCAL_OUT[本地 CLI] -->|用户输入| POST[POST /v1/sessions/id/events]
    LOCAL_OUT -->|取消| INTERRUPT[POST /v1/sessions/id/interrupt]
    LOCAL_OUT -->|权限回复| PERM[POST /v1/sessions/id/control]
    POST --> CCR_IN[CCR 容器]
    INTERRUPT --> CCR_IN
    PERM --> CCR_IN
```

**为什么用两个通道？** WebSocket 是服务端 → 客户端的推送（事件流），HTTP 是客户端 → 服务端的请求（用户输入、权限回复）。

### WebSocket 重连策略

| 关闭码 | 行为 | 原因 |
|--------|------|------|
| 4001 | 重试 3 次 | Session not found（compaction 期间瞬时错误） |
| 4003 | 立即停止 | Unauthorized（永久） |
| 其他 | 重试 5 次，2s 间隔 | 瞬时网络问题 |

Ping/Pong 每 30 秒保活。

## Polling 与完成检测

### Polling Loop

```mermaid
flowchart TB
    POLL[每 1s 轮询] --> FETCH["GET /events?after=lastEventId\n最多 50 页"]
    FETCH --> PROCESS[处理新事件]
    PROCESS --> UPDATE["更新 task.log + 磁盘输出"]
    UPDATE --> CHECK{完成检测}

    CHECK -->|以上都不是| POLL
    CHECK -->|完成条件满足| DONE[任务完成]

    DONE --- NOTE["完成条件：\n1. session archived\n2. result message\n3. 自定义 checker 返回非 null\n4. 稳定 idle 连续 5 次\n5. 超时 30min for review"]

    style NOTE fill:#f0f0f0,stroke:#999,stroke-dasharray: 5 5
```

### 稳定 Idle 防误判

远程 session 在 tool turns 之间会短暂变为 idle。需要 **连续 5 次** polling 都是 idle 且 log 没增长，才认为真的完成。

### 自定义完成检查器

每种任务类型可以注册自己的完成检查器：

```typescript
type RemoteTaskCompletionChecker = 
  (metadata?: RemoteTaskMetadata) => Promise<string | null>
  // 返回 string → 完成（string 成为通知文本）
  // 返回 null → 继续 polling
```

例如 `autofix-pr` 可以查询 GitHub API 检查 PR 状态。

## Ultraplan — 远程规划

### 独特之处：浏览器审批

Ultraplan 不是简单的"远程执行然后返回结果"。它有一个**浏览器审批循环**：

```mermaid
sequenceDiagram
    participant CLI as 本地 CLI
    participant CCR as 远程 Agent
    participant BROWSER as 浏览器 Modal

    CLI->>CCR: "重构 auth 模块"
    CCR->>CCR: 分析代码，制定计划
    CCR->>CCR: ExitPlanModeV2Tool(plan)
    CCR-->>CLI: plan_ready 状态

    CLI->>BROWSER: 显示计划审批 Modal
    
    alt 用户批准
        BROWSER->>CLI: approve
        CLI->>CCR: tool_result(success)
        CCR->>CCR: 开始执行计划
    else 用户拒绝 + 反馈
        BROWSER->>CLI: reject + "也考虑性能"
        CLI->>CCR: tool_result(error, feedback)
        CCR->>CCR: 修改计划...
        CCR->>CCR: ExitPlanModeV2Tool(revised_plan)
        CCR-->>CLI: plan_ready（新计划）
        CLI->>BROWSER: 显示修改后的计划
    end
```

### Ultraplan 阶段

| 阶段 | 含义 |
|------|------|
| `running` | 远程 Agent 在思考/执行 |
| `needs_input` | Agent 问了一个问题，等待用户回复 |
| `plan_ready` | ExitPlanMode 被调用，计划显示在浏览器 |

### ExitPlanModeScanner

一个状态机，从 polling 事件流中检测计划提交：

```typescript
class ExitPlanModeScanner {
  ingest(newEvents: SDKMessage[]): ScanResult
  // 返回:
  // { kind: 'approved', plan: string }     — 用户批准
  // { kind: 'rejected', id: string }       — 用户拒绝（可迭代）
  // { kind: 'teleport', plan: string }     — 拒绝 + 转移到本地执行
  // { kind: 'pending' }                    — 等待中
  // { kind: 'terminated', subtype: string } — session 终止
}
```

## Ultrareview — 远程代码审查

### 两种模式

**Bughunter 模式**（生产路径）：
```
远程 session 启动 → SessionStart hook 运行 run_hunt.sh → 
hook 在后台执行（session 保持 idle）→ 
hook 输出 <remote-review-progress> 标签 → 
hook 输出 <remote-review> 最终结果
```

**Prompt 模式**：
```
远程 session 启动 → 正常 assistant turns → 
assistant 输出 <remote-review> 标签
```

### 审查进度追踪

Hook 输出 JSON 格式的进度：
```json
{
  "stage": "finding",        // finding → verifying → synthesizing
  "bugs_found": 5,
  "bugs_verified": 3,
  "bugs_refuted": 1
}
```

本地 UI 实时显示进度。

## 定时触发 — Scheduled Triggers

### 架构

```mermaid
graph TB
    subgraph "配置时（本地）"
        SKILL[scheduleRemoteAgents skill]
        TOOL[RemoteTriggerTool]
        SKILL --> TOOL
        TOOL -->|POST /v1/code/triggers| API[Triggers API]
    end

    subgraph "执行时（云端）"
        CRON[Cron 定时器<br/>最小间隔 1 小时]
        CRON --> CREATE[创建 CCR Session]
        CREATE --> EXEC[执行 Agent]
        EXEC --> ARCHIVE[归档 Session]
    end

    API --> CRON
```

### Trigger 配置

```json
{
  "name": "daily-test-runner",
  "cron_expression": "0 9 * * 1-5",
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "env_xxx",
      "session_context": {
        "model": "claude-sonnet-4-6",
        "sources": [
          {"git_repository": {"url": "https://github.com/org/repo"}}
        ],
        "allowed_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep"]
      },
      "events": [
        {
          "data": {
            "type": "user",
            "message": {"content": "运行所有测试，如果有失败自动修复并提 PR", "role": "user"}
          }
        }
      ]
    }
  },
  "mcp_connections": [
    {"connector_uuid": "uuid", "name": "github", "url": "https://..."}
  ]
}
```

### 管理接口

| 操作 | API | 说明 |
|------|-----|------|
| 创建 | `POST /v1/code/triggers` | 设置定时 Agent |
| 列表 | `GET /v1/code/triggers` | 查看所有 triggers |
| 查看 | `GET /v1/code/triggers/{id}` | 查看详情 |
| 更新 | `POST /v1/code/triggers/{id}` | 修改配置 |
| 手动运行 | `POST /v1/code/triggers/{id}/run` | 立即触发一次 |

## 三种执行环境

```mermaid
graph TB
    subgraph "CCR — Anthropic 云（默认）"
        CCR_C[容器化环境<br/>自动 clone 代码<br/>自动安装依赖]
    end

    subgraph "BYOC — 用户自己的机器"
        ENV_R[Environment Runner<br/>用户容器中运行]
        SELF_R[Self-Hosted Runner<br/>用户服务器上运行]
    end

    subgraph "连接方式"
        PROXY[Upstream Proxy<br/>容器内代理<br/>token 注入]
    end

    CCR_C --> PROXY
    ENV_R --> PROXY
    SELF_R --> PROXY
```

### CCR（默认）

- Anthropic 托管的容器环境
- 自动从 GitHub clone 代码（需要安装 Claude GitHub App）
- 如果没有 GitHub App → fallback 到 git bundle 上传
- 支持 outcome branch（PR 场景）
- Session TTL 自动清理

### BYOC Environment Runner

- **Feature flag**: `BYOC_ENVIRONMENT_RUNNER`
- 在用户自己的基础设施上运行
- 通过 `upstreamproxy` 连接回 CCR API
- 入口：`claude environment-runner`

### Self-Hosted Runner

- **Feature flag**: `SELF_HOSTED_RUNNER`
- 另一种用户自托管模式
- 入口：`claude self-hosted-runner`

### Upstream Proxy（容器内安全）

容器内的代理层，处理认证和网络：

```mermaid
flowchart TB
    TOKEN[/run/ccr/session_token] --> READ[读取 token]
    READ --> PRCTL["prctl(PR_SET_DUMPABLE, 0)<br/>阻止 ptrace"]
    PRCTL --> UNLINK[删除 token 文件<br/>只存在于进程堆中]
    UNLINK --> CA[下载 CA 证书<br/>拼接系统 CA bundle]
    CA --> RELAY[启动 CONNECT→WebSocket relay<br/>127.0.0.1:port]
    RELAY --> ENV["设置环境变量<br/>HTTPS_PROXY=http://127.0.0.1:port<br/>SSL_CERT_FILE=ca-bundle.crt"]
```

**NO_PROXY 排除**（不走代理的域名）：
- `anthropic.com`（不 MITM API 调用）
- `github.com`、`*.githubusercontent.com`
- npm、PyPI、crates.io、Go proxy（包管理器）
- localhost、RFC1918、IMDS

**Fail-Open**：任何代理错误都会禁用代理（而不是阻塞 session）。

## 远程权限处理

```mermaid
sequenceDiagram
    participant CCR as 远程 Agent
    participant WS as WebSocket
    participant CLI as 本地 CLI
    participant USER as 用户

    CCR->>CCR: 需要调用 BashTool("rm -rf old/")
    CCR->>WS: SDKControlPermissionRequest<br/>{tool: "Bash", input: "rm -rf old/"}
    WS->>CLI: 推送权限请求
    CLI->>CLI: 创建 synthetic AssistantMessage<br/>（让本地 UI 显示工具调用）
    CLI->>USER: 显示权限对话框
    USER->>CLI: 批准
    CLI->>CCR: HTTP POST SDKControlResponse(allow)
    CCR->>CCR: 执行 BashTool
```

对于本地没有的工具（如远程 MCP 工具），`remotePermissionBridge.ts` 会创建 **stub Tool 对象** 供 UI 渲染。

## Session 恢复（`--resume`）

```mermaid
flowchart TB
    RESUME[claude --resume] --> SCAN[扫描 sidecar 元数据]
    SCAN --> FETCH[对每个 session 调用 fetchSession()]
    FETCH --> CHECK{session 还存在？}
    CHECK -->|404| DROP[丢弃元数据]
    CHECK -->|200| RESTORE[重建 RemoteAgentTaskState]
    RESTORE --> POLL[重启 polling loop]
```

## 关键文件

| 文件 | 大小 | 功能 |
|------|------|------|
| `src/remote/RemoteSessionManager.ts` | 9.3KB | WebSocket + HTTP 会话管理 |
| `src/remote/SessionsWebSocket.ts` | 12.5KB | WebSocket 重连逻辑 |
| `src/remote/sdkMessageAdapter.ts` | 9KB | SDK 消息格式转换 |
| `src/remote/remotePermissionBridge.ts` | 2.4KB | 远程权限桥接 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | ~800 行 | 远程任务生命周期 |
| `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts` | ~200 行 | Triggers CRUD |
| `src/skills/bundled/scheduleRemoteAgents.ts` | 19KB | 调度 Skill |
| `src/utils/ultraplan/ccrSession.ts` | ~500 行 | Ultraplan 浏览器审批 |
| `src/upstreamproxy/upstreamproxy.ts` | 15KB | 容器内代理 |
| `src/upstreamproxy/relay.ts` | 10KB | CONNECT→WebSocket relay |

## 设计洞察

1. **本地是 UI，云端是算力** — 本地 CLI 只负责展示和权限审批，实际计算在 CCR 容器中。这让笔记本可以指挥长时间运行的任务。

2. **Polling 而非 Push** — 虽然有 WebSocket，主要的事件获取仍然用 polling（1s 间隔）。WebSocket 用于低延迟的权限请求。这保证了即使 WebSocket 断开，polling 仍能工作。

3. **稳定 Idle 防误判** — 5 次连续 idle 才认为完成。这是从实践中学到的：远程 Agent 在 tool turns 之间会短暂 idle。

4. **Fail-Open 设计** — 代理错误 → 禁用代理；CA 证书下载失败 → 跳过；token 刷新失败 → 用旧 token 重试。永远不阻塞用户。

5. **三种部署形态** — CCR（最简单，Anthropic 管理）、BYOC（用户机器，需要自己维护）、Self-Hosted（独立进程）。同一套 API，不同的执行环境。

6. **Ultraplan 的迭代循环** — 用户可以拒绝计划并给反馈，远程 Agent 修改后重新提交。这比"一次性规划"灵活得多。

7. **Session 持久化** — Metadata 写入 sidecar 文件，`--resume` 可以恢复正在运行的远程任务。不会因为关闭终端就丢失远程 session。
