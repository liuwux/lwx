# Day 11 · RAG 进阶（第二轮：重排救不回漏召回、融合分数不能判断「没答案」、追问先改写）（2026-11-08）

## 今日 AI 动态（2026-11-08）

过去 24 小时关注的厂商没有新的官方发布。

---

> 第二轮说明：第一轮 Day 11 讲过 BM25 原理、RRF 公式、cross-encoder 与 bi-encoder、Contextual Retrieval、多查询 / HyDE / 后退提问、GraphRAG 和「按诊断结果加技巧」。今天不再重复这些，只讲三个上线后才会碰到的边界问题：两阶段检索的上限在哪、融合之后怎么判断「知识库里没有」、多轮追问怎么检索。

## 读前三问

1. 你加了一个很强的重排模型，可故障码类问题的准确率还是上不去。最可能的原因是什么？该先查哪个指标？
2. 混合检索用 RRF 融合之后，能不能设一个分数阈值来判断「知识库里没有答案」？为什么？
3. 司机先问「后排座椅怎么加热」，接着问「那前面的呢？」。直接拿第二句去检索会怎样？怎么改？

## 从一个场景开始

你的车机「用车手册助手」已经用上了混合检索加重排，评测分数不错。

上线一周，客服转来三条投诉。

第一条：司机说「仪表报 P0420」，助手先讲了一大段「发动机失火要立即停车」，把司机吓得靠边。故障码那一块其实被检索到了，只是排在第二。

第二条：司机问「怎么加装拖车钩」，手册里根本没有这部分，助手却拿座椅加热和胎压的内容拼了一个答案。

第三条：司机问完后排座椅加热，又说「那前面的呢？」，助手回答了 P0420。

三件事都不是「模型不够聪明」。第一条是排序问题，第二条是「检索永远会返回点什么」，第三条是检索拿到的问题本身不完整。今天逐个拆开。

## 一句话总结

两阶段检索里，召回决定上限，重排只负责排序；RRF 只看排名，不能拿来判断有没有答案，要靠重排分数或评审模型；多轮追问要先结合历史改写成独立问题再检索。

## 先跑起来

用 LangChain 的 `EnsembleRetriever` 把向量检索和 BM25 合起来，再用一个假 LLM（`FakeListChatModel`）演示追问改写的链路结构。向量部分用一个只认识主题词的玩具 embedding，故意让它不认识 P0420 这种编号。

```python
# hybrid_followup.py —— 故障码靠 BM25、追问先改写、无关问题也会返回结果（玩具 embedding + 假 LLM，无需 API Key）
from langchain_core.documents import Document
from langchain_core.embeddings import Embeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.language_models.fake_chat_models import FakeListChatModel
from langchain_core.output_parsers import StrOutputParser
from langchain_community.retrievers import BM25Retriever
from langchain_classic.retrievers import EnsembleRetriever

TOPICS = [["排放", "催化", "故障灯"], ["座椅", "加热"], ["胎压", "轮胎"], ["前排"], ["后排"]]
class Concept(Embeddings):                     # 玩具“语义”向量：只认识主题词，不认识 P0420 这种编号
    def embed_query(self, t):
        v = [sum(w in t for w in ws) for ws in TOPICS] + [0.1]
        n = sum(x * x for x in v) ** 0.5
        return [x / n for x in v]
    def embed_documents(self, ts):
        return [self.embed_query(t) for t in ts]

texts = ["故障灯常亮：多为排放系统异常，检查催化器", "故障灯闪烁：发动机失火，立即减速停车",
         "胎压报警：前后轮胎压标准 2.4 bar", "后排座椅加热：长按后排空调面板座椅图标 3 秒",
         "前排座椅加热：点中控屏左下角座椅图标", "P0420：三元转化效率低于阈值，尽快到店"]
docs = [Document(t, metadata={"id": f"c{i}"}) for i, t in enumerate(texts)]
bigram = lambda s: [s[i:i + 2] for i in range(len(s) - 1)]   # 中文不能按空格切，用二元组
vec = InMemoryVectorStore.from_documents(docs, Concept()).as_retriever(search_kwargs={"k": 2})
bm25 = BM25Retriever.from_documents(docs, preprocess_func=bigram, k=2)
hybrid = EnsembleRetriever(retrievers=[vec, bm25], weights=[0.5, 0.5], id_key="id")
ids = lambda r, q: [d.metadata["id"] for d in r.invoke(q)]

q = "仪表亮故障灯，报 P0420"
print("① 向量:", ids(vec, q), " BM25:", ids(bm25, q), " 混合:", ids(hybrid, q))

history = [("human", "后排座椅怎么加热？"), ("ai", "长按后排空调面板座椅图标 3 秒。")]
rewrite = ChatPromptTemplate.from_messages(
    [("system", "结合对话历史，把用户最后一句改写成可独立检索的问题，只输出问题")]
    + history + [("human", "{q}")]
) | FakeListChatModel(responses=["前排座椅怎么加热？"]) | StrOutputParser()  # 换成 ChatAnthropic 即真实改写
raw = "那前面的呢？"
print("② 原句:", ids(hybrid, raw), " 改写后:", ids(hybrid, rewrite.invoke({"q": raw})))

print("③ 无关问题:", ids(hybrid, "怎么加装拖车钩"), "← 照样返回，混合检索不会说“没有”")
```

