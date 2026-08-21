# 05 工具系统：Agent 的"手脚"是怎么接上去的

> 阅读目标：搞清楚"工具（Tool）"在这个项目里到底是什么、怎么把一个普通函数变成工具、模型怎么调用工具，以及执行前有哪些安全检查。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\tool\`

---

## 1. 先打个比方：工具就是"手脚"

之前我们把模型（Model）比作"大脑"。但大脑光会想不行，还得会做事。比如：

- 用户问"帮我算一下 2+3"，大脑可以直接算；
- 但用户问"帮我查一下今天天气"，大脑没长眼睛，看不到外面；
- 用户问"帮我把这 100 个文件改名"，大脑再聪明，手也伸不进磁盘。

所以框架给大脑配了"手脚"——也就是**工具**。工具就是一段真实的代码，模型（大脑）决定"我要用哪个工具、传什么参数"，然后框架帮它真的去执行。

一句话总结：**模型负责"想"，工具负责"做"。**

---

## 2. 工具长什么样：`Tool` 接口

先看最核心的文件：

```text
C:\Users\EDY\workplace\trpc-agent-go\tool\tool.go
```

里面定义了一个非常简单的接口：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

type Tool interface {
    // 最基本的工具：只要会交出自己的说明书就行
    Declaration() *Declaration // 交出自己的"求职简历"（名字、用途、参数格式）
}

type CallableTool interface {
    Tool                                                // 在 Tool 的基础上多了个 Call 方法
    Call(ctx context.Context, jsonArgs string) (string, error) // 接收一段 JSON 格式的参数，执行后返回结果
}
```

大白话：

- `Tool` 接口只有一件事：**交出自己的说明书**（`Declaration`）。
- `CallableTool` 在 Tool 的基础上，多了一个**真正干活**的方法 `Call`，它接收一段 JSON 格式的参数，执行后返回结果。

为什么接口这么简单？因为框架并不关心工具内部是怎么实现的，它只关心两件事：**你叫什么、能干什么**（说明书），以及**怎么叫你去干活**（Call）。

---

## 3. 说明书（Declaration）和 JSON Schema

`Declaration` 就是工具的"求职简历"：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

type Declaration struct {
    Name         string // 工具的名字（比如 calculator）
    Description  string // 一句话说明这工具干嘛的
    InputSchema  Schema // 输入格式说明书：要传什么参数、什么类型
    OutputSchema Schema // 输出格式说明书：返回的结果长什么样
}
```

对应到现实：

| 字段 | 大白话 |
|------|--------|
| Name | 工具的名字（比如 `calculator`） |
| Description | 一句话说明这工具干嘛的（比如"做四则运算"） |
| InputSchema | **输入格式说明书**：要传什么参数、参数是什么类型 |
| OutputSchema | 输出格式说明书：返回的结果长什么样 |

`Schema` 就是"JSON Schema"——一个描述数据格式的标准写法。你不需要现在学它，只要知道：**它告诉模型"调用我的时候，参数该传什么、传几个、什么类型"。** 模型看了说明书才知道怎么正确地调用工具。

---

## 4. 最强魔法：把一个普通函数变成工具

项目里你不用手动去写那一大堆接口，有个"魔法工厂"可以一键把普通 Go 函数变成工具：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 一个普普通通的 Go 函数：两个数相加
type addReq struct {
    A int `json:"a"`
    B int `json:"b"`
}

type addResult struct {
    Sum int `json:"sum"`
}

func add(req *addReq) (*addResult, error) {
    return &addResult{Sum: req.A + req.B}, nil
}

// "魔法工厂"把它变成 AI 能用的工具
tool, err := function.NewFunctionTool(
    add,
    function.WithName("calculator"),             // 起名字
    function.WithDescription("计算两个数字的和"),  // 写介绍
)
```

这个工厂在：

```text
C:\Users\EDY\workplace\trpc-agent-go\tool\function\function_tool.go
```

它的神奇之处在于：你只需要写一个**普普通通的 Go 函数**（输入一个结构体，输出一个结构体），框架就自动帮你生成说明书、自动处理 JSON 参数的转换。`WithXxx` 这一系列选项就像填表时的"附加选项"：

| 选项 | 作用 |
|------|------|
| WithName / WithDescription | 起名字、写介绍（模型靠这个判断什么时候用它） |
| WithLongRunning | 标记这是耗时工具（比如要跑很久的） |
| WithSkipSummarization | 标记结果太啰嗦、不用被总结 |
| WithConcurrencySafe | 声明这个工具可以安全地同时被调用 |
| WithInputSchema / WithOutputSchema | 手动指定输入输出格式（不指定就自动推断） |

有个小规则：**工具名只能用英文、数字、下划线、连字符**。这就像"起名不能带生僻字"，因为模型和框架之间是靠名字来对暗号的。

---

## 5. 一次完整的"工具调用闭环"

工具不是框架自己主动调用的，而是**模型决定调用的**。整个流程像餐厅里点菜：

