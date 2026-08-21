# 06 多智能体与图编排：一个员工不够，就开一家公司

> 阅读目标：搞懂"多智能体"解决什么问题、Team 的两种协作模式、以及用图（Graph）编排工作流是怎么回事。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\team\` 和 `C:\Users\EDY\workplace\trpc-agent-go\graph\`

---

## 1. 为什么要多个 Agent？

单个 Agent（员工）什么都会一点，但"什么都会"往往意味着"什么都不精"。真实世界里，一家公司不会让一个员工又当前台、又当会计、又当程序员——而是各司其职，需要协作。

多智能体就是这么回事：**让多个各有所长的 Agent 分工合作，完成单个 Agent 干不了（或干不好）的复杂任务。**

比如一个客服系统：

- 一个 Agent 负责识别用户情绪；
- 一个 Agent 负责查订单；
- 一个 Agent 负责退款；
- 一个协调者负责"这单该找谁"。

---

## 2. 两种协作模式：老板分活 vs 员工互相交接

项目里 `team` 包提供两种最常用的协作模式：

### 模式一：协调者模式（ModeCoordinator）

像公司里的**老板**：

- 有一个协调者 Agent（老板）；
- 其他 Agent（员工）被包装成"工具"挂在老板名下；
- 老板收到任务后，自己判断"这活该派给谁"，然后像调工具一样调用对应员工；
- 员工干完，把结果交回给老板，由老板汇总给用户。

### 模式二：Swarm 模式（ModeSwarm）

像**员工互相交接**：

- 没有老板，从"入口员工"开始接单；
- 员工觉得"这单不该我管"，就写一张**交接单**（transfer）把任务转给另一个员工；
- 后一个员工接单继续干，可能再转给下一个，直到有人能把活干完。

为了防止员工之间互相踢皮球踢个没完，Swarm 有几个安全设置：

| 设置 | 大白话 |
|------|--------|
| MaxHandoffs | 最多交接多少次（默认 20 次） |
| NodeTimeout | 每个员工最多干多久 |
| RepetitiveHandoffWindow | 发现"同样两个人来回转"就停下来 |

对应代码在：

```text
C:\Users\EDY\workplace\trpc-agent-go\team\swarm.go
```

---

## 3. Team 是什么？就是一个"能当普通员工用的团队"

在代码里，`Team` 实现了一个叫 `agent.Agent` 的接口——意思是：**从外面看，一个团队和单个 Agent 没什么区别，都能"接话、干活、回话"。**

这有个很大的好处：你可以把一个 Team 当成一个普通 Agent 塞进另一个更大的系统里，一层套一层，像俄罗斯套娃。这就是"可组合"。

```text
C:\Users\EDY\workplace\trpc-agent-go\team\team.go
```

---

## 4. 更底层的编排方式：SubAgent

在 Team 之外，单个 Agent 也可以带"手下"。用 `WithSubAgents` 选项给一个 Agent 挂一堆专业子 Agent：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

mainAgent, err := llmagent.New(
    "coordinator-agent",
    llmagent.WithModel(modelInstance),                     // 指定大脑
    llmagent.WithDescription("协调者 Agent，负责任务委派"), // 写介绍
    llmagent.WithSubAgents(mathAgent, weatherAgent),       // 挂一堆手下
)
```

框架内置了几种组合模式：

| 模式 | 大白话 |
|------|--------|
| ChainAgent | 流水线：一个干完传给下一个 |
| ParallelAgent | 同时开工：几个员工同时处理同一件事的不同方面 |
| CycleAgent | 循环：反复迭代直到满足条件 |

另外还有两个"工具化"技巧：

- **AgentTool**：把整个 Agent 包装成工具，别的 Agent 可以"调用"它；
- **Agent Transfer**：通过一个 `transfer_to_agent` 工具，让 Agent 之间直接交接任务（Swarm 的底层就是这个）。

---

## 5. 更自由的编排：图（Graph）

Team 适合"老板/交接"这种固定套路。但有些业务流程更复杂，比如：

```text
先做准备 → 问模型 → 如果模型想调工具就去执行工具 → 执行完再问模型 → 否则直接结束
```

这种"分叉、循环、并行"的流程，用"图"来画最清楚。**图就是"流程地图"：节点（Node）是步骤，连线是下一步去哪。**

图里有两个特殊节点：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

const (
    START = "__start__" // 入口：你的工作流从这里出发
    END   = "__end__"   // 出口：走完所有步骤到这里结束
)
```

你的工作流从 Start 出发，走完所有步骤，最后到 End 结束。

---

## 6. 节点和三种"函数"

每个节点都是一个步骤，节点的"工作内容"由函数决定：

| 函数类型 | 签名 | 大白话 |
|----------|------|--------|
| NodeFunc | `func(ctx, state) (any, error)` | 干一件活，更新一下状态 |
| ConditionalFunc | `func(ctx, state) (string, error)` | 看情况**选一条路**走 |
| MultiConditionalFunc | `func(ctx, state) ([]string, error)` | 看情况**同时走多条路**（并行） |

类比：NodeFunc 是"员工干活"，ConditionalFunc 是"在路口看红绿灯决定走哪条街"，MultiConditionalFunc 是"把文件复印三份，分三个部门同时处理"。

节点本身还可以挂模型（LLM 节点）、挂工具（工具节点）、挂别的 Agent（Agent 节点），甚至支持缓存和重试。代码在：

```text
C:\Users\EDY\workplace\trpc-agent-go\graph\graph.go
```

---

## 7. 怎么搭一张图：StateGraph

搭图的步骤就像搭积木，代码位置：

```text
C:\Users\EDY\workplace\trpc-agent-go\graph\state_graph.go
```

流程是：

1. `NewStateGraph(schema)`：先定好"状态"长什么样（整张图共用的数据格式）；
2. `AddNode(...)`：把步骤一个个加进去；
3. `SetEntryPoint(...)`：指定从哪个节点开始；
4. `SetFinishPoint(...)`：指定到哪个节点结束；
5. `Compile()`：编译成可执行的图；
6. `NewExecutor(graph)`：创建执行器跑起来。

这里的"状态（State）"可以理解为**所有步骤共用的工作台**：每一步干完活，都把结果放到工作台上，下一步从工作台拿材料继续干。这样步骤之间不用互相认识，只要约好"工作台上有啥"就行。

---

## 8. 什么时候用 Team，什么时候用 Graph？

| 情况 | 推荐 |
|------|------|
| 一个老板带几个专家，分工明确 | Team（协调者模式） |
| 客服/工单流转，谁都能接，互相转单 | Team（Swarm 模式） |
| 流程有明确先后、分叉、循环、并行 | Graph |
| 又想编排、又要 Team 那样的团队协作 | 两者可以结合，Team 也能包进 Graph |

---

## 9. 小结

| 概念 | 大白话 | 位置 |
|------|--------|------|
| Team | 一个"看起来像普通员工"的团队 | `team/team.go` |
| ModeCoordinator | 老板分活模式 | `team/` |
| ModeSwarm | 员工互相交接模式 | `team/swarm.go` |
| SubAgent | 挂在父 Agent 名下的专业员工 | agent 包 |
| StateGraph | 用图来编排工作流 | `graph/state_graph.go` |
| Start / End | 图的入口和出口 | `graph/graph.go` |
| ConditionalFunc | 路口看红绿灯选路 | `graph/graph.go` |

核心一句话：**一个 Agent 是员工，多个 Agent 是公司；Team 提供"老板分活"和"员工交接"两种管理模式，Graph 则把任何流程画成一张"地图"来执行。**
