# 技术架构文档：Terminal LLM Agent（学习复现版）

> 本文档基于《需求文档》（`docs/derived-requirements.md`），分析目标系统的功能模块划分与具体实现思路，供从零编码参考。

## 1. 引言

目标：从零实现需求文档定义的Terminal LLM Agent，重点理解「模型、Agent、工具、宿主」四者的协作。架构遵循三条原则：

1. **少框架**：优先 Go 标准库，仅在渲染/终端控制/序列化等点上引入成熟库；
2. **清晰分层**：宿主 → 交互 → 智能 → 模型 → 工具，各层单向依赖；
3. **核心能力完整**：模型调用、工具调用、循环、压缩、记忆、持久化、流式、控制全部自实现。

## 2. 总体架构

> 本框架通用于本地工作区操作；内置工具使其具备完整的 **coding 能力**（读改源码、跑测试、按报错修复），coding 是首要应用场景，但也可用于运维、数据分析等其他领域。

### 2.1 分层

| 层 | 包/目录 | 职责 |
|----|---------|------|
| 宿主层 | `cmd/minicode` | 进程入口，最小参数解析，组装依赖 |
| 交互层 | `internal/cli` | 全屏 UI、输入编辑、流式渲染、审批交互、统计展示 |
| 智能层 | `internal/agent` | Agent Loop、上下文压缩、项目记忆、流式聚合、输出接口 |
| 模型层 | `internal/provider` | Provider 抽象 + OpenAI 兼容客户端（非流式 / SSE） |
| 工具层 | `internal/tools` | 工具注册表 + `read`/`write`/`edit`/`bash` + 文件系统句柄 |
| 支撑层 | `session`/`compaction`/`memory`/`config`/`telemetry`/`token` | 持久化、压缩、记忆、配置、追踪、token 估算 |

### 2.2 单轮数据流

```
用户输入
  → cli 收集
  → agent 组装消息（system + 记忆 + 摘要 + 历史）
  → provider.Chat(ctx, messages, tools)
  → 响应（content / tool_calls）
  → 若有 tool_calls：
       逐条 → 审批(如需) → registry.Execute → 结果追加历史 → 再次请求
  → 若无 tool_calls：输出最终回复
  → cli 渲染
  → 循环直到无 tool_calls 或达轮次上限
```

## 3. 模块划分与职责

- **`cmd/minicode`**：`main.go` 仅做启动入口，调用 `cli.Run`，不承载逻辑。
- **`internal/cli`**：
  - `run.go`：配置加载、信号注册、会话组装；
  - `input.go`：多行编辑、按键、鼠标事件；
  - `screen.go`：全屏会话、固定底部输入区、消息滚动；
  - `output.go` / `stream_preview.go`：流式预览、统计、最终回答；
  - `markdown.go`：终端检测与渲染；
  - `approval.go`：副作用工具审批与自动批准；
  - `terminal_mode_unix.go` / `terminal_mode_other.go`：终端模式接入。
- **`internal/agent`**：
  - `run.go`：Session、交互循环、持久化接入；
  - `prompt.go`：系统 Prompt 构造；
  - `stream.go`：流式调用与完整响应聚合 + 校验；
  - `compact.go`：压缩入口、预算重置、超限恢复；
  - `memory.go`：项目记忆加载与写后刷新识别；
  - `output.go`：最终输出、统计、流式预览接口；
  - `approval.go`：工具审批接口。
- **`internal/provider`**：
  - `provider.go`：Provider 接口、流式契约、配置工厂；
  - `types.go`：消息/响应/工具调用/错误；
  - `messages.go`：消息序列与工具配对校验；
  - `openai.go`：非流式 HTTP 调用；
  - `openai_stream.go`：SSE 请求、事件解析、取消、关闭。
- **`internal/tools`**：`registry.go`、`bash.go`、`read.go`、`write.go`、`edit.go`、`filesystem.go`。
- **`internal/session`**：`store.go` —— JSON 读取与原子写入。
- **`internal/compaction`**：`trigger.go`（阈值）、`compact.go`（保留/序列化/分块摘要）、`state.go`（检查点）、`prompt.go`（pi 摘要模板）。
- **`internal/memory`**：`load.go`、`files.go`（详情路径识别与候选校验）。
- **`internal/config`**：`config.go` —— TOML 读取与校验。
- **`internal/telemetry`**：`langfuse.go` —— OTLP/HTTP 追踪。
- **`internal/token`**：`estimate.go` —— token 粗估（记忆与上下文共用）。

