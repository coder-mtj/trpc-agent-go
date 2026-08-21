# 12 我中标的那一个 Issue：#2003 评估 + 优化自动闭环

> 阅读目标：读懂你认领的 Issue #2003 到底要做什么、示例代码是怎么实现的、为什么它能证明"优化真的有效而不是过拟合"。
> 阅读对象：对"提示词优化""模型评估"还不熟悉的人。这一篇把每个专业词都用大白话解释一遍。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\examples\evaluation\promptiter_regression_loop\`

---

## 1. 一句话总结

**这个 Issue 要做一个"自动改提示词、自动检验改得好不好、只有真变好才收下"的闭环流水线。**

大白话：我们的 AI 程序表现不好，通常是"给它的提示词（Prompt）写得不够好"。以前要人工一遍遍试提示词，又慢又容易越改越糟。这个示例做了一个全自动的"优化流水线"：

```text
先给当前提示词打分
  → 分析到底哪道题做错了、为什么错
  → 让 AI 优化器提出一版新提示词
  → 拿新提示词重新考试
  → 对比：是真的变好了，还是只是"背下了训练题"？
  → 通过四道检查才收下，并输出审计报告
```

整个流程不需要任何真实的模型 API Key（用一个"假大脑"就能跑），跑完会生成两份报告：一份给机器读（JSON），一份给人读（Markdown）。

---

## 2. 先搞懂几个词（大白话版）

### 2.1 提示词（Prompt）

就是我们写给 AI 的那段"工作说明"。比如：*"你是头条卡片生成器，请严格输出包含 headline 和 source 字段的 JSON。"*

**AI 表现好不好，很大程度取决于这段话说得清不清楚。** 这个 Issue 的核心就是：让 AI 自己把这段话改得更好。

### 2.2 评估（Evaluation）与评测集

评估就是"给 AI 出题、判分"。题目集合叫**评测集（EvalSet）**，每一道题是一个 **case**。

本示例的任务域是"生成结构化头条卡片"：输入一段 JSON（含 `headline` 标题和 `source` 来源），AI 要输出一个格式正确的卡片 JSON。判分规则只有两条：

| 判分规则（metric） | 考什么 | 不通过说明什么 |
|---|---|---|
| `headline_format_validity` | 输出是不是合法 JSON 格式 | 格式错了（format_error） |
| `headline_exact_match` | 输出和标准答案一字不差 | 内容答错了（final_response_mismatch） |

### 2.3 训练集 vs 验证集（最容易混的两个词）

- **训练集（train）**：AI 优化器"做作业"用的题。它看着这些题去改进提示词。
- **验证集（validation）**：AI 优化器"考试"用的题。**它做作业时绝对没看过这些题**，用来检验"改出来的提示词是真的变聪明了，还是只是把作业背下来了"。

本示例训练集 3 道题，验证集 3 道题（其中 `validation_02_ice_hockey` 是关键 case）。

### 2.4 过拟合（Overfitting）

**"背答案"现象。** 优化器把训练题全做对了，但一到没见过的验证题就露馅——因为它是"死记硬背"而不是"学会方法"。

这个 Issue 最看重的就是：**必须能识别并拒绝过拟合。**

### 2.5 失败归因（Attribution）

**分析"这道题为什么做错了"。** 是格式写错了？还是内容答错了？还是工具调用出错？给每个失败找出一个可解释的原因，AI 优化器才知道该往哪个方向改。

### 2.6 门禁（Gate）

**"验收委员会"。** 新提示词就算训练分提高了，也要过四道检查，全部通过才允许被"录用"（作为新的好版本，甚至建议回写进源文件）。

### 2.7 回写（Write-back）

把被接受的提示词**覆盖回源 prompt 文件**，让以后运行的程序用上更好的版本。

---

## 3. Issue #2003 到底要求什么

Issue 标题是"构建 **Evaluation + Optimization 的自动回归与提示词优化闭环**"（犀牛鸟开源活动专享，中高难度）。

要求做一个完整闭环：

```text
Baseline 评估 → 失败归因 → PromptIter 优化 → 验证集回归 → 接受门禁 → 审计报告
```

并且必须满足：

1. 输出 `optimization_report.json` 和 `optimization_report.md` 两份审计报告；
2. **用 fake model（假模型）也能完整跑通**，不需要 API Key；
3. 必须能拒绝"训练集提升但验证集回退"的过拟合轮次。

你提交的 PR #2419 就是这个示例：`examples/evaluation/promptiter_regression_loop/`。

---

## 4. 整个闭环长什么样（Go 伪代码）

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

func runRegressionLoop(cfg *Config) error {
    // ---------- 第 1 步：基线评估（Baseline）----------
    // 用当前提示词把训练集和验证集各考一遍，记住分数
    baselineTrain := evaluate(evalsetTrain)      // 0.00
    baselineVal := evaluate(evalsetValidation)   // 0.00

    // ---------- 第 2 步：失败归因（Attribution）----------
    // 分析训练集每道错题："是格式错，还是内容错？"
    reasons := attributeFailures(baselineTrain)

    acceptedPrompt := cfg.BaselinePrompt // 当前公认的好提示词

    // ---------- 第 3~6 步：一轮轮优化 ----------
    for roundNo := 0; roundNo < cfg.MaxRounds; roundNo++ { // 本示例最多 3 轮
        // 第 3 步：让"优化器"根据失败原因，提出一版新提示词
        candidate := optimizer.Propose(previousReasons)

        // 第 4 步：用新提示词重新考试（训练集 + 验证集都要考）
        trainScore := evaluateWith(candidate, evalsetTrain)
        valScore := evaluateWith(candidate, evalsetValidation)

        // 第 5 步：逐题对比验证集（delta），算"谁新过了、谁新挂了"
        deltas := compareCaseByCase(baselineVal, valScore)

        // 第 6 步：过门禁——四道检查全过，才允许"录用"
        if gate.PassCheck(
            gate.WithValScore(valScore),      // ① 验证分提升 ≥ 阈值？
            gate.WithDeltas(deltas),          // ② 新增"由过转败"的题 ≤ 上限？
            gate.WithKeyCases(cfg.KeyCases),  // ③ 关键题没回退？
            gate.WithCost(modelCalls),        // ④ 模型调用次数 / 耗时在预算内？
        ) {
            acceptedPrompt = candidate // 录用，作为新的对比基准
        } else {
            logReject(roundNo, gate.Reason) // 记下为什么拒绝
        }
    }

    // ---------- 第 7 步：审计报告 ----------
    return writeReport(
        "optimization_report.json",
        "optimization_report.md",
        acceptedPrompt,
    )
}
```

