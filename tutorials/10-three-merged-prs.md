# 10 三个已合并 MR 分别做了什么

> 阅读目标：这是你最关心的部分——三个已合并的 MR（pr-2088 / pr-2089 / pr-2090）分别解决了什么问题、改了什么代码、怎么验证效果。
> 代码位置：本地仓库 `C:\Users\EDY\workplace\trpc-agent-go`，三个分支分别为 `pr-2088`、`pr-2089`、`pr-2090`。

---

## 1. 总览：三句话先记住

| MR | 一句话 | 新增内容 |
|----|--------|----------|
| pr-2088 | 给"会话/记忆存到不同后端"做了一个**一致性测试框架**，确保换后端行为不变 | `session/replaytest/`，约 10 个文件、3300 多行 |
| pr-2089 | 给工具执行加了**安全检查器**，危险命令先拦住再执行 | `tool/safety/`，约 16 个文件、4500 行 |
| pr-2090 | 做了一个**代码审查 Agent 示例**：给 diff 就能自动出审查报告 | `examples/code_review_agent/`，约 77 个文件、1 万多行 |

三个 MR 之间还有"裙带关系"：pr-2090 的代码审查流程里，直接复用了 pr-2089 的工具安全检查器。

---

## 2. pr-2088：会话回放一致性测试框架

### 它解决什么问题？

第 7 篇讲过，会话和记忆可以存在很多种后端里（内存、MySQL、Redis……）。理论上"同一句话存在哪都应该一样"，但不同后端的实现细节可能偷偷不一样——比如某个后端对"空查询"的返回顺序不同。

这种不一致很难发现，因为**每个后端单独测试都是绿的**，只有把结果摆在一起对比才露馅。

### 它是怎么做的？

思路非常直观，像"同一批试卷发给不同学生，对比谁答得不一样"：

1. **ReplayCase（同一套试卷）**：定义一组与后端无关的确定性输入——事件序列、记忆写入/查询、摘要步骤、轨道事件。
2. **驱动多个后端（不同学生）**：把同一套输入分别喂给内存后端、文件持久化后端等。
3. **归一化（Normalizer）**：每个后端返回的结果都剥掉噪声（ID、时间戳、排序等），变成统一的 Snapshot（快照）。
4. **对比（Comparator）**：两两对比快照，任何差异都会产出结构化的 DiffReport（差异报告），精确到"哪个会话、第几个事件、哪个字段"。

核心文件都在：

```text
C:\Users\EDY\workplace\trpc-agent-go\session\replaytest\
```

### 怎么证明它有效？

项目用"**故障注入**"来验收：故意往数据里塞毛病（比如丢掉一条摘要、把摘要写到别人的会话里），然后看测试框架能不能 100% 抓出来。结果：

| 验收项 | 结果 |
|--------|------|
| 10 条故障注入 case | 100% 检出 |
| 正常情况误报率 | ≤ 5% |
| 摘要丢失 / 覆盖 / 写错会话 | 100% 检出 |

运行全部测试（只要不到 2 秒）：

```bash
go test ./session/replaytest/ -count=1
```

---

## 3. pr-2089：工具安全检查器（tool/safety）

### 它解决什么问题？

第 5 篇讲过，Agent 能执行命令。但"能执行"和"能乱执行"是两码事——`rm -rf /`（删光磁盘）这种命令绝不能放行。

以前框架对"工具执行前有没有安全检查"没有统一答案，这个 MR 补上了：**在执行前，先静态扫描命令，给出放行 / 拒绝 / 询问的决定。**

### 它是怎么做的？

核心是一个扫描器：

```text
C:\Users\EDY\workplace\trpc-agent-go\tool\safety\scanner.go
```

