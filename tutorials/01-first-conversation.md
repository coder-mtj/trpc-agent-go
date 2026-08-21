# 01 · 从一次对话看整个流程（计算器例子）

> 阅读目标：跟着"用户问 2+3 等于几"这一句话，把 tRPC-Agent-Go 从头到尾走一遍，建立整体画面感。
> 这一篇只讲"发生了什么"，不讲细节实现；细节在后面的文章里。

---

## 1. 开场：一个能算数的智能体

官方 README 里有一个最经典的示例（文件在 `trpc-agent-go/README.md`），我们把它拆开看。

它做的事是：**造一个"会算数的 AI 助手"**。用户问它"2+3 等于几"，它先用模型理解这句话，然后调用一个叫 `calculator` 的工具算出结果，再回答用户。

完整代码大概 70 行，但真正"干活"的部分只有四步。我们一步步来。

---

## 2. 第一步：造一个"大脑"（模型）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 创建一个模型对象，底层调用 DeepSeek 的 deepseek-chat 模型
modelInstance, err := openai.New(
    "deepseek-chat",
    openai.WithVariant("deepseek"), // 对应 Go 的 WithVariant(...)
)
```

这行代码的意思是：**创建一个模型对象**，底层调用的是 DeepSeek 的 `deepseek-chat` 模型（OpenAI 兼容接口）。

大白话：你请了一位"大厨"。`openai.New(...)` 就是"签订雇佣合同"，指定用哪家的厨师（DeepSeek）、哪位厨师（deepseek-chat）。

> 注意：模型可以是任何"会说话的 AI 大脑"——OpenAI、DeepSeek、通义、本地模型（Ollama）、Google Gemini 都行，框架提供了很多"厂商适配器"（在 `model/` 目录下）。

---

## 3. 第二步：给一个"工具"（计算器）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 把一个普通的 Go 函数变成 AI 能用的"工具"
calculatorTool, err := function.NewFunctionTool(
    calculator,                             // 一个普通的函数（下面定义）
    function.WithName("calculator"),        // 对应 WithName(...)
    function.WithDescription("Execute addition, subtraction, multiplication, and division. ..."),
)
```

这里发生了这个框架最神奇的一件事：**把一个普通 Go 函数变成 AI 能用的工具**。

`calculator` 就是下面这个平平无奇的函数——两个数，一个运算符，返回结果：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 计算器的核心逻辑：两个数 + 一个运算符，返回结果
func calculator(req *calculatorReq) (*calculatorResult, error) {
    switch req.Op {
    case "add", "+":
        return &calculatorResult{Result: req.A + req.B}, nil
    case "sub", "-":
        return &calculatorResult{Result: req.A - req.B}, nil
    case "mul", "*":
        return &calculatorResult{Result: req.A * req.B}, nil
    case "div", "/":
        return &calculatorResult{Result: req.A / req.B}, nil
    default:
        return nil, fmt.Errorf("invalid operation: %s", req.Op)
    }
}
```

`function.NewFunctionTool(calculator, ...)` 做三件事：

1. **给工具起名**：`calculator`（模型靠名字认识它）。
2. **写使用说明**：`WithDescription` 里告诉模型"这个工具能算加减乘除，参数 a、b 是数字，op 是 add/sub/mul/div"。
3. **自动生成"参数说明书"**：框架会看 `calculatorReq` 结构体上的标签（`json` 和 `jsonschema`），自动生成一份"这个工具要什么参数"的格式说明，然后塞给模型看。

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// Go 里叫"结构体（struct）"，用它来描述工具的输入参数：
// 每个字段后面跟的小标签（json / jsonschema）就是"参数说明书"
type calculatorReq struct {
    A  int    `json:"a" jsonschema:"description=First integer operand,required"`
    B  int    `json:"b" jsonschema:"description=Second integer operand,required"`
    Op string `json:"op" jsonschema:"description=Operation type,enum=add|sub|mul|div,required"`
}

// 输出结构体：装计算结果
type calculatorResult struct {
    Result int `json:"result"`
}
```

大白话：就像给一个新手员工一份"设备使用手册"——设备叫 calculator、能干什么、需要输入什么参数，都写得清清楚楚。模型看了手册，就知道怎么正确"使用"这台设备。

---

## 4. 第三步：把"大脑 + 工具"装进"智能体"

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

agent, err := llmagent.New(
    "assistant",
    llmagent.WithModel(modelInstance),          // 对应 WithModel(...)：把大脑装进去
    llmagent.WithTools(calculatorTool),         // 对应 WithTools(...)：把工具塞给它
    llmagent.WithGenerationConfig(genConfig),   // 对应 WithGenerationConfig(...)：开启流式输出
)
```

`llmagent` 是框架提供的"最常用的智能体类型"：**一个会思考、会调用工具、会说话的 AI 角色**。

这里的 `"assistant"` 是给这个角色起的名字。`WithModel` 把大脑装进去，`WithTools` 把工具塞给它，`WithGenerationConfig` 开启"流式输出"（见第 8 节）。

---

## 5. 第四步：创建"调度器"（Runner）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

runner, err := runner.NewRunner("calculator-app", agent)
```