1. **菜单摆上桌**：框架把所有工具的说明书（Declaration）发给模型，模型知道"这家店有什么菜"。
2. **模型点菜**：模型说"我要用 calculator 工具，参数是 a=2, b=3"（这串内容就是 JSON 格式的参数）。
3. **后厨做菜**：框架收到指令，调用工具的 `Call` 方法，真的把函数跑起来。
4. **端菜上桌**：框架把执行结果（比如 5）返回给模型。
5. **模型继续**：模型看到结果，继续思考，要么再调下一个工具，要么给出最终回答。

这个循环（模型思考 → 调工具 → 拿结果 → 再思考）就是 Agent 能"干活"的核心机制，代码在 runner 里驱动，我们在第 4 篇已经见过它的身影。

---

## 6. 项目里现成的"工具超市"

框架自带了一大批现成工具，都在 `tool\` 目录下，每个子目录就是一个工具：

```text
trpc-agent-go\tool\
├── webfetch        # 抓网页内容
├── wikipedia       # 查维基百科
├── arxivsearch     # 查学术论文
├── duckduckgo      # 搜索引擎
├── google          # 谷歌搜索
├── email           # 发邮件
├── file            # 读写文件
├── codeexec        # 执行代码
├── workspaceexec   # 在隔离工作区里执行命令
├── hostexec        # 在宿主机上执行命令（更危险）
├── taskrun         # 运行子任务
├── todo            # 待办清单
├── vision          # 看图
├── mcp / mcpbroker # 对接 MCP 外部服务
├── openapi         # 对接 OpenAPI 接口
├── skill           # 加载技能（第 9 篇细讲）
└── agent           # 把另一个 Agent 当工具用（第 6 篇细讲）
```

你可以直接用它们，也可以照着它们的样子写自己的工具。工具多了以后，安全问题就来了——下一节是重点。

---

## 7. 工具安全：执行前的"安检门"

让模型能执行命令是好事，但也危险：万一模型（或被黑客诱导的模型）说"帮我执行 `rm -rf /`"（删光整个磁盘）呢？

所以项目加了一道**安检门**：在执行工具之前，先对命令做一次静态扫描，决定放行、拒绝还是问人。这就是 `tool/safety` 包，代码位置：

```text
C:\Users\EDY\workplace\trpc-agent-go\tool\safety\
```

用法非常简单：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

scanner := safety.NewScanner(nil)   // 用默认策略（nil 表示默认）
report := scanner.Scan(
    &safety.ScanRequest{
        ToolName: "workspace_exec",
        Command:  "rm -rf /",
        Backend:  "workspaceexec",
    },
)

if report.Decision == safety.DecisionDeny { // 拒绝执行
    fmt.Println("拒绝执行！")
}
```

扫描结果 `ScanReport` 会告诉你三件事：

| 字段 | 大白话 |
|------|--------|
| Decision | 决策：allow（放行）/ deny（拒绝）/ ask（去问人）/ needs_human_review（必须人工复核） |
| RiskLevel | 风险等级：low / medium / high / critical（低/中/高/致命） |
| RuleID + Evidence + Recommendation | 哪条规则判的、证据是什么、建议怎么办 |

安检员会按顺序检查 8 大类风险，全都在 `scanner.go` 里：

1. **危险命令**：`rm -rf`、覆盖系统目录、碰 `.ssh`、`.env` 等敏感文件；
2. **Shell 绕过**：`sh -c`、反引号、`$()`、管道 `wget | sh` 这类"绕道执行"的写法（解析器搞不定的就默认拒绝，绝不心大放行）；
3. **网络外连**：`curl`、`wget` 访问不在白名单里的域名；
4. **宿主机执行**：`top`、`tail -f`、编辑器等交互式长会话命令；
5. **依赖安装**：`go install`、`npm install`、`pip install` 等改环境的操作；
6. **资源滥用**：sleep 太久、输出刷屏、并发过高；
7. **敏感信息泄漏**：命令里带 API Key、token、私钥；
8. **禁止路径**：不允许碰的目录。

这个安检门可以接到框架的权限系统里，让**每次工具执行前都先过安检**。这个功能正是三个已合并 MR 之一（pr-2089）带来的，第 10 篇会专门讲。

权限相关的接口在：

```text
C:\Users\EDY\workplace\trpc-agent-go\tool\permission.go
```

里面定义了 allow / deny / ask 三种动作，以及 `PermissionPolicy` 接口——安检门就是靠实现这个接口，让框架"每次执行工具前先问它一声"。

---

## 8. 小结

| 概念 | 大白话 | 位置 |
|------|--------|------|
| Tool 接口 | 一个能交说明书的东西 | `tool/tool.go` |
| Declaration | 工具的"求职简历" | `tool/tool.go` |
| NewFunctionTool | 把普通函数一键变成工具 | `tool/function/function_tool.go` |
| 工具调用闭环 | 模型点菜 → 框架做菜 → 端回去 | runner 驱动 |
| PermissionPolicy | 工具执行前的权限问询 | `tool/permission.go` |
| tool/safety | 命令执行前的安检门 | `tool/safety/` |

核心一句话：**模型是大脑，工具是手脚；大脑看说明书决定用哪只手脚，手脚干完活把结果告诉大脑，而执行前会先过一道安检门。**
