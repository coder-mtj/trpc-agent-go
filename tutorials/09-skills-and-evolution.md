# 09 Skills 与进化：让 Agent 学会"写操作手册"

> 阅读目标：搞懂 Skill（技能）是什么、SKILL.md 怎么组织、Agent 怎么按需加载技能，以及 Evolution（进化）怎么让 Agent 自己从经验里总结技能。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\skill\` 和 `C:\Users\EDY\workplace\trpc-agent-go\evolution\`

---

## 1. Skill 是什么？——一本"操作手册"

假设你是一家餐厅的新员工，第一次做红烧肉手忙脚乱；但老师傅有本**菜谱**，上面写着：

```text
红烧肉菜谱：
1. 五花肉切块焯水
2. 炒糖色
3. 加料酒生抽小火炖 40 分钟
...
```

你照着做，一次就成功。Skill（技能）就是给 Agent 的"菜谱"：**一份描述"遇到某类任务该怎么一步步做"的说明文档。**

有了技能，Agent 遇到熟悉的任务就不用从头摸索，直接照着手册干，又快又稳。

---

## 2. 一个技能就是"一个文件夹 + SKILL.md"

在项目里，一个技能就是一个文件夹，里面最核心的文件叫 `SKILL.md`：

```text
skills/
└── code-review/          # 技能名（文件夹名）
    ├── SKILL.md          # 核心：技能的目标、说明、步骤
    ├── references/       # 附加参考文档（可选）
    │   └── security.md
    └── scripts/          # 配套脚本（可选）
```

`SKILL.md` 的开头一般有"简介"（front matter），里面是技能的名字和一句话介绍；正文才是真正的操作步骤。代码里的定义在：

```text
C:\Users\EDY\workplace\trpc-agent-go\skill\repository.go
```

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

type Skill struct {
    Summary Summary  // 名字 + 一句话介绍（Name + Description）
    Body    string   // SKILL.md 的正文（真正的操作步骤）
    Docs    []string // 附加参考文档（references 目录里的文件）
}
```

Repository（技能仓库）就是管理这些技能的地方，接口只有三个方法：

| 方法 | 大白话 |
|------|--------|
| Summaries() | 列出所有技能的名字和简介 |
| Get(name) | 按名字取某个技能的完整内容 |
| Path(name) | 找到某个技能在磁盘上的位置 |

技能放在哪？由一个环境变量指定：

```text
SKILLS_ROOT   # 技能根目录
```

---

## 3. 三层加载：不一次性全塞给模型

一个技能可能很长，如果把所有技能全文都塞给模型，上下文一下子就爆了。所以采用**按需加载**，分三层：

1. **概览层**：只把每个技能的"名字 + 一句话介绍"给模型，成本极低。模型知道"有这些技能，各自管什么"。
2. **正文层**：模型确定要用某个技能时，调用 `skill_load` 工具，把那个技能的 `SKILL.md` 正文加载进来。
3. **文档/脚本层**：需要更细的资料时，再加载 `references` 文档；脚本不在提示词里跑，而是在工作区里执行，把结果拿回来。

类比：**先看菜单点菜，菜端上来了再动筷子，而不是把整个后厨的菜谱都背下来。**

---

## 4. 进化（Evolution）：让 Agent 自己写菜谱

光有老师傅给的菜谱还不够——新员工干活干多了，也会有自己的心得。Evolution 做的事就是：**让 Agent 从自己的对话经历里，自动总结出新的技能（SKILL.md），下次照着用。**

代码位置：

```text
C:\Users\EDY\workplace\trpc-agent-go\evolution\service.go
```

用法大致是：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

service, err := evolution.NewService(reviewModel)
service.EnqueueLearningJob(job)   // 把一段对话经历丢进学习队列
// ...
service.Close()
```

它跑在**后台**（异步），不耽误主任务。整个流程是一条流水线：

| 环节 | 大白话 |
|------|--------|
| ReviewPolicy | 先判断"这段经历值不值得学"（比如默认至少调过 3 次工具才算有料） |
| Reviewer | 让一个评审模型（LLM）看对话记录，提炼出技能草稿 |
| Reconciler | 去重、合并：和已有技能重了就吸收掉 |
| SpecGate / SafetyGate | 质量检查：格式对不对、有没有危险内容 |
| EffectivenessGate | 效果检查：这次任务失败的经历不写进手册 |
| HumanGate | 可选的人工审批 |
| Publisher | 把最终 SKILL.md 写到磁盘上 |

这套机制默认是**关闭**的（需要显式开启），因为"让 AI 自己写文件"涉及安全边界——你可以选择只允许应用级或用户级范围内生效。

效果有多好？官方基准测试说：加载技能后，相似任务能省 17%–33% 的 token（也就是钱和时间的开销），某些容易卡死的场景最高省 94.6%。

---

## 5. 小结

| 概念 | 大白话 | 位置 |
|------|--------|------|
| Skill | 一本操作手册（文件夹 + SKILL.md） | `skill/` |
| Repository | 管理技能的手册架 | `skill/repository.go` |
| skill_load | 模型按需加载技能的工具 | `tool/skill/` |
| SKILLS_ROOT | 技能放哪的环境变量 | `skill/` |
| Evolution | 从对话经验里自动总结新技能 | `evolution/service.go` |
| Reviewer → Publisher | 学技能的流水线：提炼→检查→发布 | `evolution/` |

核心一句话：**Skill 是 Agent 的操作手册（SKILL.md），按需加载不占脑子；Evolution 则是让 Agent 干完活后自己总结经验、写成新手册，下次照着抄作业。**