## 4. 核心数据结构

```go
// 消息与工具调用（provider 层）
type Message struct {
    Role       string
    Content    string
    ToolCalls  []ToolCall
    ToolCallID string // 工具结果归属
}
type ToolCall struct {
    ID       string
    Function struct{ Name, Arguments string }
}

// 工具（tools 层）
type Tool struct {
    Definition provider.Tool       // JSONSchema 声明
    Execute    func(ctx, call, stdout) (string, error)
}
type ToolRegistry struct {
    tools map[string]Tool
    names []string
}

// 会话与用量
type Session struct {
    ID         string
    Messages   []provider.Message
    Workspace  string
    Checkpoint CompactionCheckpoint
}
type Usage struct{ Input, Output, Total int }
```

## 5. 关键流程

### 5.1 启动流程

`cmd` → `cli.Run` → 读 `config/config.toml` → `provider.New`（按配置创建 Provider）→ `agent.NewSession`（开工作区、注册内置工具、加载项目记忆、写系统 Prompt）→ 进入输入循环。

### 5.2 单轮 Agent Loop（伪代码）

```go
for {
    预算检查 → 如需则 compact()
    resp := provider.Chat(ctx, buildMessages(), registry.Definitions())
    messages = append(messages, resp.Message)
    if len(resp.ToolCalls) == 0 { break }   // 输出最终回复
    for call := range resp.ToolCalls {
        stats.ToolCalls++
        if requiresApproval(call.Function.Name) && approver != nil {
            approved, _ := approver.Approve(ctx, content, call)
            if !approved { result = "tool execution denied by user"; goto append }
        }
        result, _ = registry.Execute(ctx, call, io.Discard)
    append:
        messages = append(messages, toolResultMessage(call.ID, result))
    }
    persistSession()
}
```

### 5.3 工具调用与审批

- `requiresApproval(name)` 对 `bash`/`write`/`edit` 返回 true，`read` 返回 false。
- 用户**拒绝不是运行错误**：结果 `"tool execution denied by user"` 回传模型，模型可换方式。

### 5.4 流式累积与校验

- SSE 事件按 `index` 累积 `tool_calls`；检查索引连续性与调用/结果配对；
- 整条响应正常结束且校验通过才执行；**提前断线不执行半截 assistant 消息与工具**。

### 5.5 上下文压缩

- 触发：剩余预算 < 16K（窗口固定 256K）；
- 保留最近约 20K token 原文，更早内容用 LLM 摘要（pi 模板）替代；
- 摘要输入完整发送，超预算分块更新；模型报上下文超限时重试一次主请求；
- 检查点独立保存摘要与近期起点，恢复时重建模型上下文。

### 5.6 项目记忆加载

- 启动与每轮前读取 `MiniCode.md`（≤64KiB，≤4096 token）；
- 按需详情由模型调用 `read` 加载 `.minicode/memory/*.md`；
- 根目录记忆成功写入后，整批工具完成再刷新。

### 5.7 会话恢复

- `--session <id>`：读 JSON → 重建消息/检查点 → 重建系统 Prompt → 继续输入循环；
- 不恢复运行中的进程、审批状态或文件读取缓存。

## 6. 各模块具体实现思路

### 6.1 Agent Loop（核心）
- 状态即 `messages` 切片，循环纯逻辑；每轮结束持久化。
- 关键约束：tool_calls 与结果的配对由消息序列保证；**发送前**集中校验（角色、参数 JSON、ID 唯一、配对）。

### 6.2 Provider 抽象
- 定义 `Provider` 接口：`Chat(ctx, messages, tools) (*Response, error)`；可选 `StreamingProvider`：`Stream(...) 事件流`。
- `OpenAI` 实现：构造请求体、HTTP 调用、解析、`SSE` 解析。
- `FakeProvider`（测试）：返回预设 `tool_calls`/响应，无需真实模型，验证 Loop/历史/校验。

### 6.3 工具注册与执行
- `Tool` 把 `Definition`（JSONSchema）与 `Execute` 绑定；
- `Registry`：`Register`/`Lookup`/`Definitions`/`Execute`；
- 每个工具自校验参数（`decodeToolArguments`：必填 / 未知 / null / JSON 解码）。