调用方式：

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
```

返回的 `ScanReport` 包含：

| 字段 | 大白话 |
|------|--------|
| Decision | allow（放行）/ deny（拒绝）/ ask（问人） |
| RiskLevel | low / medium / high / critical |
| RuleID / Evidence / Recommendation | 哪条规则、证据、建议 |

扫描按顺序查 8 大类风险（详情在第 5 篇第 7 节）：危险命令、Shell 绕过、网络外连、禁止路径、允许命令白名单、环境变量白名单、资源滥用、宿主机风险。

有个设计特别值得说：**解析不了就默认拒绝**（fail-closed）。也就是说，遇到 `sh -c`、反引号、`$(...)` 这类"绕道执行"的写法，如果解析器搞不懂，宁可拒绝也不冒险放行。

### 配套的安全措施

| 配套 | 大白话 |
|------|--------|
| policy.go | 从 YAML 配置文件加载策略（严格模式：写错字段直接报错，防悄悄削弱安全） |
| audit.go | 审计日志：执行过什么命令，**只存命令的 SHA-256 哈希**，不存明文（防日志泄密） |
| otel.go | 遥测：每次扫描的结果埋点，方便监控 |
| examples/demo | 命令行演示工具，可以自己试 |

可以跑一下试试（`--demo` 内置了各种危险命令演示）：

```bash
go run ./tool/safety/examples/demo --demo
go run ./tool/safety/examples/demo --command "rm -rf /" --backend workspaceexec
```

---

## 4. pr-2090：代码审查 Agent 示例

### 它解决什么问题？

代码合并之前要审查，但人工看 diff 又慢又容易漏。这个 MR 做了一个**自动代码审查工具**：你把 git diff（改动内容）给它，它输出一份结构化的审查报告。

### 它是怎么做的？

入口是一个命令行程序：

```text
C:\Users\EDY\workplace\trpc-agent-go\examples\code_review_agent\
├── main.go              # 命令行入口
└── internal\
    ├── pipeline.go      # 核心管线
    ├── scanner.go       # 规则扫描器
    ├── sandbox.go       # 沙箱执行 + 安全检查
    ├── security.go      # 敏感信息脱敏
    ├── storage.go       # SQLite 落库
    └── reporter.go      # 生成报告
```

整条管线（`pipeline.go`）像一条流水线：

```text
解析 diff → 静态规则扫描 → 安全检查/沙箱执行 go vet → 去重
→ 敏感信息脱敏 → 存入 SQLite → 生成 JSON/Markdown 报告
```

静态规则覆盖 7 大类（共 14 条正则规则）：

| 类别 | 查什么 |
|------|--------|
| security | 硬编码密钥、SQL 注入、命令注入 |
| goroutine_context | goroutine 泄漏、context 没检查 |
| resource_cleanup | 文件没关、HTTP Body 没释放 |
| error_handling | 错误没检查、错误缺上下文 |
| test_coverage | 新文件没写测试 |
| db_lifecycle | 数据库连接没 Ping、没 Close |
| sensitive_info | 信用卡号、私钥泄露 |

审查结果会：

- 生成 `review_report.json`（机器可读）和 `review_report.md`（人可读）；
- 存进 SQLite 数据库（任务、发现的问题、沙箱执行记录、权限决定、模型调用等表），方便后续按任务 ID 查询。

### 怎么证明它有效？

项目自带一个评测集（benchmark）：12 个"有问题"样本 + 12 个"没问题"样本，要求：

| 指标 | 要求 |
|------|------|
| recall（有问题能查出来） | ≥ 0.8 |
| fp（没问题的别误报） | ≤ 0.15 |

### 怎么跑起来？

```bash
go run . --diff-file=testdata/security_issue/diff.patch --dry-run
```

`--dry-run` 是"只读模式"，不需要 API Key 也能跑，非常适合入门体验。想要更真实的语义审查，还有 `--fake-model` 选项，用确定性的假模型做离线审查，不联网。

---

## 5. 三个 MR 的关系和阅读建议

```text
pr-2088  ──> 保证数据后端一致（测试基础设施）
pr-2089  ──> 保证工具执行安全（安全基础设施）
pr-2090  ──> 用示例把工具/安全串起来（应用示例，复用了 pr-2089）
```

建议阅读顺序：

1. 先看 **pr-2090** 的 README（最有画面感，能直接跑出报告）；
2. 再看 **pr-2089** 的 README（理解"安全检查"是怎么实现的）；
3. 最后看 **pr-2088** 的 README（偏测试工程，理解了后端概念再看更顺）。

三个 MR 的完整说明文档都可以直接看：

```text
pr-2088 → session/replaytest/README.md
pr-2089 → tool/safety/README.md
pr-2090 → examples/code_review_agent/README.md
```

---

## 6. 小结

| MR | 类型 | 核心产出 | 一句话验收 |
|----|------|----------|------------|
| pr-2088 | 测试框架 | replaytest 回放对比 | 同一输入跨后端不一致能 100% 抓出 |
| pr-2089 | 安全能力 | tool/safety 安检器 | 危险命令执行前就被拦下 |
| pr-2090 | 应用示例 | 代码审查 Agent | 给 diff 出报告，漏报误报都达标 |

核心一句话：**2088 保证"数据没问题"，2089 保证"执行不危险"，2090 把两者用在真实的代码审查场景里给你看。**
