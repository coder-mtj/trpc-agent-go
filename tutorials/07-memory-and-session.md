# 07 记忆与会话：Agent 是怎么"记住"你的

> 阅读目标：搞懂 Session（一场对话）和 Memory（长期记忆）的区别，会话数据能存在哪些地方，以及记忆工具都有哪些。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\session\` 和 `C:\Users\EDY\workplace\trpc-agent-go\memory\`

---

## 1. 先分清两个词：会话 vs 记忆

一个 AI 应用要"记得"东西，其实有两种完全不同的记法：

| 概念 | 大白话 | 例子 |
|------|--------|------|
| Session（会话） | **这一场聊天的记录本** | 你今天和客服从"我要退货运费"聊到"地址怎么填"，全都在这一场对话里 |
| Memory（记忆） | **跨越很多场对话的长期笔记** | "这个用户是上海人"——下次他开新对话，你还能想起来 |

类比：Session 是**一次性纸杯**——喝完这杯就扔；Memory 是**随身的记事本**——换了杯子也带着。

---

## 2. Session：一场对话的完整档案

Session 的数据结构在：

```text
C:\Users\EDY\workplace\trpc-agent-go\session\session.go
```

一个 Session 大概长这样：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

type Session struct {
    ID        string         // 会话 ID
    AppName   string         // 应用名
    UserID    string         // 用户 ID
    State     map[string]any // 工作台：临时记住的状态
    Events    []*event.Event // 流水账：这场的每一条消息、每一次工具调用
    Tracks    []*Track       // 轨道记录：按类别分开记的流水
    Summaries []*Summary     // 摘要：对话太长时先浓缩一下
}
```

翻译成大白话：

| 字段 | 大白话 |
|------|--------|
| ID / AppName / UserID | 这场对话的"门牌号"：哪个应用、哪个用户、哪一场 |
| State | 工作台：临时记住的状态（比如"用户正在填写地址，填到省了"） |
| Events | 流水账：这场的每一条消息、每一次工具调用 |
| Tracks | 轨道记录（下面细讲） |
| Summaries | 摘要：对话太长了，先浓缩一下 |

框架用 **应用名 + 用户 ID + 会话 ID** 三个维度定位一场对话，就像用"小区 + 楼栋 + 房号"找到一户人家。第 4 篇里 `runner.Run` 的第 3 步就是在做这件事。

---

## 3. 会话可以存在哪里？——后端（Backend）

记录本本身只是一个"概念"，它到底记在哪儿，由**后端**决定。项目 `session\` 目录下有一大排：

```text
trpc-agent-go\session\
├── inmemory    # 记在内存里：最快，但程序重启就没了
├── sqlite      # 记在本地一个小文件里
├── mysql       # 记在 MySQL 数据库
├── postgres    # 记在 PostgreSQL 数据库
├── redis       # 记在 Redis 缓存里
├── mongodb     # 记在 MongoDB
├── clickhouse  # 记在 ClickHouse（适合大量日志分析）
├── pgvector    # PostgreSQL + 向量检索
├── tdsql       # 腾讯 TDSQL
└── noop        # 什么都不记（测试用）
```

类比：会话数据是"同一本日记"，你可以写在便利贴上（inmemory），也可以锁进保险柜（数据库）。**业务代码不用改，只换后端**——这正是后端模式的意义。

---

## 4. Track：给流水账"分赛道"

有时候一场对话里混着多种信息，像一条马路又走汽车又走行人。Track（轨道）就是把流水账按"类别"分开记：

```text
C:\Users\EDY\workplace\trpc-agent-go\session\track.go
```

`TrackService.AppendTrackEvent` 就是往某条轨道上追加一条记录。一个 `TrackEvent` 包含：

- **Track**：轨道名（比如 `user_preferences` 用户偏好）；
- **Payload**：具体内容；
- **Timestamp**：时间戳。

这样后续想查"用户偏好变化史"，直接看那条轨道就行，不用翻整本流水账。

---

## 5. Summary：对话太长怎么办？

聊到第 50 轮，把全部历史都发给模型既慢又费钱。于是框架会定期把前面的对话**浓缩成摘要**（Summaries），就像读书笔记——细节丢了，但主线还在。需要回忆时，先看摘要，再看最近几轮原文。

---

## 6. Memory：跨对话的长期记忆

Session 是"一场对话"的记忆，Memory 是"这个人"的记忆。它更像数据库里的一个表：

```text
C:\Users\EDY\workplace\trpc-agent-go\memory\memory.go
```

Memory 提供了 6 个记忆工具，都是给模型用的：

| 工具名 | 大白话 |
|--------|--------|
| memory_add | 记一条新笔记 |
| memory_update | 改一条旧笔记 |
| memory_delete | 删一条笔记 |
| memory_clear | 清空记忆 |
| memory_search | 搜记忆 |
| memory_load | 加载记忆 |

每条记忆还有"类型（Kind）"之分：

| Kind | 大白话 | 例子 |
|------|--------|------|
| fact | 事实 | "用户住在上海" |
| episode | 情节/经历 | "上个月用户咨询过退款流程" |

记忆同样可以存在不同后端：`inmemory`、`redis`、`sqlite`、`mysql`、`postgres`、`pgvector`、`sqlitevec`、`chromadb`、`mem0`、`tencentdb` 等。带 "vec" 的说明支持向量检索（第 8 篇会讲向量是什么）。

如果调用记忆工具时漏了参数，框架会明确报错"appName / userID / memoryID 必填"——这提醒我们：**记东西之前先想清楚记给谁。**

---

## 7. 会话、记忆、轨道、摘要的"一致性"问题

数据可以存到那么多后端里，就会冒出一个问题：

> 同一个用户、同一句话，存到 MySQL 和存到 Redis，结果应该一样吧？

理论上应该一样，但实际实现细节（比如"搜空字符串返回什么"）可能不同，这就是**一致性回归**。为了抓这种问题，pr-2088 专门做了一个"回放一致性测试框架"，把同一组输入分别喂给不同后端，再对比结果是否一致。第 10 篇会专门讲它。

---

## 8. 小结

| 概念 | 大白话 | 位置 |
|------|--------|------|
| Session | 一场对话的记录本 | `session/session.go` |
| 会话后端 | 记录本存放在哪（内存/数据库/缓存） | `session/` 各子目录 |
| Track | 给流水账分赛道 | `session/track.go` |
| Summaries | 对话太长时的浓缩摘要 | `session/` |
| Memory | 跨对话的长期记忆 | `memory/memory.go` |
| fact / episode | 事实型记忆 / 经历型记忆 | `memory/` |

核心一句话：**会话是"这一场的记录本"，记忆是"随身记事本"；两者都可以选择存放在不同的后端，而框架要保证换后端不换行为。**