对应到 Go 代码，这个主循环就在 `pipeline.go` 的 `RunPipeline` 函数里。

---

## 5. 代码位置：每个文件是干什么的

示例在 `examples/evaluation/promptiter_regression_loop/`，一共 11 个 Go 源文件 + 数据目录：

| 文件 | 大白话 | 类比 Go |
|---|---|---|
| `main.go` | 程序入口：解析命令行参数、加载配置、调起流水线 | Go 程序都会有的 `func main()` 入口，调用流水线 |
| `config.go` | 读 `promptiter.json` 配置并校验 | 读配置文件 |
| `agent.go` | 构造"考生"（候选智能体）和优化流水线里的各个角色 | 工厂函数 |
| `model_fake.go` | **假大脑**：固定剧本的假模型，不用 API Key | 假的 API 客户端 |
| `pipeline.go` | **总导演**：编排上面 7 个步骤 | `main()` 里的主流程 |
| `attribution.go` | 失败归因分类器（为什么错） | 关键词分类函数 |
| `delta.go` | 逐题对比（新过 / 新挂 / 分数涨跌） | 对比函数 |
| `gate.go` | 四道验收检查（能不能录用） | 验收函数 |
| `report.go` | 生成 JSON / Markdown 报告 | 报告生成器 |
| `types.go` | 共用数据结构定义 | 数据类定义 |
| `*_test.go` | 每个模块的单元测试 | Go 自带 `go test` 的测试文件 |

---

## 6. 样例数据：头条卡片应用

数据都在 `data/headline-card-app/`：

| 文件 | 内容 |
|---|---|
| `train.evalset.json` | 3 道训练题（`train_01` ~ `train_03`） |
| `validation.evalset.json` | 3 道验证题（`validation_01_baseball`、`validation_02_ice_hockey`、`validation_03_badminton`） |
| `headline-card.metrics.json` | 2 条判分规则（合法 JSON + 精确匹配） |
| `promptiter.json` | 闭环配置（最多 3 轮、门禁阈值、预算等） |
| `prompt.txt` | 最初的提示词（baseline） |

`promptiter.json` 里最关键的几项：

```json
{
  "targetSurfaceID": "candidate#instruction",
  "maxRounds": 3,
  "gate": {
    "minScoreGain": 0.05,
    "maxNewHardFails": 0,
    "keyCaseIDs": ["validation_02_ice_hockey"],
    "maxModelCalls": 200,
    "maxLatencyMs": 180000
  }
}
```

意思：只优化"候选智能体提示词"这一处（`candidate#instruction`）；最多优化 3 轮；验证分要涨 0.05 以上；不允许新增"由过转败"的题；关键题 `validation_02_ice_hockey` 不能退步；整个流程模型调用不超 200 次、耗时不超过 3 分钟。

---

## 7. 三类场景：示例怎么"演"给你看

假模型按提示词里的"舞台标记"切换行为，确定性地覆盖 Issue 要求的三种情况：