**运行**：`pip install -U langchain-core langchain-classic langchain-community rank_bm25 && python hybrid_followup.py`

**实测输出**（2026-11-08，Python 3.13，langchain-core 1.7.6，langchain-classic 1.0.8，langchain-community 0.4.2）：

```
① 向量: ['c1', 'c0']  BM25: ['c5', 'c1']  混合: ['c1', 'c5', 'c0']
② 原句: ['c5', 'c1', 'c4']  改写后: ['c4', 'c3']
③ 无关问题: ['c5', 'c1', 'c4'] ← 照样返回，混合检索不会说“没有”
```

（c0 故障灯常亮，c1 故障灯闪烁/失火，c4 前排座椅加热，c5 P0420。运行时还会看到一条 `langchain-community` 停止维护的提示，不影响结果。）

**该看什么**：
- ①：向量检索根本没找到 c5（P0420），BM25 找到了，融合后 c5 才进入候选。但 c5 只排第二，排第一的是「失火停车」，这就是场景里的第一条投诉，需要重排来修正顺序。
- ②：原句「那前面的呢？」检索到的是故障码和失火；改写成独立问题后，c4 排第一。
- ③：手册里没有拖车钩，检索照样返回 3 条。`EnsembleRetriever` 只返回排好序的文档，不返回分数，你连阈值都没地方设。

## 核心概念

### 1. 召回决定上限，重排只负责排序

- **是什么**：两阶段检索先「宽召回」拿一批候选，再「窄精排」挑出最相关的送给模型。重排只能在候选里重新排顺序，候选里没有的块，它救不回来。
- **没有它会怎样**：如果只看最终答案的准确率，你会以为「换个更强的重排模型」就能解决问题。可要是 P0420 那一块在第一阶段就没进候选（像 ① 里的纯向量检索），重排模型再强也没用。
- **对应你熟悉的**：列表页先用数据库按条件查出 200 条，再在内存里排序。排序写得再好，查询条件漏掉的数据也不会出现。

排查顺序：先看第一阶段的 recall@N（N 是送去重排的候选数），再看重排后的 MRR。Anthropic 的做法是先取 150 块，重排后留 20 块；他们试过 5、10、20 三种数量，20 效果最好。候选越多，重排越慢越贵，要在自己的评测集上找平衡点。

### 2. 融合只看排名，所以它不能告诉你「没答案」

- **是什么**：RRF 给每个文档的分数是「各路排名的倒数加权和」，只用排名、不用原始分数。所以 BM25 的大分数和向量的余弦值可以直接合并，不用先换算成同一个尺度。
- **没有它会怎样**：直接把两路分数相加，BM25 的分数没有上限，会压倒余弦值。但 RRF 也有代价：一个完全无关的问题，排第一的文档照样能得到最高分。分数只说明「谁排在前面」，不说明「到底相不相关」。
- **对应你熟悉的**：应用商店的「排行榜第一名」，只说明它比其他应用下载多，不说明它好用。

判断「知识库里没有」要靠能给出绝对相关度的环节：
- 重排模型的相关度分数加阈值（阈值用评测集定）；
- 一个评审步骤，比如 LangGraph 的 Agentic RAG 教程里，用模型对检索结果打「yes / no」，不相关就去改写问题；
- 生成时要求模型「资料里没有就说没有」，并让它引用块编号，方便核对。

