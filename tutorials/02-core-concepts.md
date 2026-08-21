# 02 · 六个核心概念：Agent / Model / Tool / Session / Event / Runner

> 阅读目标：把整个框架的六个核心概念一个一个讲透。读完后，你对"这个框架由哪些积木组成"有清晰认识。
> 每个概念我都会给出：大白话解释 + 代码里的真实定义 + 一个生活类比。

---

## 0. 先来一个总类比：一家餐厅

想象你开了一家餐厅，tRPC-Agent-Go 就是一套餐厅管理系统：

| 框架概念 | 餐厅里的东西 | 作用 |
|----------|-------------|------|
| **Model** | 大厨 | 负责"想"和"做菜"（生成回答） |
| **Tool** | 厨房设备/食材供应商 | 大厨不能啥都自己做，要用设备、订食材 |
| **Agent** | 服务员（有主见的服务员） | 接单、安排、用设备、最终上菜 |
| **Session** | 每桌的点单记录本 | 记住这一桌聊到哪了 |
| **Event** | 厨房传菜口递出来的每道菜/每个通知 | 所有进展的对外出口 |
| **Runner** | 店长 | 管理所有桌子、指挥服务员、处理突发 |

带着这个类比读下面的细节，会轻松很多。

---

## 1. Model（模型）——"大脑"

### 大白话

模型就是那个"会说话的 AI 大脑"。它接收你给它的一串消息，然后一个字一个字地生成回答。

### 代码里的定义

框架定义了一个非常简单的接口（`model/model.go`）：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 所有模型厂商都要实现这个"标准接口"（Model interface）
type Model interface {
    // 核心方法：输入请求，返回一个"响应流"（流式结果）
    // 返回的是一个通道 <-chan *Response，可以理解成一条事件流
    GenerateContent(ctx context.Context, req *Request) (<-chan *Response, error)

    // 告诉框架：这个模型叫什么名字等基本信息
    Info() *Info
}
```

**你只需要实现两个方法**：

- `GenerateContent`：核心方法。给它请求，它返回一个通道，里面不断流出响应片段（这就是"流式"）。
- `Info()`：告诉框架这个模型叫什么名字等基本信息。

### 关键点

1. **框架不绑定任何一家模型厂商**。只要实现了这个接口，OpenAI、DeepSeek、通义、Gemini、本地 Ollama……都能用。`model/` 目录下就是各家厂商的适配器。
2. **返回值是通道，不是一次性结果**。这是刻意的设计——AI 生成文字需要时间，用通道可以一个字一个字往外吐，用户看到的是打字机效果。
3. 还有一个加强版接口 `IterModel`，支持更灵活的迭代式生成。普通模型实现 `Model` 就够了。

---

## 2. Tool（工具）——"手脚"

### 大白话

工具是智能体能主动使用的"功能按钮"。模型不会真的执行代码，它只会说"我想用哪个按钮、传什么参数"，然后由框架按下按钮。

### 代码里的定义

（`tool/tool.go`）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 最基本的工具：只要会"自我介绍"就行
type Tool interface {
    // 告诉别人：我叫什么、能干嘛、要什么参数
    Declaration() *tool.Declaration
}

// 会实际干活（执行）的工具，需要再实现这个
type CallableTool interface {
    Tool
    // 真正执行。参数是一段 JSON 字符串，返回值可以是任何东西
    Call(ctx context.Context, jsonArgs string) (any, error)
}

// 支持流式输出的工具（比如边执行边输出日志）
type StreamableTool interface {
    Tool
    // 边执行边往外吐结果
    StreamableCall(ctx context.Context, jsonArgs string) (<-chan *tool.Result, error)
}
```

### 大白话翻译

- `Declaration()`：工具的"自我介绍"——名字、用途说明、参数格式。这份介绍会被塞进给模型的提示词里，让模型知道"你有这些工具可以用"。
- `Call()`：工具被调用时真正执行的函数。参数是一段 JSON，返回值可以是任何东西。

### 框架内置了哪些工具？

`tool/` 目录下有很多现成工具，我按功能分类：