| 轮次 | 优化器产出的提示词 | 训练分 | 验证分 | 结果 |
|---|---|---|---|---|
| Round 1 | `[STAGE_GOOD]` 泛化指令 | 0.00 | 1.00 | **优化成功**，engine 和 gate 都接受 |
| Round 2 | `[STAGE_OVERFIT]` 过拟合指令 | 0.67 | 0.67 | **验证集从 1.00 跌到 0.67**，训练升了但验证回退，gate 拒绝 |
| Round 3 | `[STAGE_INEFFECTIVE]` 无效指令 | 0.67 | 0.00 | **优化无效**，gate 拒绝 |

最终接受 **Round 1**，并建议把它的提示词回写到源文件。

**Round 2 是这个示例最有价值的地方**：优化器把训练题全做对了（训练分 0.00 → 0.67），但验证集从 1.00 掉到 0.67——`validation_01` 和 `validation_03` 从"通过"变成"失败"。这就是标准的**过拟合**，门禁的"验证分提升检查 + 新增 hard fail 检查"同时抓住它，干净利落地拒绝。

```
Round 2 逐题对比（相对已接受的 Round 1 baseline）：
validation_01_baseball     通过 → 失败   newly_failed（新挂了）
validation_02_ice_hockey   通过 → 通过   unchanged （关键题没退步）
validation_03_badminton    通过 → 失败   newly_failed（新挂了）
```

---

## 8. 为什么不需要 API Key：假大脑（fake model）

真实模型要花钱、要联网、每次回答还不一样，没法稳定复现"过拟合"这种场景。所以示例写了一个**完全确定性的假模型**（`model_fake.go`）。

它怎么"演"：

```go
// 概念版伪代码：真实项目就是 Go 实现

func fakeBrain(ctx context.Context, req *model.Request) (*model.Response, error) {
    text := req.Text
    // 看请求里带的是哪个角色
    if strings.Contains(text, "Optimize one PromptIter surface") { // 优化器角色
        return nextItem(optimizerPlan)                            // 按剧本依次吐出 3 版提示词
    }
    if strings.Contains(text, "Aggregate PromptIter gradients") { // 梯度汇总角色
        return aggregatorResponse()
    }
    if strings.Contains(text, "Compute PromptIter backward attribution") { // 反向归因角色
        return backwarderResponse()
    }
    // 否则是"考生"角色：按提示词里的舞台标记决定答案
    return candidateAnswer(text)
}
```

考生答案的规则（`model_fake.go` 里写死）：

| 提示词里的标记 | 考生的表现 |
|---|---|
| `[STAGE_GOOD]` | 验证集 3 题全对 + 训练题 `train_01` 对，故意留 `train_02/03` 错（让下一轮还有优化空间） |
| `[STAGE_OVERFIT]` | 训练题全对 + 关键验证题 `validation_02` 对，其余验证题全错（过拟合） |
| `[STAGE_INEFFECTIVE]` | 不管什么题都输出同一句固定文案（优化无效） |
| 没有标记（baseline） | 输出一句既不是合法 JSON、也不匹配的文案（全挂） |

优化器剧本（`optimizerPlan`）依次吐出这三版提示词，正好对应 Round 1 / 2 / 3。这就是为什么**不用真模型也能完整走通闭环、并且结果每次一模一样**（随机种子 42）。

---

## 9. 失败归因：三档置信度，保证每道错题都有解释

`attribution.go` 给每道失败的题找一个原因类别。一共有 7 类：

`final_response_mismatch`（答错）、`tool_call_error`（工具调用错）、`tool_argument_error`（工具参数错）、`route_error`（路由错）、`format_error`（格式错）、`knowledge_recall`（知识没召回）、`other`（其他）。

查找顺序是"由准到宽"：

1. **先看判分规则的名字**（比如 `headline_format_validity` 里有 `format`）→ 命中，置信度 **0.9**（最有把握）；
2. **名字没命中，再看失败原因的文字**（比如 reason 里有 "not valid" 或 "mismatch"）→ 命中，置信度 **0.7**；
3. **都查不到** → 兜底为 `other`，置信度 **0.5**（至少有解释，不算空）。

```go
// 概念版伪代码：真实项目就是 Go 实现

func attributeCase(caseScore *caseScore) *Attribution {
    if caseScore.Passed {
        return nil // 做对了不用归因
    }
    for _, metric := range failedMetrics(caseScore) {
        if matchKeyword(metric.Name) { // 第一档：规则名含关键词
            return &Attribution{Confidence: 0.9}
        }
    }
    if matchKeyword(strings.Join(caseScore.Reasons, " ")) {
        return &Attribution{Confidence: 0.7} // 第二档：失败文字含关键词
    }
    return &Attribution{Reason: "other", Confidence: 0.5} // 第三档：兜底
}
```

