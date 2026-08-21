# 11 动手篇：怎么跑示例和测试

> 阅读目标：把前面的知识变成"亲眼看到"——教你编译并运行项目里的示例，以及跑测试，让你亲手验证一切。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\examples\`

---

## 1. 准备工作

### 需要装什么？

- **Go 语言环境**，版本 1.21 或更高（项目是 Go 写的，必须用 Go 来编译运行）。
- 大部分示例需要一个 **LLM API Key**（比如 OpenAI 兼容接口的 Key），通过环境变量提供：

```bash
export OPENAI_API_KEY="你的-key"
export OPENAI_BASE_URL="你的-接口-地址"   # 可选，默认连官方
```

> 小提示：Windows 下用 PowerShell 设置环境变量是：
> `$env:OPENAI_API_KEY="你的-key"`

### 没有 Key 怎么办？

没关系！很多示例不需要 Key 也能跑：

- 代码审查 Agent 的 `--dry-run` 模式（纯本地规则扫描）；
- 安全检查器的 `--demo` 演示；
- 回放一致性测试（纯测试，不需要联网）；
- 所有 `go test` 测试大多不需要 Key。

---

## 2. 跑第一个示例：交互式聊天（llmagent）

这个示例最直观：启动后你就能和 Agent 对话。

```bash
cd examples/llmagent
go run .
```

可选参数：

| 参数 | 作用 |
|------|------|
| -model | 指定模型名 |
| -streaming | 开启流式输出（一个字一个字蹦出来） |

这个示例的内部说明在 `examples/llmagent/README.md`，可以对照第 4 篇的 Runner 流程一起看。

---

## 3. 跑多智能体示例

想看"多个 Agent 分工协作"？

```bash
cd examples/team       # 团队协作（协调者 / Swarm 模式）
go run .
```

```bash
cd examples/multiagent # SubAgent 组合（流水线 / 并行 / 循环）
go run .
```

```bash
cd examples/graph      # 用图编排工作流
go run .
```

这些示例基本都要 API Key。跑之前可以先看看各自的 README 确认参数。

---

## 4. 跑 pr-2090：代码审查 Agent

这个不需要 API Key（dry-run 模式），强烈建议第一个动手试：

```bash
cd examples/code_review_agent
go run . --diff-file=testdata/security_issue/diff.patch --dry-run
```

执行完当前目录会生成：

- `review_report.json`：机器可读的审查结果；
- `review_report.md`：给人看的报告。

你可以打开 `review_report.md` 看看它从那个故意有安全问题的 diff 里抓出了什么。

常用参数：

| 参数 | 作用 |
|------|------|
| --diff-file | 指定 diff/patch 文件 |
| --dry-run | 只扫描不真跑沙箱（不需要 Key） |
| --fake-model | 用假模型做离线语义审查（不联网） |
| --repo-path | 指定真实仓库路径（用于跑 go vet） |
| --db-path | 审查结果存哪个 SQLite 文件 |

---

## 5. 跑 pr-2089：安全检查器演示

```bash
# 内置 14 个危险命令场景，一个个演示扫描结果
go run ./tool/safety/examples/demo --demo

# 自己指定一条命令试试
go run ./tool/safety/examples/demo --command "rm -rf /" --backend workspaceexec

# 指定策略文件
go run ./tool/safety/examples/demo --policy tool/safety/tool_safety_policy.yaml --command "curl https://api.github.com"
```

你会看到 `rm -rf /` 被拒绝（deny），并附上风险等级和建议。这是理解"工具安全检查"最快的办法。

---

## 6. 跑 pr-2088：回放一致性测试

```bash
# 全量回放测试（不到 2 秒）
go test ./session/replaytest/ -count=1

# 跨后端对比（把内存后端和文件持久化后端都拉进矩阵）
$env:REPLAYTEST_BACKENDS="inmemory,persistent"
go test ./session/replaytest/ -count=1 -run TestInMemorySessionReplayEventsStateAndMemoryMatch
```

测试跑完后，还可以看它生成的样例差异报告：

```text
session/replaytest/testdata/session_memory_summary_track_diff_report.json
```

---

## 7. 跑你中标的 Issue #2003：评估 + 优化闭环示例

这是第 12 篇讲的示例（`examples/evaluation/promptiter_regression_loop/`）。它默认用**假模型**，不需要 API Key，就能把"评估 → 失败归因 → 优化 → 验证集回归 → 门禁 → 审计报告"完整跑一遍：

```bash
cd examples/evaluation/promptiter_regression_loop
go run . -config data/headline-card-app/promptiter.json
```

跑完会在 `output/` 下生成两份报告：

- `optimization_report.json`：机器可读的结构化审计报告；
- `optimization_report.md`：给人看的中文报告（基线分、每轮候选分、逐题变化、门禁决策、归因分布、成本预算、回写建议）。

跑单测：

```bash
go test ./examples/evaluation/promptiter_regression_loop/...
```

想体会"过拟合被拒绝"的现场，直接看 `output/optimization_report.md` 里 Round 2：训练分涨了，但验证集从 1.00 掉到 0.67，门禁给出拒绝。

---

## 8. 跑整个项目的测试

仓库没有 Makefile，统一用 Go 原生命令：

```bash
go test ./...
```

这会编译并运行全仓库的测试。第一次跑会比较慢（要下载依赖、编译），属正常现象。

只跑某个包：

```bash
go test ./tool/safety/...      # 只跑安全检查相关
go test ./session/...          # 只跑会话相关
go test ./knowledge/...        # 只跑知识库相关
```

---

## 8. 常见问题

| 问题 | 原因 / 解决 |
|------|-------------|
| 提示找不到 API Key | 按第 1 节设置 `OPENAI_API_KEY`；或改用不需要 Key 的示例 |
| 下载依赖很慢 / 失败 | 国内网络可能需要配置 Go 代理：`go env -w GOPROXY=https://goproxy.cn,direct` |
| 编译报错版本过低 | 检查 Go 版本：`go version`，需要 1.21+ |
| 跑不起来没报错也没输出 | 有些示例是服务型，需要看 README 里的启动方式和参数 |

---

## 9. 小结

| 想体验什么 | 命令 |
|------------|------|
| 和 Agent 聊天 | `cd examples/llmagent && go run .` |
| 代码审查（无需 Key） | `cd examples/code_review_agent && go run . --diff-file=testdata/security_issue/diff.patch --dry-run` |
| 安全检查演示 | `go run ./tool/safety/examples/demo --demo` |
| 回放一致性测试 | `go test ./session/replaytest/ -count=1` |
| Issue #2003 闭环示例 | `cd examples/evaluation/promptiter_regression_loop && go run . -config data/headline-card-app/promptiter.json` |
| 全仓库测试 | `go test ./...` |

核心一句话：**先跑不需要 Key 的四个（code_review_agent 的 dry-run、safety 的 demo、replaytest、Issue #2003 闭环示例），建立信心后再上需要 Key 的对话示例，最后用 `go test ./...` 验证整个项目。**
