# 03 · 源码地图：一张表看懂整个仓库

> 阅读目标：把 `trpc-agent-go` 仓库的顶层目录过一遍，每个目录是干什么的，心里有张地图。
> 阅读方法：打开 `C:\Users\EDY\workplace\trpc-agent-go`，对照着看。看到不认识的目录就回来查这张表。

---

## 1. 总览

仓库根目录下有 30 多个文件夹。不要慌——它们可以分成几大块：

| 板块 | 目录 | 一句话 |
|------|------|--------|
| 核心运行 | `runner`、`agent`、`model`、`tool`、`event` | 框架的心脏：跑对话、思考、动手、发事件 |
| 记忆与状态 | `session`、`memory`、`artifact` | 聊天记录、长期记忆、产出物 |
| 知识与技能 | `knowledge`、`skill`、`planner` | 知识库（RAG）、技能说明书、计划 |
| 协作与编排 | `graph`、`team` | 流程图编排、团队协作 |
| 进化与评估 | `evolution`、`evaluation` | 自我进化、效果评测 |
| 外部接入 | `server`、`plugin`、`openclaw` | 对外提供接口（HTTP/AG-UI/A2A）、插件系统 |
| 基础设施 | `internal`、`telemetry`、`log`、`storage`、`prompt`、`codeexecutor` | 内部工具库、监控日志、存储适配、提示词处理、代码执行 |
| 示例与文档 | `examples`、`docs`、`benchmark`、`test` | 示例程序、文档、性能测试、集成测试 |

下面逐个讲。

---

## 2. 核心运行（最重要，优先读）

### `runner/` — 总指挥

**用户每说一句话，入口都在这里**（`runner.Run`）。

职责：查/建会话、选智能体、整理消息、跑智能体、转发事件。

值得看：`runner.go`（主流程，下一篇细讲）、`runner/trpcagent/`（和 tRPC 框架对接的版本）。

### `agent/` — 各种"员工"

各种智能体类型：

| 子目录 | 智能体类型 |
|--------|-----------|
| `llmagent` | 最常用：大脑+工具，会自己决定调用工具 |
| `graphagent` | 按流程图走 |
| `chainagent` / `parallelagent` / `cycleagent` | 串行 / 并行 / 循环 |
| `a2aagent` | 跨系统协作 |
| `claudecode` / `codex` | 接外部编码 AI |

值得看：`agent.go`（Agent 接口定义）、`llmagent/llm_agent.go`（智能体怎么工作）。

### `model/` — 各种"大脑"

各家模型厂商的适配器：

```
openai/ anthropic/ gemini/ ollama/ huggingface/ bedrock/ hunyuan/ ...
```

值得看：`model.go`（Model 接口）、`model/openai/`（最常用的适配器）。

### `tool/` — 各种"工具"

框架内置工具全集：

| 类别 | 例子 |
|------|------|
| 代码执行 | `codeexec`、`workspaceexec`、`hostexec` |
| 联网 | `webfetch`、`duckduckgo`、`google`、`wikipedia`、`arxivsearch` |
| 办公 | `email`、`file`、`todo` |
| 协议 | `mcp`（MCP 工具）、`openapi` |
| 智能体协作 | `agent`、`transfer`、`taskrun` |
| 框架功能 | `skill`、`memory`、`knowledge`、`vision` |
| 安全 | `safety`（⚠️ 第 10 篇的 pr-2089 就是新增这个） |

值得看：`tool.go`（Tool 接口）、`tool/function/`（函数变工具）。

### `event/` — "通知单"

定义 Event 结构体和事件类型。整个框架跑起来后，所有进展都变成 Event 流出去。

---

## 3. 记忆与状态

### `session/` — 会话档案

一场对话的完整记录（聊天记录、状态、摘要）。

存储后端：`inmemory`（内存）、`redis`、`mysql`、`postgres`、`mongodb`、`sqlite`、`clickhouse`。

⚠️ 第 10 篇的 pr-2088 就是在 `session/replaytest/` 新增了一个"回放一致性测试框架"。

### `memory/` — 长期记忆

跨会话的长期记忆（不只记住一场对话，而是记住"这个用户"）。

存储后端：`inmemory`、`redis`、`mysql`、`pgvector`、`sqlitevec`、`chromadb`、`mem0` 等。

记忆的实现方式：把重要信息存成"记忆条目"，用向量检索找相关记忆。`memory/tool/` 把记忆功能包装成工具，智能体可以主动"写记忆、查记忆"。

### `artifact/` — 产出物

智能体产生的文件/成果（比如生成的图片、报告、代码文件）。

存储后端：`inmemory`、`s3`、`cos`（腾讯云对象存储）。

---

## 4. 知识与技能

### `knowledge/` — 知识库（RAG）

给模型配"外接硬盘"：先把文档切块（`chunking`）、转成向量（`embedder`）、存进向量库（`vectorstore`），回答时先检索相关内容（`retriever`、`reranker`）再喂给模型。

第 8 篇详细讲。

### `skill/` — 技能说明书

Skill 就是一份 `SKILL.md` 说明书，告诉智能体"遇到某类任务时，按这个流程做"。框架支持把技能作为工具交给智能体，也支持动态查找技能。

第 9 篇详细讲。

### `planner/` — 计划器