在本示例里：`headline_format_validity` 挂 → `format_error`；`headline_exact_match` 挂 → `final_response_mismatch`。报告里还有一个 `coverage` 字段，统计"有多少失败的题被归到了具体原因（而不是 other）"，用于衡量归因质量。

---

## 10. 门禁的四道检查 + 防过拟合策略

### 四道检查（`gate.go`，全部通过才录用）

| 检查 | 考什么 | 本示例配置 |
|---|---|---|
| ① `validation_score_gain` | 验证集总分提升 ≥ 阈值 | ≥ 0.05 |
| ② `no_new_hard_fail` | 新增"由过转败"的题 ≤ 上限 | ≤ 0 个 |
| ③ `key_cases_no_regression` | 关键题不能退步 | `validation_02_ice_hockey` |
| ④ `budget_within_limit` | 模型调用次数 / 耗时在预算内 | ≤ 200 次 / ≤ 180000 ms |

### 为什么它能抓住过拟合？

关键设计是：**每轮候选提示词都必须重新跑一遍验证集**，而且逐题对比是相对"当前已接受的 baseline"来算的（`delta.go`）。

也就是说，即使优化器把训练分刷上去了，只要验证集有题目从"通过"变成"失败"，第 ①、② 道检查就会同时报警。过拟合轮次**不会成为新的 baseline**，不会污染后续比较。

还有个细节：PromptIter 引擎只在"训练集还有失败"时才继续调用优化器。Round 1 故意留下 `train_02/03` 的失败，就是为了让第 2、3 轮还有"梯度"可优化，从而把三种场景都演出来。

---

## 11. 审计报告长什么样

跑完会在 `output/` 下生成两份报告：

### `optimization_report.json`（给机器读）

顶层有 8 大块：

| 字段 | 内容 |
|---|---|
| `pipeline` | 运行信息：名字、版本、运行 ID、开始时间、耗时、随机种子、模型 |
| `input` | 输入信息：训练/验证集 ID、判分文件、baseline 提示词、最多轮数 |
| `baseline` | 基线分数：训练分、验证分、逐题成绩 |
| `optimization` | 每轮优化的完整记录（候选提示词、训练/验证分、engine/gate 是否接受、逐题 delta、归因、调用次数） |
| `gate` | 最终门禁决策：接受与否、原因、四道检查各自结果 |
| `attribution` | 失败归因分布、覆盖度 |
| `cost` | 成本：模型调用次数、总耗时、是否在预算内 |
| `recommendation` | 最终建议：接受并回写哪一轮 |

### `optimization_report.md`（给人读）

是一份排版好的中文报告：基线评测表 → 每轮优化分数表 → 逐题 delta 表 → 门禁检查表 → 失败归因分布 → 成本预算 → 优化建议。

---

## 12. 怎么跑起来 + 对照验收标准

### 运行命令

```bash
cd examples/evaluation/promptiter_regression_loop
go run . -config data/headline-card-app/promptiter.json
```

可选参数：`-output-json`、`-output-md` 可以改报告输出路径。

跑单元测试：

```bash
go test ./...
```

测试覆盖：门禁决策（含过拟合/新增 hard fail/关键题回退/超预算全被拒）、逐题 delta 五类结果、失败归因分类精度、报告生成完整性、假模型各角色响应。

### 对照 Issue 验收标准

| Issue 验收项 | 示例怎么满足 |
|---|---|
| 6 条样例 case 全可跑，出完整报告 | 训练 3 条 + 验证 3 条，跑完生成 JSON/MD 报告 |
| 隐藏样本接受/拒绝准确率 ≥ 80% | 假模型确定性覆盖三类场景，测试断言 gate 决策正确 |
| "验证集回退但训练集提升"必须拒绝 | Round 2 完整演示并被拒 |
| 失败归因准确率 ≥ 75%，每题至少一个可解释原因 | 三档归因 + `other` 兜底，报告带 `coverage` |
| fake model 下完整流程 ≤ 3 分钟 | 配置上限 180000 ms，实际跑一次只要几十毫秒 |
| 报告含 baseline/candidate 分数、逐题 delta、gate 决策与理由 | 报告 8 大块全部覆盖 |

---

## 13. 小结

| 概念 | 一句话 |
|---|---|
| 提示词 | 写给 AI 的工作说明，本示例要优化它 |
| 训练集 / 验证集 | 做作业的题 / 没见过的考试题 |
| 过拟合 | 背答案，验证集一考就露馅 |
| 失败归因 | 解释每道错题为什么错（三档置信度） |
| 门禁 | 四道检查，全过才录用新提示词 |
| 回写 | 把被接受的提示词写回源文件 |

核心一句话：**这个示例解决的是"AI 提示词优化没人把关"的痛点——它不止会改提示词，还会用验证集和门禁证明"这版真的更好，而且不是背下来的"。**