### 3. 多轮追问要先改写成独立问题

- **是什么**：拿对话历史加上最后一句，让模型改写成一个不依赖上下文也能看懂的问题，再拿去检索。「那前面的呢？」改成「前排座椅怎么加热？」。
- **没有它会怎样**：检索只看到「那前面的呢」这几个字，没有主题词，返回的基本是随机内容（② 里拿到的是故障码）。车机语音里这种省略特别多，司机不会每次都把话说完整。
- **对应你熟悉的**：Navigation 跳转时只传了一个 `id`，目标页面还得知道它属于哪个列表才能加载。改写就是把隐含的上下文补进参数里。

几个边界：
- 改写结果只用于检索。生成答案时仍把原始对话交给模型，防止改写理解偏了还被当成用户原话。
- 改写会多一次模型调用，可以用小模型，或者只在句子很短、带「那 / 它 / 这个」等指代词时才触发。
- 含故障码、VIN 这类精确编号的句子，要求改写时原样保留编号。HyDE 这类「先编一个答案再检索」的方法会把编号编错，不适合这类问题。

### 4. 改写和重检索要有次数上限

- **是什么**：Agentic RAG 里常见的回路是「检索 → 评审不相关 → 改写 → 再检索」。这条回路要设上限，超过就转为「没找到，请联系客服」之类的兜底。
- **没有它会怎样**：LangGraph 官方 Agentic RAG 教程的图里，评审结果不是「yes」就会去改写，教程本身没有设重试计数。知识库里真没有答案时，这条回路会一直转，直到触发 LangGraph 的递归上限报错。
- **对应你熟悉的**：网络请求的重试策略，要有最大次数和退避，不能无限重试。

工程做法：在 State 里加一个 `rewrite_count`，在条件边里判断；再给每次检索记日志，统计「改写了几次才命中」，这是发现知识库缺内容的好信号。

## 高频面试题

**Q1：混合检索为什么常用 RRF 融合，而不是把两路分数直接相加？**
- BM25 分数没有上限，余弦值在 -1 到 1 之间，不在一个尺度上。
- RRF 只用排名，不需要归一化，也不用调分数的换算方式。
- 两路都排得靠前的文档会胜出，适合「语义 + 精确词」互补的场景。
- 两路的权重可以调，用评测集来定。

**Q2（追问）：那融合之后，怎么判断「知识库里没有答案」？**
- RRF 分数只反映相对排名，无关问题的第一名照样分最高，不能设阈值。
- 用重排模型的相关度分数加阈值，阈值在评测集上定。
- 或者加一步评审，判断检索结果是否相关，不相关就改写或兜底。
- 生成时要求「没有就说没有」并引用块编号；评测集里专门放一批「库里没有」的问题。

**Q3（场景题）：车机助手里，司机经常说「那前面的呢」「它多久换一次」这类省略句，检索效果很差，怎么改？**
- 检索前结合最近几轮对话，把最后一句改写成独立问题。
- 改写只用于检索，生成时仍带完整对话。
- 用小模型或只在出现指代词时触发，控制语音场景的延迟。
- 评测集里加入多轮样本，分别看改写前后的 recall。

**Q4（追问）：加了重排之后准确率还是不够，你会先换更强的重排模型吗？**
- 不会先换。先看第一阶段送进重排的候选里有没有正确的块，即 recall@N。
- recall@N 低：问题在召回，加 BM25、补块的上下文、调切块，或者增大 N。
- recall@N 高但 MRR 低：才是重排的问题，再考虑换模型或改重排提示词。
- 增大 N 会增加重排的延迟和费用，要和响应时间要求一起权衡。

## 常见误区

- **「重排模型越强，检索效果越好」**：重排只能在候选里调顺序。第一阶段漏掉的块，任何重排模型都找不回来。
- **「检索结果为空才说明没答案」**：向量检索和 RRF 融合几乎总会返回 k 条结果。是否相关要靠重排分数或评审步骤判断。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：混合检索 + 追问改写链**（取自 `hybrid_followup.py`）

