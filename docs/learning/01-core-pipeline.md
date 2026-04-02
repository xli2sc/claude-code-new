# 核心管道 — QueryEngine & Query Loop

> Claude Code 的心脏：从用户输入到 API 响应到工具执行的完整流程。

## 概览

核心管道由三层组成：

```mermaid
graph TB
    subgraph "QueryEngine (会话层)"
        QE_INIT[初始化：组装 system prompt]
        QE_SUBMIT[submitMessage：处理用户输入]
        QE_RECORD[记录 transcript]
    end

    subgraph "Query Loop (执行层)"
        QL_CONTEXT[上下文准备：compact/snip/collapse]
        QL_API[API Streaming 调用]
        QL_TOOL[工具执行]
        QL_RETRY[错误恢复 & 重试]
    end

    subgraph "API Client (传输层)"
        AC_STREAM[Streaming 调用 Anthropic SDK]
        AC_RETRY[withRetry 指数退避]
        AC_TOKEN[Token 计数 & 费用追踪]
    end

    QE_SUBMIT --> QL_CONTEXT
    QL_CONTEXT --> QL_API
    QL_API --> AC_STREAM
    AC_STREAM --> AC_RETRY
    QL_API --> QL_TOOL
    QL_TOOL --> QL_API
    QL_API --> QL_RETRY
    QL_RETRY --> QL_API
    QL_API --> QE_RECORD
    AC_STREAM --> AC_TOKEN
```

## QueryEngine (`src/QueryEngine.ts`)

**职责**：管理一个完整的对话 session。一个 QueryEngine 对应一个对话，多次 `submitMessage()` 共享同一个 engine。

### 核心数据结构

```typescript
class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]          // 对话历史
  private abortController: AbortController    // 取消控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage        // 累计 token 使用
  private discoveredSkillNames: Set<string>   // 已发现的 skills
  private loadedNestedMemoryPaths: Set<string> // 已加载的记忆文件
}
```

### QueryEngineConfig — 配置一切

```typescript
type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具
  commands: Command[]            // 可用命令
  mcpClients: MCPServerConnection[]  // MCP 连接
  agents: AgentDefinition[]      // Agent 定义
  canUseTool: CanUseToolFn       // 权限检查函数
  getAppState: () => AppState    // 读取全局状态
  setAppState: (f) => void       // 修改全局状态
  initialMessages?: Message[]    // 初始消息（恢复 session）
  readFileCache: FileStateCache  // 文件读取缓存
  customSystemPrompt?: string    // 自定义 system prompt
  maxTurns?: number              // 最大轮次
  maxBudgetUsd?: number          // 预算上限（USD）
  thinkingConfig?: ThinkingConfig // thinking 模式配置
  // ... 更多配置
}
```

### submitMessage 流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant QE as QueryEngine
    participant QL as Query Loop
    participant API as Anthropic API

    User->>QE: submitMessage(prompt)
    QE->>QE: 1. 组装 system/user context
    QE->>QE: 2. 处理 slash 命令
    QE->>QE: 3. 加载 skills & plugins
    QE->>QE: 4. yield SystemInitMessage
    
    alt 本地命令 (/cost, /version)
        QE->>User: yield 命令结果，直接返回
    else 需要 LLM
        QE->>QL: 进入 query() 循环
        loop 直到 end_turn
            QL->>API: streaming 请求
            API-->>QL: streaming 响应
            alt 包含 tool_use
                QL->>QL: 执行工具
                QL->>API: 工具结果作为 user message
            else end_turn
                QL->>QE: 返回最终响应
            end
        end
        QE->>QE: 5. 记录 transcript
        QE->>User: yield ResultMessage(cost, usage, denials)
    end