| 类别 | 工具 | 干什么 |
|------|------|--------|
| 计算/代码 | `codeexec`、`workspaceexec`、`hostexec` | 执行代码、在隔离工作区跑命令、在宿主机跑命令 |
| 联网 | `webfetch`、`duckduckgo`、`google`、`arxivsearch`、`wikipedia` | 抓网页、搜资料 |
| 办公 | `email`、`todo`、`file` | 发邮件、管理待办、读写文件 |
| 智能体协作 | `agent`、`transfer`、`taskrun` | 调用子智能体、把对话转给别的智能体 |
| 协议 | `mcp`、`openapi`、`vision` | 接入 MCP 服务、OpenAPI 接口、图片识别 |
| 框架自身 | `skill`、`memory`、`knowledge` | 读取技能说明书、读写记忆、查知识库 |

### 把任意 Go 函数变成工具

这是最常用的方式（上一篇的 calculator 就是）：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 把任意一个函数变成工具
tool, err := function.NewFunctionTool(myFunc, function.WithName("my_tool"))
```

框架会通过 Go 的反射机制自动分析 `myFunc` 的参数类型，生成参数格式说明（JSON Schema），模型就能照着格式调用它。

---

## 3. Agent（智能体）——"员工"

### 大白话

Agent 是**有大脑、有工具、会自己安排工作流程的 AI 角色**。它是你真正"雇佣"的那个员工。

### 代码里的定义

（`agent/agent.go`）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 所有智能体都要实现这个"标准接口"（Agent interface）
type Agent interface {
    // 开工：给它这次任务的"工单"（invocation），它边干边返回事件
    // 返回一个事件通道 <-chan *event.Event
    Run(ctx context.Context, invocation *agent.Invocation) (<-chan *event.Event, error)

    // 它手里有哪些工具
    Tools() []tool.Tool

    // 名字、描述等基本信息
    Info() *agent.Info

    // 它手下有没有子智能体
    SubAgents() []Agent

    // 按名字找手下
    FindSubAgent(name string) (Agent, error)
}
```

### 大白话翻译

- `Run`：开工。给它这次任务的"工单"（invocation），它边干边返回事件。
- `Tools`：它手里有哪些工具。
- `SubAgents`：它可以管着手下（子智能体），比如老板 Agent 把活分给员工 Agent。

### 框架提供了哪些现成 Agent？

（都在 `agent/` 目录下）

| Agent 类型 | 是什么 |
|-----------|--------|
| `llmagent` | **最常用**。一个大脑 + 一些工具，能自己决定要不要调工具 |
| `graphagent` | 按流程图（图）走固定流程的智能体 |
| `chainagent` | 串行：A 干完给 B，B 干完给 C |
| `parallelagent` | 并行：几个子智能体同时干活 |
| `cycleagent` | 循环：按条件反复执行某一步 |
| `a2aagent` | 能通过 A2A 协议跟别的系统里的智能体协作 |
| `claudecode` / `codex` | 接入外部编码智能体的适配器 |

---

## 4. Session（会话）——"点单记录本"

### 大白话

Session 是**一场对话的完整档案**：谁在聊、聊了哪些话、聊到哪了、临时状态是什么。

### 代码里的定义

（`session/session.go`）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 一场对话的完整档案（Go 里叫结构体 struct）
type Session struct {
    ID        string                 // 会话 ID（比如 session-001）
    AppName   string                 // 属于哪个应用
    UserID    string                 // 属于哪个用户
    State     map[string]any         // 临时状态（比如"正在等用户确认"）
    Events    []*event.Event         // 这场对话的所有事件（完整聊天记录）
    Tracks    map[string][]*event.Event // 按"轨道"分类的事件
    Summaries []*Summary             // 对话太长时生成的"摘要"
}
```

### 大白话翻译

- `Events`：完整聊天记录（所有发生过的事）。
- `Summaries`：聊了 100 轮之后，为了省 tokens（模型处理量），把前面的内容压缩成"摘要"，只把摘要 + 最近几轮喂给模型。
- `State`：一些临时标记，比如"这个智能体正在等用户回答一个问题"。
- `Tracks`：把事件按主题分开存，方便查找。

### Session 存在哪？

`session/` 目录下提供了很多"仓库"（后端存储）：

| 后端 | 说明 |
|------|------|
| `inmemory` | 存内存里，程序重启就没了（测试/开发用） |
| `redis` / `mysql` / `postgres` / `mongodb` / `sqlite` / `clickhouse` | 存各种数据库里（生产用） |
| `pgvector` / `mysqlvec` | 支持向量检索的版本（配合记忆/知识库） |

你只需要实现 `session.Service` 接口（取/存会话），就可以把会话存到任何地方。

---

## 5. Event（事件）——"传菜口的通知"

### 大白话

Event 是**系统每一步进展的"通知单"**。整个框架运行过程中，所有发生的事情都会变成事件，通过通道（channel）流出去。

### 代码里的定义

（`event/event.go`）Event 结构体关键字段：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 一个"打了标签的快递盒"：每一步进展的通知单
type Event struct {
    Response     any       // 如果是模型输出，这里装的就是输出内容
    RequestID    string    // 这次运行（Run）的编号
    InvocationID string    // 这次调用（Invocation）的编号
    Author       string    // 谁产生的：user / assistant / tool ...
    ID           string    // 事件自己的编号
    Timestamp    time.Time // 发生时间
    // ... 还有类型、过滤标签等
}
```

