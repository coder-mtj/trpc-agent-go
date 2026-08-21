# 08 知识库与 RAG：给 Agent 配一个"图书馆"

> 阅读目标：搞懂 RAG 是什么、Agent 是怎么"查资料"的，以及知识库系统里每个零件（切块、向量、检索、重排）各自干什么。
> 代码位置：`C:\Users\EDY\workplace\trpc-agent-go\knowledge\`

---

## 1. 为什么要知识库？——开卷考试

模型（大脑）的"常识"是训练时学来的，但你公司的内部资料、产品手册、最新政策，模型并不知道。就像闭卷考试，考到没学过的内容就抓瞎。

RAG（Retrieval-Augmented Generation，检索增强生成）的思路特别简单：**把考试改成开卷。**

流程是：

1. 提前把资料整理好放进"图书馆"（知识库）；
2. 用户提问时，先从图书馆里**检索**出最相关的几段资料；
3. 把资料和问题一起交给模型；
4. 模型看着资料回答，准确率大大提升。

---

## 2. 入口：Knowledge 接口只有一个方法

```text
C:\Users\EDY\workplace\trpc-agent-go\knowledge\knowledge.go
```

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

type Knowledge interface {
    // 知识库的标准接口：整个知识库对外就一个动作——搜索
    Search(ctx context.Context, req *SearchRequest) (*SearchResult, error) // 给一个问题，返回一批相关资料
}
```

整个知识库对外就一个动作：**Search（搜索）**。你给一个问题，它给你一批相关资料。

`SearchRequest` 里可以带上：

| 字段 | 大白话 |
|------|--------|
| Query | 用户的问题 |
| History | 之前的对话（帮助理解上下文） |
| UserID / SessionID | 谁在问、哪场对话在问 |
| MaxResults | 最多返回几条 |
| MinScore | 相似度低于多少的不要 |
| SearchFilter / SearchMode | 过滤条件和搜索模式 |

返回的 `SearchResult` 里装着搜到的文档和相似度分数（Score）。

---

## 3. 图书馆是怎么建起来的：五件套

`knowledge\default.go` 里有一个现成的 `BuiltinKnowledge`（内置知识库），它内部拼了 5 个零件：

| 零件 | 大白话 | 代码目录 |
|------|--------|----------|
| chunking | 切块：把大文档切成小段 | `knowledge\chunking\` |
| embedder | 向量化：把文字变成"数字指纹" | `knowledge\embedder\` |
| vectorstore | 向量库：存这些指纹的地方 | `knowledge\vectorstore\` |
| retriever | 检索：找最像的几段 | `knowledge\retriever\` |
| reranker | 重排：把检索结果再精排一遍 | `knowledge\reranker\` |

它们像一条流水线：

```text
原始文档 → 切块 → 向量化 → 存入向量库
用户提问 → 向量化 → 从向量库检索 → 重排 → 返回给模型
```

---

## 4. "向量"到底是什么？——文字的指纹

向量这个词听起来吓人，其实可以这样理解：

把一句话交给一个专门模型（embedder），它会吐出一长串数字（比如 1536 个数）。这些数字合起来就是这句话的**"含义指纹"**：

- 意思相近的话，指纹也相近；
- 意思差很远的话，指纹差很远。

所以"检索"在向量库里其实是在算"哪段话的指纹离问题最近"。这样搜出来的不靠关键词硬碰，而是靠**意思匹配**——你问"怎么退款"，就算资料里写的是"退货款项处理"，也能搜出来。

---

## 5. 向量库有哪些选择？

向量库（vectorstore）就是专门存"指纹"并支持快速找近邻的数据库。项目支持：

```text
knowledge\vectorstore\
├── chromadb
├── inmemory     # 内存版，测试用
├── milvus
├── pgvector     # PostgreSQL 的向量插件
├── qdrant
├── tcvector     # 腾讯向量数据库
└── elasticsearch
```

跟 Session 后端一个道理：**业务代码不变，换存储底座。**

---

## 6. 其他零件：文档处理全家桶

知识库不只是"搜"，前面还有一大套文档处理流程，`knowledge\` 下这些目录都在干活：

| 目录 | 大白话 |
|------|--------|
| document | 文档模型：文档本身长什么样 |
| source | 知识来源：文件、目录、URL 等从哪导入 |
| extractor | 抽取：从 PDF、HTML 等提取正文 |
| transform | 转换：把格式转成 Markdown/文本 |
| ocr | 识别：扫描件图片转文字 |
| query | 查询处理：把问题整理得更适合检索 |
| searchfilter | 过滤：按元数据筛选知识 |
| graph / graphstore | 知识图谱：实体之间的关系网 |

---

## 7. 怎么把知识库接到 Agent 上？

有几种接法，官方文档在：

```text
C:\Users\EDY\workplace\trpc-agent-go\docs\mkdocs\zh\knowledge\index.md
```

最推荐的是"手动创建搜索工具"：

```go
// 概念版伪代码：用 Go 语法演示逻辑，真实项目就是 Go 实现

// 把"知识搜索"变成一个普通工具，模型就能按需调用它了
tool, err := knowledge.NewKnowledgeSearchTool(knowledgeInstance)
```

这样知识搜索就变成一个普通工具，模型按需调用（符合第 5 篇的"工具调用闭环"）。如果想省事，也可以用 `WithKnowledge()` 选项让框架自动挂一个 `knowledge_search` 工具。

---

## 8. 小结

| 概念 | 大白话 | 位置 |
|------|--------|------|
| RAG | 开卷考试：先查资料再回答 | `knowledge/` |
| Search | 知识库唯一入口：给问题返回资料 | `knowledge/knowledge.go` |
| chunking | 把大文档切成小段 | `knowledge/chunking/` |
| embedder | 把文字变成数字指纹 | `knowledge/embedder/` |
| vectorstore | 存指纹的库 | `knowledge/vectorstore/` |
| retriever / reranker | 先粗找、再精排 | `knowledge/retriever/`、`knowledge/reranker/` |
| NewKnowledgeSearchTool | 把知识搜索变成工具给模型用 | `knowledge/` |

核心一句话：**RAG 就是"开卷考试"——资料提前切块、变成数字指纹存进向量库，提问时先检索最相关的段落，再连同问题一起交给模型回答。**