### 6.4 终端 UI
- 全屏备用屏幕；输入区固定底部；上方滚动；
- 行编辑（readline/ansi）处理多行、折行、鼠标；
- Markdown 实时渲染（glamour），合并刷新（~50ms），窗口变化重排；
- PTY + 窗口尺寸事件驱动重排。

### 6.5 上下文压缩
- 预算校准：每次请求前估算 token；
- 摘要：完整原文送摘要请求（不截断），超预算分块；
- 检查点：独立保存摘要与近期起点，供恢复重建。

### 6.6 项目记忆
- 常驻加载整个 `MiniCode.md`；按需详情由模型 `read`；
- 目录维护在根文件；写后整批刷新。

### 6.7 会话持久化
- JSON 序列化；临时文件 + 原子重命名；`0600` 权限；
- 拒绝非法 ID / 损坏 / 版本不兼容 / 工作区不匹配 / 不完整工具消息。

### 6.8 执行控制
- context 取消链路：`Ctrl+C` → `SIGINT` → `ctx.Cancel` → 取消模型请求 + `Kill` 进程组（Unix `Setpgid`+负 PID）；
- 区分中断 / 超时 / 失败提示，退出码不同。

### 6.9 可观测
- 每轮累加统计；Langfuse 经 OTLP/HTTP 上报输入、请求、结果、用量、错误。

## 7. 技术选型与依赖

- **语言**：Go 1.26.x
- **Markdown 渲染**：`charmbracelet/glamour`
- **终端控制**：`charmbracelet/x/{ansi,term,termios}`
- **PTY / 虚拟终端**：`creack/pty`、`hinshun/vt10x`
- **行编辑**：`ergochat/readline`
- **UUID**：`google/uuid`
- **TOML**：`pelletier/go-toml/v2`
- **追踪**：`OpenTelemetry` + Langfuse OTLP
- **测试**：标准 `testing` + `-race`

## 8. 开发里程碑（对照 Day 0–15）

| 阶段 | 内容 |
|------|------|
| Day 0 | 项目定位、范围、计划 |
| Day 1 | 协议结构体 + 单次非流式模型调用 |
| Day 2 | 工具 Schema 定义 + 工具调用响应解析 |
| Day 3 | Agent Loop |
| Day 4 | `bash`/`read`/`write`/`edit` 四工具 + 注册表 |
| Day 5 | 系统 Prompt + 默认 CLI 循环 + 会话内跨轮记忆 |
| Day 6 | 工具输出截断、500 轮上限、超时、副作用审批 |
| Day 7 | Ctrl+C/SIGTERM 中断、context 取消、提示区分 |
| Day 8 | token 统计与过程可视化 |
| Day 9 | Session 持久化 + `--session` 恢复 |
| Day 10 | Provider 接口、配置工厂、统一响应/错误、Fake Provider |
| Day 11 | SSE 请求与事件解析、终端增量预览、用量事件 |
| Day 12 | tool_calls 分片累积与校验、流式 Markdown 渲染 |
| Day 13 | 上下文预算、自动/手动压缩、固定保留近期 20K |
| Day 14 | 合法切分、发送前校验、连续压缩与恢复、语义验收 |
| Day 15 | 常驻记忆、按需索引、写后刷新、迁移到工作区 |

## 9. 风险与取舍

- **Unix-only**：`bash` 用进程组/信号，Windows 需 WSL；本期不实现 Windows 原生适配。
- **测试重于实现**：`test/` 行数约为 `internal/` 的 2 倍，需投入足够测试成本。
- **无危险命令拦截**：安全靠审批，不提供黑名单。
- **单一 provider**：扩展多模型需新增适配器。

## 10. 扩展点（为后续能力预留）

- **新工具**：实现 `Tool` 接口并注册即可（如 `kubectl`、`rag_query`、`browser` 工具）。
- **多 Agent（Day 16）**：主 Agent 委派 Subagent，启动独立上下文的 `Session`，结果回传。
- **Skill**：复用按需记忆机制，新增 `use_skill` 工具 + skill 目录（`.md` 指令）。
- **RAG**：工具式（主动检索）或启动前 Prompt 注入（被动检索）。