`Runner` 是整栋大楼的"物业经理"：谁来了、说了什么、聊天记录放哪、要调哪个智能体、事件怎么转发，全归它管。

这里创建了一个叫 `calculator-app` 的应用级 Runner，负责跑上面那个 `assistant` 智能体。

---

## 6. 高潮：用户发来一句话

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// Run 是框架的唯一主入口
events, err := runner.Run(
    ctx,
    "user-001",                                        // 谁在说话（用户 ID）
    "session-001",                                     // 哪一场对话（会话 ID）
    event.NewUserMessage("Calculate what 2+3 equals"), // 用户说了什么
)
```

`Run` 是框架的**唯一主入口**。它接收三个关键信息：

- **user-001**：这是谁。像登录账号。
- **session-001**：这是哪场对话。同一个用户在同一场对话里，能接着上一轮继续聊（记住上下文）；换一个 session 就是"重新开始"。
- **消息内容**：用户具体说了什么。

`Run` 返回一个事件通道（Go 里叫 `<-chan *event.Event`）——可以理解成**一条事件流**。

大白话：框架不是等全部干完才告诉你结果，而是**边干边往外扔"事件"**。比如：

1. "用户消息已收到"（事件 1）
2. "模型开始说话了……正在生成文字"（事件 2、3、4……每个字都可能是一个事件）
3. "模型决定调用 calculator 工具"（事件 5）
4. "工具执行完毕，结果是 5"（事件 6）
5. "模型基于工具结果，给出最终回答"（事件 7、8……）

你的程序就像在窗口前排队领包裹，**来一个事件拿一个**。

---

## 7. 处理事件流

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 事件流是一条"通道"（channel），一个接一个地来，用 for range 遍历
for ev := range events {
    // 如果这个事件是"模型生成的文字片段"，就把它打印出来
    if ev.Object == "chat.completion.chunk" {
        fmt.Print(ev.Response.Choices[0].Delta.Content)
    }
}
```

这段代码在"领包裹"：每收到一个事件，就看一下它是不是"模型生成的文字片段"（`chat.completion.chunk`），如果是，就把文字直接打印出来。

效果就是：用户会看到 AI 像打字机一样**一个字一个字蹦出来**，而不是等很久突然冒出整段话。

---

## 8. 整个流程串起来（重点）

把上面的代码合起来，一次完整对话是这样的：

```
用户："Calculate what 2+3 equals"
   │
   ▼
┌─────────────────────────────────────────────────────┐
│ Runner（物业经理）                                    │
│  1. 记录：user-001 在 session-001 说话了             │
│  2. 查/建这场对话的"档案"（Session）                 │
│  3. 找到该跑的智能体（assistant）                    │
│  4. 把用户消息 + 历史记录 + 工具手册 一起交给智能体   │
└─────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────┐
│ LLMAgent（智能体 = 大脑 + 工具）                     │
│  · 大脑（模型）看到："用户要算 2+3，我有个工具      │
│    calculator，参数是 a=2, b=3, op=add"              │
│  · 模型说："我不算，我要调用 calculator(a=2,b=3,    │
│    op=add)"  —— 这叫"工具调用"（Tool Call）          │
└─────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────┐
│ 框架执行工具：calculator(2, 3, add) → 返回 5        │
└─────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────┐
│ 模型看到工具结果"5"，最终回答：                     │
│ "2+3 equals 5"                                      │
└─────────────────────────────────────────────────────┘
   │
   ▼
一路上的"事件"通过 channel 流到你的程序，逐个打印出来
```

**最关键的一步是"工具调用"**：模型不直接算数，而是说"我要用 calculator 工具"，框架替它把工具跑起来，把结果（5）再喂回给模型，模型才给出最终回答。这就是"会思考又会动手"的完整闭环。

---

## 9. 几个值得注意的细节

1. **模型可以"要"工具，但不能自己"跑"工具**。它只是说"我要调用 calculator"，真正执行的是框架代码。这个设计很重要——意味着框架可以在工具执行前做安全检查（第 10 篇讲的就是这个）。
2. **工具调用可能发生多轮**。模型算完第一步，可能还要算第二步；每轮都是"模型说话 → 框架跑工具 → 结果喂回模型"。
3. **会话（session）让对话有记忆**。因为 Runner 每次都把这场对话的完整历史交给模型，模型才知道你前面聊了什么。
4. **事件（Event）是唯一的"对外出口"**。前端、日志、监控，都是靠订阅事件流来工作的。

---

## 10. 小结

这一篇你只需要记住四件事：

| 概念 | 一句话 |
|------|--------|
| Model | 大脑，只会说话 |
| Tool | 手脚，负责动手（执行函数） |
| Agent | 大脑 + 手脚组装成的"员工" |
| Runner | 物业经理，负责安排整个流程、管理档案、转发事件 |

下一篇，我们把这六个核心概念一个一个讲透。