让智能体先"做计划"再执行：`react`（思考→行动→观察循环）、`builtin`、`a2ui`。

---

## 5. 协作与编排

### `graph/` — 图编排

把智能体流程画成"图"：节点（Node）是步骤，连线（Edge）是转移条件。`graphagent` 就是按图跑的智能体。

第 6 篇详细讲。

### `team/` — 团队

多个智能体组成团队，按角色分工协作。

---

## 6. 进化与评估

### `evolution/` — 自我进化

回顾历史会话 → 提取做得好的方法 → 写成新的 Skill → 审核通过后发布。这就是"智能体越用越强"。

第 9 篇详细讲。

### `evaluation/` — 评估

评测智能体效果：定义评测集（`evalset`）、跑评测（`evaluator`）、算指标（`metric`）、出结果（`evalresult`）。

`evaluation/workflow/promptiter/` 是"提示词自动优化"工作流（PromptIter engine），第 12 篇你中标的 Issue #2003 示例就是复用它，把"评估 → 失败归因 → 优化 → 验证集回归 → 门禁 → 审计报告"串成一个闭环。

`examples/evaluation/promptiter_regression_loop/` 就是那个闭环示例（不需要 API Key，用假模型就能跑通）。

---

## 7. 外部接入

### `server/` — 对外服务

把智能体包装成可对外访问的服务：

| 子目录 | 提供什么 |
|--------|---------|
| `agui` | AG-UI 协议（给前端聊天界面用） |
| `openai` | OpenAI 兼容的 API（任何 OpenAI SDK 都能连） |
| `a2a` | A2A 协议（智能体之间互联） |
| `promptiter` | 提示词迭代服务 |
| `trpcagent` | tRPC 服务接入 |

### `plugin/` — 插件

在智能体运行前后"插一脚"的钩子：

`guardrail`（安全护栏）、`debuglog`（调试日志）、`errormessage`（错误处理）、`identity`（身份）、`messagemerger`、`toolcallid`、`toolsearch`。

### `openclaw/` — 一体化应用

一个相对独立的、功能完整的上层应用（有界面、有浏览器插件、有对话管理），可以理解成"用这个框架做出来的一个成品示例"。

---

## 8. 基础设施

| 目录 | 干什么 |
|------|--------|
| `internal/` | 框架内部实现细节（外部用户不需要直接使用）：状态管理、工具调用、会话路由、JSON 处理等 |
| `telemetry/` | 可观测性：调用链追踪、指标（OpenTelemetry） |
| `log/` | 日志库 |
| `storage/` | 通用存储适配（clickhouse、elasticsearch、milvus、s3 等） |
| `prompt/` | 提示词处理（模板渲染、缓存） |
| `codeexecutor/` | 代码执行器（本地、容器、e2b、jupyter 等） |
| `artifact/` | （上面已讲）产出物存储 |
| `planner/` | （上面已讲）计划器 |

---

## 9. 示例与文档

### `examples/` — 示例大全（学习利器）

60+ 个示例，每个都是一个独立的 Go 模块，覆盖几乎所有功能：

```
examples/llmagent/          # 最基础：一个智能体
examples/calculator...      # 计算器
examples/multiagent/        # 多智能体
examples/memory/            # 记忆
examples/knowledge/         # 知识库
examples/graph/             # 图编排
examples/evaluation/        # 评估
examples/evolution/         # 进化
examples/telemetry/         # 监控
...
```

**学习建议：先跑 `examples/llmagent/`，再看别的。** 每个示例都有 README。

### `docs/` — 官方文档

网站版文档的源文件（mkdocs 格式）。

### `benchmark/` — 性能测试

框架本身的性能基准测试。

### `test/` — 集成测试

端到端测试模块。

---

## 10. 仓库根目录的文件

| 文件 | 干什么 |
|------|--------|
| `README.md` | 官方英文说明（很多示例代码） |
| `README.zh_CN.md` | 中文版说明 |
| `AGENTS.md` | **给 AI/开发者的工程规范**：项目结构、编码规范、PR 要求（很值得读） |
| `go.mod` / `go.sum` | Go 模块依赖定义 |
| `.golangci.yml` | 代码检查配置 |
| `.coderabbit.yaml` | AI 代码审查机器人配置 |
| `.typos.toml` | 拼写检查配置 |

---

## 11. 一张"学习路线图"

如果你想按重要程度读源码，推荐顺序：

```
第一梯队（必读）：
  runner/runner.go         ← 总流程
  agent/agent.go           ← Agent 接口
  model/model.go           ← Model 接口
  tool/tool.go             ← Tool 接口
  event/event.go           ← Event 结构

第二梯队（常用）：
  agent/llmagent/          ← 最常用的智能体实现
  session/session.go       ← 会话结构
  memory/memory.go         ← 记忆接口
  tool/function/           ← 函数变工具

第三梯队（进阶）：
  graph/  knowledge/  skill/  evolution/  evaluation/  server/
```

---

## 12. 小结

仓库很大，但**核心只有一条主线**：`runner → agent → model/tool → event`，其余都是围绕这条主线扩展的能力（记忆、知识、技能、进化、评估、服务化）。

下一篇，我们钻进 `runner.go`，看看"用户说了一句话之后，内部到底一步步做了什么"。