### 大白话翻译

`Event` 就是一个"打了标签的快递盒"：

- `Author`：谁寄的（用户、智能体、工具）。
- `RequestID` / `InvocationID`：属于哪一次任务。
- 内容：模型的话、工具的结果，或者框架的通知。

你的程序拿到事件流之后，可以：

- 把模型说的话实时显示给用户；
- 记录日志；
- 触发某个业务逻辑（比如工具执行完去刷新页面）。

---

## 6. Runner（调度器）——"店长"

### 大白话

Runner 是**整个系统的总指挥**。用户每说一句话，都从 `runner.Run(...)` 进入，由它安排：

1. 查/建这场对话的档案（Session）；
2. 选一个合适的智能体（Agent）；
3. 把历史记录整理好；
4. 把任务交给智能体跑起来；
5. 把智能体产生的事件转发给你。

### 代码里的定义

（`runner/runner.go`）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 整个系统的总指挥（Runner interface）
type Runner interface {
    // 用户每说一句话，都从这里进入。
    // 返回一条事件流，你的程序逐个处理。
    Run(ctx context.Context, userID, sessionID string, msg *event.Message, opts ...RunOption) (<-chan *event.Event, error)
}
```

### Runner 还能挂很多东西

创建 Runner 时，可以通过"选项"（Option）给它配齐各种服务：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

runner, err := runner.NewRunner(
    "app", agent,
    runner.WithSessionService(sessionSvc),     // 对应 WithSessionService(...)：会话存哪
    runner.WithMemoryService(memorySvc),       // 对应 WithMemoryService(...)：长期记忆存哪
    runner.WithKnowledgeService(knowledgeSvc), // 对应 WithKnowledgeService(...)：知识库
    runner.WithPlugins(plugin1, plugin2),      // 对应 WithPlugins(...)：插件（日志、护栏等）
)
```

这就是框架的"组合式设计"：**核心的 Runner 很简单，能力全靠往里挂组件**。

---

## 7. 六个概念怎么协作？（回顾图）

```
你（用户）
   │ 说了一句话
   ▼
┌─────────┐  查/建档案   ┌──────────┐
│ Runner  │ ───────────► │ Session  │ （聊天记录）
│ (店长)  │              └──────────┘
└─────────┘
   │ 选 Agent、发工单
   ▼
┌─────────┐  用大脑     ┌─────────┐
│  Agent  │ ──────────► │  Model  │ （生成回答）
│ (员工)  │             └─────────┘
└─────────┘
   │ 决定调用工具
   ▼
┌─────────┐  执行      ┌─────────┐
│  Tool   │ ◄───────── │ (结果回传)
└─────────┘           └─────────┘
   │ 所有进展都变成
   ▼
┌─────────┐
│  Event  │ ──► 流出通道，你的程序逐个处理
└─────────┘
```

---

## 8. 小结

| 概念 | 一句话记忆 |
|------|-----------|
| Model | 大脑：只会生成文字 |
| Tool | 手脚：真正执行动作的函数 |
| Agent | 员工：大脑+工具，会安排工作 |
| Session | 档案：一场对话的所有记录和状态 |
| Event | 通知：每一步进展都变成一个事件流出去 |
| Runner | 店长：总指挥，把上面所有东西串起来 |

下一篇：把仓库目录翻一遍，一张表看懂整个项目的"地图"。