```

详细步骤：
1. **Wrap canUseTool** — 包装权限函数以追踪拒绝记录
2. **确定模型和 thinking 配置** — 根据用户设置和 feature flags
3. **获取 system/user context** — git status、claude.md 内容、当前日期
4. **处理用户输入** — 解析 slash 命令、附件、工具限制
5. **Push messages** — 添加到 mutableMessages，持久化到存储
6. **加载 skills 和 plugins** — 缓存模式（headless）或完整加载
7. **yield SystemInitMessage** — 告诉 SDK 调用者有哪些 tools/commands
8. **进入 query() 循环** — 核心 streaming + tool execution
9. **yield ResultMessage** — 最终结果，包含费用、使用量、权限拒绝

## Query Loop (`src/query.ts`)

**职责**：核心 while(true) 循环，负责 API 调用、工具执行、上下文管理、错误恢复。

### 循环状态

```typescript
type State = {
  messages: Message[]                // 当前消息列表
  toolUseContext: ToolUseContext      // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState  // 自动压缩追踪
  maxOutputTokensRecoveryCount: number    // max_tokens 恢复次数
  hasAttemptedReactiveCompact: boolean    // 是否已尝试响应式压缩
  maxOutputTokensOverride: number         // token 上限覆盖
  turnCount: number                       // 当前轮次
  transition: Continue | undefined         // 上一轮为什么继续
}
```

### 主循环流程

```mermaid
flowchart TB
    START[初始化 State] --> CONTEXT_PREP

    subgraph CONTEXT_PREP["上下文准备"]
        A1[应用 tool result budget]
        A2[应用 snip<br/>HISTORY_SNIP]
        A3[应用 microcompact<br/>缓存编辑]
        A4[应用 context collapse]
        A5[检查 autocompact 阈值]
        A1 --> A2 --> A3 --> A4 --> A5
    end

    CONTEXT_PREP --> API_CALL

    subgraph API_CALL["API Streaming"]
        B1[callModel 发起请求]
        B2[处理 stream events:<br/>message_start<br/>content_block_delta<br/>tool_use]
        B1 --> B2
    end

    API_CALL --> CHECK{stop_reason?}

    CHECK -->|tool_use| TOOL_EXEC
    CHECK -->|end_turn| TERMINAL
    CHECK -->|max_output_tokens| RECOVERY

    subgraph TOOL_EXEC["工具执行"]
        C1[StreamingToolExecutor]
        C2[checkPermissions]
        C3[执行工具]
        C4[yield 工具结果]
        C1 --> C2 --> C3 --> C4
    end

    TOOL_EXEC --> CONTEXT_PREP

    subgraph RECOVERY["错误恢复"]
        D1[增大 max_tokens 重试]
        D2[响应式 compact<br/>413 too long]
        D3[Model fallback]
        D1 --> CONTEXT_PREP
        D2 --> CONTEXT_PREP
        D3 --> CONTEXT_PREP
    end

    TERMINAL[返回 Terminal Result]
```

### 循环继续的原因（Continue Sites）

| 原因 | 触发条件 | 处理方式 |
|------|---------|---------|
| `tool_use` | API 返回 tool_use block | 执行工具后继续 |
| `max_output_tokens` | 输出超长 | 增大 limit 重试 |
| 响应式 compact | 收到 413 错误 | compact 后重试 |
| Model fallback | 特定错误 | 换模型重试 |
| Stop hook | hook 返回 Continue | 重新进入循环 |

### 循环终止条件

| 条件 | 说明 |
|------|------|
| `end_turn` | API 正常结束 |
| `tool_limit` | 工具调用次数超限 |
| `max_turns` | 达到最大轮次 |
| `blocking_limit` | 上下文窗口达到阻塞限制 |
| `budget_exceeded` | 超出 token/USD 预算 |
| `abort` | 用户取消 |

## API Client (`src/services/api/claude.ts`)

### 核心 Streaming 调用

```typescript
const stream = await anthropic.beta.messages.stream({
  model: normalizeModelStringForAPI(model),
  max_tokens: Math.min(capped, modelMaxTokens),
  messages: normalizeMessagesForAPI(messages),
  system: renderSystemPrompt(systemPrompt),
  temperature: 1,
  tools: toolToAPISchema(tools),
  thinking: { type: thinkingType, budget_tokens: maxThinkingTokens },
  betas: getMergedBetas(model),
  metadata: getAPIMetadata(),
})