```python
# 中文没有空格，用相邻两个字作为 BM25 的词
bigram = lambda s: [s[i:i + 2] for i in range(len(s) - 1)]
# 向量检索：用文档建内存向量库，转成检索器，每次取 2 条
vec = InMemoryVectorStore.from_documents(docs, Concept()).as_retriever(search_kwargs={"k": 2})
# 词法检索：BM25，指定中文分词函数，每次取 2 条
bm25 = BM25Retriever.from_documents(docs, preprocess_func=bigram, k=2)
# 混合检索：两路等权 RRF 融合，按 metadata 里的 id 去重
hybrid = EnsembleRetriever(retrievers=[vec, bm25], weights=[0.5, 0.5], id_key="id")
# 小工具：执行检索，只取出每条结果的 id
ids = lambda r, q: [d.metadata["id"] for d in r.invoke(q)]

# 对话历史：一问一答，用 (角色, 内容) 元组表示
history = [("human", "后排座椅怎么加热？"), ("ai", "长按后排空调面板座椅图标 3 秒。")]
# 改写链：系统指令 + 历史 + 最后一句，交给模型，输出纯文本
rewrite = ChatPromptTemplate.from_messages(
    # 系统指令：把最后一句改写成能独立检索的问题
    [("system", "结合对话历史，把用户最后一句改写成可独立检索的问题，只输出问题")]
    # 拼上历史，最后一句用占位符 {q}
    + history + [("human", "{q}")]
    # 接模型（这里是假模型），再接字符串解析器
) | FakeListChatModel(responses=["前排座椅怎么加热？"]) | StrOutputParser()
# 先改写，再用改写结果去混合检索
print(ids(hybrid, rewrite.invoke({"q": "那前面的呢？"})))
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 向量检索的条数写在 `as_retriever(search_kwargs={"k": 2})` 里；BM25 的条数直接是 `k=2`，两者写法不一样。
- `EnsembleRetriever` 的 `weights` 个数必须和 `retrievers` 一致；不设 `id_key` 时按正文去重。
- 改写链的输入是字典 `{"q": ...}`，不是直接传字符串；`bigram` 里是 `range(len(s) - 1)`，不是 `len(s)`。

**昨日复习（只做第③步）**：第二轮 Day 10 的「带来源 metadata 的切块、增量同步、按车型过滤检索」（`rag_sync.py`）。只看你自己写的注释重写，重点核对：`index()` 的前三个参数顺序是「文档、记账本、向量库」；`rm.create_schema()` 不能漏；`similarity_search_with_score` 返回的是元组。

## 动手练习（30 分钟）

在 `hybrid_followup.py` 的基础上：

1. 写一个 `rerank(query, docs)` 函数：问题里出现 `P` 加数字的编号时，把正文含这个编号的块提到最前面。让 ① 的混合结果变成 c5 排第一。
2. 给每个块手工打一个「和问题的相关度」（0 到 1），设阈值 0.5。让 ③ 的拖车钩问题输出「手册里没有找到相关内容」。
3. 把 `FakeListChatModel` 的回复改成「它多久换一次？」原样不变（模拟改写失败），观察检索结果，并在 `rewrite` 后加一个判断：改写结果和原句一样时，打印警告。

## 读后自测

1. 你加了一个很强的重排模型，可故障码类问题的准确率还是上不去。最可能的原因是什么？该先查哪个指标？——见「核心概念」第 1 节和 Q4
2. 混合检索用 RRF 融合之后，能不能设一个分数阈值来判断「知识库里没有答案」？为什么？——见「核心概念」第 2 节和 Q2
3. 司机先问「后排座椅怎么加热」，接着问「那前面的呢？」。直接拿第二句去检索会怎样？怎么改？——见「核心概念」第 3 节和「先跑起来」②

## AI 历史知识点：倒数排名融合 RRF（2009，滑铁卢大学）

**是什么**：Reciprocal Rank Fusion，一种把多个排序结果合成一个的方法。每个文档的得分是它在各个排序里「1 / (常数 + 排名)」的和，只看排名、不看原始分数。常数越大，排名靠前和靠后的差距越小。

**何时由谁提出**：滑铁卢大学的 Gordon V. Cormack、Charles L. A. Clarke 和 Stefan Büttcher，发表在 2009 年 7 月于波士顿举行的 SIGIR 会议上，是一篇只有两页（758–759 页）的短论文。标题直接给出了结论：RRF 的效果超过 Condorcet 投票融合，也超过单独的排序学习方法。

**为什么重要**：它几乎没有参数，不需要训练，也不需要把不同系统的分数换算到同一尺度。实现只要几行代码，效果却能和更复杂的方法打平甚至更好，所以成了「多路检索结果怎么合并」的默认答案。

**和今天的趋势有什么联系**：RAG 的混合检索正是「多路排序合一」的问题。LangChain 的 `EnsembleRetriever` 实现的就是加权 RRF，常数默认 60，源码注释直接链接到这篇论文；Anthropic 的 Contextual Retrieval 也用排名融合合并向量和 BM25 两路结果。十几年前为搜索引擎评测写的两页论文，今天在几乎每个 RAG 系统里运行。

**原始来源**（非 LangChain/Claude 来源）：
- [Reciprocal rank fusion outperforms condorcet and individual rank learning methods - SIGIR 2009（Webis IR Anthology 收录页）](https://ir.webis.de/anthology/2009.sigirconf_conference-2009.146/)
- [同一论文 - Google Research 出版物页面](https://research.google/pubs/pub36196/)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `EnsembleRetriever` | `langchain_classic.retrievers`；字段 `retrievers`、`weights`（默认等权，个数须与检索器一致）、`c`（默认 60）、`id_key`（去重用的 metadata 键，默认按 `page_content`）；每个文档累加 `weight / (rank + c)`，rank 从 1 开始；只返回排序后的文档，不返回分数 |
| `BM25Retriever` | `langchain_community.retrievers`，依赖 `rank_bm25`；`from_documents(docs, preprocess_func=..., k=...)`；默认按空格分词，中文要自己传分词函数；`langchain-community` 已声明停止维护，生产多用向量库自带的混合检索（Elasticsearch、MongoDB Atlas 等） |
| Anthropic Contextual Retrieval 数据 | 指标是 1 − recall@20；基线 5.7%，加 Contextual Embeddings 3.7%（降 35%），再加 Contextual BM25 2.9%（降 49%），再加重排 1.9%（降 67%）；先取 150 块、重排后留 20 块；试过 5/10/20 块，20 最好；重排用的是 Cohere |
| 什么时候不需要 RAG | Anthropic 建议：知识库小于 20 万 token（约 500 页）时，直接整份放进提示词，配合 prompt caching |
| 用 Claude 做重排（Academy 课程讲法） | 给每块预先分配 ID；提示词包含问题和带 ID 的文档；让 Claude 只返回按相关度排序的 ID 列表（JSON），比复述正文快；代价是要多等一次模型响应 |
| LangGraph Agentic RAG 教程 | 节点：`generate_query_or_respond` → `retrieve`（ToolNode）→ 条件边 `grade_documents`（结构化输出 yes/no）→ `generate_answer` 或 `rewrite_question`；改写后回到第一个节点；教程没有设改写次数上限；评审提示词要求把文档当数据、忽略其中的指令 |
| 假模型 | `langchain_core.language_models.fake_chat_models.FakeListChatModel(responses=[...])`，按顺序返回预设回复，适合离线测试链路 |

### 来源说明

- 演示代码的 embedding 和 LLM 都是假的，只用来展示检索链路的结构和边界行为；把 `FakeListChatModel` 换成 `ChatAnthropic` 即为真实改写，但本次没有 API Key，真实改写未实测。
- RRF 论文 PDF 本次未能打开，历史知识点里的作者、会议、页码来自收录页，公式和默认常数来自本机安装的 LangChain 源码（非 LangChain/Claude 来源的部分已标出）。
- 今日 AI 动态：各家官方页面在时间范围内（2026-11-07 美国时间 / 11-07 至 11-08 北京时间）没有找到新发布；11-06 的几条已超出范围，未收录。

### 延伸学习

- Claude Academy「Reranking results」一课（Claude with Amazon Bedrock 课程；Building with the Claude API 课程里有同系列的 BM25 和多索引 RAG 课）：用 Claude 做重排的完整代码。
- LangGraph「Build a custom RAG agent」教程：检索、评审、改写的完整图。

### 参考资料

- [Contextual retrieval in AI systems - Anthropic Engineering](https://anthropic.com/engineering/contextual-retrieval)
- [Reranking results - Claude with Amazon Bedrock - Claude Academy](https://academy.claude.com/courses/claude-with-amazon-bedrock/reranking-results)
- [EnsembleRetriever - LangChain Reference](https://reference.langchain.com/python/langchain-classic/retrievers/ensemble/EnsembleRetriever)
- [Build a custom RAG agent with LangGraph - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/agentic-rag)