for await (const event of stream) {
  // stream events: message_start, content_block_start,
  // content_block_delta, message_delta, message_stop
}
```

### Prompt Caching

Claude Code 使用 prompt caching 来减少重复 token 的计算成本：
- **System prompt** — 静态部分被缓存（大量 token）
- **Scope 控制** — global vs ephemeral cache
- **1h TTL** — 部分查询源可享受 1 小时缓存

### Token & 费用追踪

```typescript
type NonNullableUsage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number  // 缓存创建
  cache_read_input_tokens: number       // 缓存命中
}
```

每次 API 调用后累加 usage，最终通过 `/cost` 命令展示。

## 重试逻辑 (`src/services/api/withRetry.ts`)

```mermaid
flowchart TB
    CALL[发起 API 调用] --> CHECK{错误类型?}
    
    CHECK -->|401/403 认证| AUTH[清除凭据缓存<br/>刷新 OAuth token<br/>重建客户端]
    CHECK -->|529/429 容量| CAPACITY{retry-after?}
    CHECK -->|ECONNRESET| CONN[禁用 keep-alive<br/>重连]
    CHECK -->|其他| BACKOFF[指数退避重试]
    
    CAPACITY -->|短延迟| RETRY_SHORT[等待后重试]
    CAPACITY -->|长延迟 + fast mode| FALLBACK_SPEED[降速重试]
    CAPACITY -->|无人值守| PERSIST[无限重试<br/>心跳保活]
    
    AUTH --> RETRY[重试]
    RETRY_SHORT --> RETRY
    FALLBACK_SPEED --> RETRY
    PERSIST --> RETRY
    CONN --> RETRY
    BACKOFF --> RETRY
    
    RETRY --> CALL
```

关键策略：
- **认证错误** (401/403)：清除缓存 → 刷新 token → 重建客户端
- **容量限制** (529/429)：检查 retry-after header，fast mode 下短延迟，无人值守模式无限重试
- **连接错误** (ECONNRESET)：禁用连接池 → 重连
- **最大重试次数**：529 错误最多 3 次，之后 fallback 到非 streaming
- **Model fallback**：特定错误可以切换到备用模型

## 上下文窗口管理

```mermaid
graph TB
    subgraph "上下文窗口"
        SP[System Prompt<br/>已缓存，静态]
        UC[User Context<br/>git status, claude.md]
        SC[System Context<br/>日期, cache breaker]
        MSG[Messages<br/>经过 compact/snip/collapse]
        TRAIL[Trailing Messages<br/>受保护的最新消息]
    end

    SP --> UC --> SC --> MSG --> TRAIL

    subgraph "管理策略"
        AC[Auto Compact<br/>token 阈值触发]
        MC[Micro Compact<br/>缓存编辑优化]
        SN[Snip<br/>消息历史裁剪]
        CC[Context Collapse<br/>高级折叠]
        RC[Reactive Compact<br/>413 错误触发]
    end

    MSG -.-> AC
    MSG -.-> MC
    MSG -.-> SN
    MSG -.-> CC
    MSG -.-> RC
```

**Auto Compact 常量**：
- `AUTOCOMPACT_BUFFER_TOKENS = 13,000`
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20,000`
- `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`（熔断器）

## SDK 消息类型

QueryEngine 通过 AsyncGenerator 向调用者 yield 消息：

| 类型 | 说明 |
|------|------|
| `SDKSystemInitMessage` | Tool/command/model 概要 |
| `SDKUserMessageReplay` | 用户消息回显 |
| `SDKAssistantMessage` | 模型响应 |
| `SDKCompactBoundaryMessage` | 上下文压缩标记 |
| `SDKLocalCommandOutputMessage` | Slash 命令输出 |
| `SDKProgressMessage` | 工具执行进度 |
| `SDKErrorMessage` | API/执行错误 |
| `SDKResultMessage` | 最终结果（费用、使用量、权限拒绝） |

最终的 `SDKResultMessage` 包含：
```typescript
{
  type: 'result',
  duration_ms: number,         // 总耗时
  num_turns: number,           // 循环轮次
  total_cost_usd: number,      // 总费用
  usage: NonNullableUsage,     // Token 使用
  permission_denials: [...],   // 被拒绝的权限
  stop_reason: string,         // 终止原因
}
```

## 关键洞察

1. **Generator 模式** — 整个 pipeline 是 `AsyncGenerator`，允许调用者逐条接收消息，实现 streaming UI
2. **不变性 + 可变状态** — State 对象在循环内可变，但 messages 数组通过引用传递，避免大量拷贝
3. **多层错误恢复** — 认证刷新 → 退避重试 → 模型降级，确保尽可能完成任务
4. **渐进式上下文管理** — 从 micro compact 到 auto compact 到 reactive compact，根据紧急程度选择策略
5. **Prompt caching** — 静态 system prompt 被缓存，大幅减少重复 token 的成本
