# Day 11 · RAG 进阶：混合检索、重排、查询改写、HyDE、GraphRAG（2026-10-09）

## 一句话总结

基础 RAG 的瓶颈通常在"召回"和"排序"：**混合检索**（向量 + BM25，用 RRF 融合）补上精确词（故障码、ID）的漏召回；**重排**（cross-encoder / LLM）把"宽召回的 150 条"精排成"送进模型的 20 条"；**查询改写 / 多查询 / HyDE** 在检索前修正问题本身；**Contextual Retrieval** 在入库时给每块补上下文；**GraphRAG** 用知识图谱回答"跨文档连线"和"全局总结"类问题。每一步都要用 recall@k / MRR 证明它值得那份延迟和成本。

## 核心概念

Day 10 搭好了 Load → Split → Embed → Store → 检索 → 生成的基础管线，并留下两个坑：故障码这类精确字符串纯向量检索容易漏；单个 chunk 缺上下文。今天逐个补。Day 12 再讲"检索作为 Agent 的工具"和多跳检索。

### 0. 全景：进阶技巧插在管线的哪一段

```
           入库阶段（离线）                             查询阶段（在线）
文档 ─> 切块 ─> [Contextual Retrieval     问题 ─> [查询改写/多查询/HyDE] ─┐
                给每块加 50~100 token 上下文]                              │
              │                                                      ▼
              ├─> 向量索引 ───────────── 语义检索 top-150 ──┐
              └─> BM25 索引 ──────────── 词法检索 top-150 ──┴─> RRF 融合去重
                                                               │
                                                     重排（cross-encoder / LLM）
                                                               │
                                                     top-20 ─> LLM 生成
```

记一个口诀：**入库补上下文，检索两路并，融合看排名，精排再截断。**

### 1. 混合检索（Hybrid Search）：向量 + BM25

**为什么向量检索会漏？** Claude Academy《BM25 lexical search》一课的例子：查一个事故编号 `INC-2023-Q4-011`，语义检索返回的是"意思相近"的财务分析段落，压根没提这个编号。embedding 擅长"意思"，不擅长"字面"。

**BM25 是什么？** 一个词法排序函数，可以看成 TF-IDF 的改进版：
1. 把查询切成词；
2. 统计词频（TF）；
3. 按稀有度加权（IDF）——"的、是"权重低，`P0420` 这种稀有词权重高；
4. 再加上**文档长度归一化**和**词频饱和**（同一个词出现 10 次不会比 3 次高出太多）。

Anthropic 的 Contextual Retrieval 文章总结：BM25 对"唯一标识符、技术术语"特别有效，正好补 embedding 的短板。

**怎么合并两路结果？——倒数排名融合 RRF（Reciprocal Rank Fusion）**

两路分数不在一个量纲（BM25 分数无上限，余弦在 -1~1），**不能直接相加**。RRF 只看排名：

```
RRF_score(d) = Σ_i  w_i / (k + rank_i(d))          k 常取 60
```

Academy《A Multi-Index RAG pipeline》的手算例子（为看清数字用 k=1）：

| 文档 | 向量排名 | BM25 排名 | RRF（k=1） |
|---|---|---|---|
| Section 2 | 1 | 2 | 1/2 + 1/3 = **0.833** |
| Section 6 | 3 | 1 | 1/4 + 1/2 = 0.75 |
| Section 7 | 2 | 3 | 1/3 + 1/4 = 0.583 |

两路都排得靠前的文档胜出。`k` 越大，排名差距的影响越平缓。

**LangChain 里怎么写？** `EnsembleRetriever`（在 `langchain_classic.retrievers`）做的就是**加权 RRF**：源码里每个文档累加 `weight / (rank + c)`，`c` 默认 60，`weights` 默认等权；`id_key` 指定按哪个 metadata 字段去重（不设就按 `page_content`）。BM25 用 `langchain_community.retrievers.BM25Retriever`（依赖 `rank_bm25`，`k` 默认 4）。注意 `langchain-community` 官方已声明停止维护，生产里更常用向量库自带的混合检索（如 Elasticsearch、MongoDB Atlas 的 hybrid search retriever）。

> **中文坑**：`BM25Retriever` 默认按空格分词，中文整句会被当成一个词。必须传 `preprocess_func`（用 jieba 分词，或像下面代码那样用汉字二元组）。

**权重怎么定？** Claude Cookbook 的 Contextual Retrieval 指南默认 **语义 0.8 / BM25 0.2**，并说明可调。没有标准答案，用评测集扫。

### 2. 重排（Reranking）：宽召回、窄精排

```
bi-encoder（embedding）：query ─> 向量 ┐
                                       ├─ 点积   快，可预先算好文档向量，但 query 和文档"互相看不见"
                        doc   ─> 向量 ┘
cross-encoder（reranker）：[query + doc] ─> 模型 ─> 相关度分数
                                            慢，每对都要推理一次，但能逐词比对，排序更准
```

- 所以要**两阶段**：先用便宜的检索取一大批候选，再用贵的 reranker 只排这批。Anthropic 的做法：取 **top-150 → 重排 → 留 top-20** 给模型。
- LangChain 写法：`ContextualCompressionRetriever(base_compressor=CrossEncoderReranker(model=HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-v2-m3"), top_n=3), base_retriever=vector_retriever_k20)`——官方文档把这称为 "retrieve broadly, rerank narrowly"。也可以接 Cohere、Pinecone 等托管 rerank。
- 也可以用 **LLM 当重排器**（把候选编号列给 Claude，让它按相关度输出编号），Day 10 提到的 Claude Cookbook RAG 指南就这么做，MRR 从 0.74 提到 0.87。
- 代价：Cookbook 里 Cohere rerank 每次查询约 **+100~200 ms、约 $0.002**。重排条数是延迟和效果的权衡点。

### 3. Contextual Retrieval：在入库时修好"孤立的块"

Day 10 提到的问题："营收同比增长 3%"——哪家公司？哪个季度？Anthropic 的方案：入库时把**整篇文档 + 当前块**交给 Claude，让它写一段 50~100 token 的上下文说明，**拼到块前面**，再分别建向量索引（Contextual Embeddings）和 BM25 索引（Contextual BM25）。

提示词要点（原文意译）：整篇文档放在 `<document>` 里，块放在 `<chunk>` 里，要求"给出一段简短的上下文，说明这块在全文中的位置，用于提升检索效果，只输出上下文本身"。

**效果**（指标：1 − recall@20，前 20 条里没命中的比例）：

| 方案 | 失败率 | 降幅 |
|---|---|---|
| 基线 | 5.7% | — |
| + Contextual Embeddings | 3.7% | 35% |
| + Contextual BM25（混合） | 2.9% | 49% |
| + 重排 | 1.9% | **67%** |

**成本**：每块都要把全文发一遍，看似很贵——但整篇文档是相同前缀，用 **prompt caching** 缓存后，每百万文档 token 一次性成本约 **$1.02**。Cookbook 实测 61.83% 的输入 token 命中缓存。这是"prompt caching 不止用于对话"的经典例子（Day 13 详讲）。

Cookbook 的结论值得面试时说：**性价比最高的是 Contextual Embeddings**（一次性入库成本），混合检索和重排再各加 2~3 个点，但分别要多一套基础设施或按次付费。

### 4. 查询改写（Query Transformation）：在检索前修问题

用户问题往往口语化、太短、或一次问了好几件事，直接拿去检索效果差。几种常见做法：

| 技巧 | 做法 | 解决什么 |
|---|---|---|
| 改写 Rewrite | LLM 把口语问题改成检索友好的查询（补术语、去噪声） | "车里冒白雾咋整"→"车窗起雾 除雾 A/C" |
| 多查询 Multi-Query | 生成 N 个不同角度的问法，分别检索后取并集去重 | 单个查询的措辞偏差 |
| 分解 Decomposition | 复合问题拆成子问题分别检索 | "A 和 B 有什么区别"这类 |
| 后退 Step-back | 先问一个更抽象的问题，取回背景原理 | 需要原理才能答的具体问题 |
| HyDE | 先让 LLM 写一个"假想答案"，用**假答案的向量**去检索 | 问题短、文档长，两者在向量空间不对齐 |

- LangChain 的 `MultiQueryRetriever.from_llm(retriever, llm)`（`langchain_classic.retrievers.multi_query`）默认提示词要求生成 **3 个**改写版本，`include_original` 默认 `False`，结果做唯一并集。
- **HyDE** 三步：LLM 生成假想文档 → embed 假想文档 → 用它的向量检索（LangChain 有 HyDE retriever 集成）。直觉：**答案和答案长得像**，比"问题和答案"更容易在向量空间里靠近。风险：假答案本身是幻觉，可能把检索带偏到错误方向；每次查询多一次 LLM 调用。
- 所有改写都会**增加一次 LLM 延迟**，所以常用小模型（如 Haiku）来做，或只对"首轮检索置信度低"的问题触发。

### 5. GraphRAG：当问题需要"连线"或"全局视角"

> 本节主要参考 Microsoft GraphRAG 官方文档 ——「非 LangChain/Claude 来源」。

向量 RAG 的两类典型失败（GraphRAG 文档原话的意思）：
1. **连点成线**：答案分散在多个文档里，靠共同实体关联（"哪些车型用了和 X 同一家供应商的电池？"）；
2. **全局总结**：问整个语料库的主题（"这一年投诉最多的问题有哪些？"）——没有哪一块单独包含答案。

```
索引：文本单元 ─> LLM 抽取 实体 / 关系 / 断言 ─> 知识图谱
                                              │ Leiden 层次聚类
                                              ▼
                                   社区（community）─> 自底向上生成社区摘要
查询：Global Search  用社区摘要回答全局问题
      Local Search   从某个实体出发，扩展到邻居和相关概念
      DRIFT Search   Local + 社区上下文
      Basic Search   就是普通 top-k 向量检索
```

- 代价：索引阶段每个文本单元都要 LLM 抽取 + 生成社区摘要，**比向量 RAG 贵得多、慢得多**；官方也提示开箱效果可能不理想，强烈建议做 prompt tuning；语料变化可能要重建索引。
- 选型：大多数问答（"某个具体事实在哪"）用混合检索 + 重排就够了；只有"关系推理 / 全局总结"类问题占比高时才值得上 GraphRAG。LangChain 也有 Graph RAG 检索器集成，可以在向量检索结果上沿 metadata 关系做图遍历，是更轻量的折中。

### 6. 怎么决定加哪一个？——按诊断结果加

```
recall@k 低？ ──是──> 精确词漏召回？ ──是──> 加 BM25 混合检索
     │                    └─否──> 块缺上下文？ ──是──> Contextual Retrieval / 调切块
     │                                └─否──> 查询表述差？ ──> 查询改写 / 多查询 / HyDE
     否
     ▼
recall 够但 MRR 低（对的块排在后面）？ ──> 加重排
     ▼
都好但答错？ ──> 不是检索问题，查 prompt / 上下文顺序 / 模型（Day 10）
     ▼
问题本身需要跨文档关系或全局总结？ ──> 考虑 GraphRAG
```

## 代码示例

混合检索（BM25 + 向量，加权 RRF）+ Claude 查询改写 + Claude listwise 重排。用车机故障码场景演示"精确词靠 BM25"。

```python
# pip install -U langchain-classic langchain-community rank_bm25 langchain-voyageai langchain-anthropic
# export ANTHROPIC_API_KEY=...  VOYAGE_API_KEY=...
import re
from langchain_core.documents import Document
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_community.retrievers import BM25Retriever
from langchain_classic.retrievers import EnsembleRetriever
from langchain_voyageai import VoyageAIEmbeddings
from langchain_anthropic import ChatAnthropic

TEXTS = ["故障码 P0420：三元催化器效率低于阈值，建议尽快到店检测。",
         "故障码 P0171：系统过稀，常见原因是进气管漏气或喷油嘴堵塞。",
         "发动机故障灯常亮但车辆无异常抖动时，可继续低速行驶并预约保养。",
         "冬季车内起雾：打开前挡除雾并开启 A/C，可在 1 分钟内除雾。",
         "胎压报警：胎压低于 2.0 bar 时中控会弹窗提醒，请检查并补气。"]
DOCS = [Document(t, metadata={"id": i}) for i, t in enumerate(TEXTS)]

def zh_tokens(text: str) -> list[str]:  # 中文没有空格：英文/数字整词 + 汉字二元组
    words = [w.lower() for w in re.findall(r"[A-Za-z0-9.\-/]+", text)]
    han = re.findall(r"[一-鿿]", text)
    return words + [a + b for a, b in zip(han, han[1:])]

bm25 = BM25Retriever.from_documents(DOCS, preprocess_func=zh_tokens, k=3)
vec = InMemoryVectorStore.from_documents(DOCS, VoyageAIEmbeddings(model="voyage-4"))
dense = vec.as_retriever(search_kwargs={"k": 3})
hybrid = EnsembleRetriever(retrievers=[bm25, dense], weights=[0.5, 0.5], id_key="id")  # 加权 RRF

llm = ChatAnthropic(model="claude-haiku-4-5", max_tokens=200, temperature=0)

def rewrite(q: str) -> list[str]:  # 查询改写：口语 -> 原问题 + 2 条检索式
    out = llm.invoke("把用户问题改写成 2 条适合检索汽车手册的查询，可补充专业术语，"
                     f"每行一条，只输出查询。\n问题：{q}").content
    return [q] + [l.strip() for l in out.splitlines() if l.strip()][:2]

def rerank(q: str, docs: list[Document], top_n: int = 2) -> list[Document]:  # LLM listwise 重排
    listing = "\n".join(f"[{i}] {d.page_content}" for i, d in enumerate(docs))
    out = llm.invoke(f"问题：{q}\n候选：\n{listing}\n输出与问题最相关的 {top_n} 个编号，"
                     "按相关度从高到低，逗号分隔，只输出编号。").content
    idx = [int(x) for x in re.findall(r"\d+", out) if int(x) < len(docs)]
    return [docs[i] for i in dict.fromkeys(idx)][:top_n]

def retrieve(q: str) -> list[Document]:
    pool: dict[int, Document] = {}
    for sub in rewrite(q):                      # 多查询 → 每条都走混合检索 → 去重合并
        for d in hybrid.invoke(sub):
            pool.setdefault(d.metadata["id"], d)
    return rerank(q, list(pool.values()))       # 宽召回、窄精排

q = "仪表盘报 P0420 是啥意思，还能开吗？"
print("BM25 :", [d.metadata["id"] for d in bm25.invoke(q)])
print("向量 :", [d.metadata["id"] for d in dense.invoke(q)])
print("混合 :", [d.metadata["id"] for d in hybrid.invoke(q)])
print("最终 :", [d.page_content for d in retrieve(q)])
```

运行：安装依赖、设置两个 API Key 后 `python day11.py`。重点看前三行：带 `P0420` 的问题，BM25 能把第 0 条排在最前；对比向量检索和混合检索的排序，再把 `weights` 改成 `[0.2, 0.8]`（Cookbook 默认偏向语义）看变化。

## 与 Android / 车载场景的联系

车机语音问答里，故障码、零件号、App 包名、版本号都是"精确词"，必须有 BM25 这一路；同时用户说法很口语（"那个小黄灯一直亮"），需要查询改写把它映射到"发动机故障灯"。端侧算力有限时，BM25 索引非常轻量，适合放在本地做离线兜底，云端再做向量 + 重排。

## 高频面试题

**1. 为什么要做混合检索？BM25 和向量检索各擅长什么？**
- 向量：语义相近、同义改写（"冷"↔"调高温度"）；短板是精确字符串（ID、错误码、专有名词）。
- BM25：字面精确匹配，稀有词权重高；短板是不懂同义词。
- 两者互补；Anthropic 实验中加上 Contextual BM25，失败率降幅从 35% 提到 49%。

**2. 两路结果怎么合并？为什么不直接把分数相加？（追问：RRF 的 k 有什么作用？）**
- 分数量纲不同（BM25 无上界，余弦有界，不同库还可能是距离），相加没有意义。
- 用 RRF：`Σ w_i/(k + rank_i)`，只依赖排名；两路都靠前的文档胜出。
- k（LangChain 里叫 `c`，默认 60）越大，第 1 名和第 10 名的分差越小，结果更"平均"；越小越偏向头部排名。权重 `w_i` 控制哪一路更可信，用评测集调。

**3. 重排器和 embedding 模型有什么区别？为什么不直接用重排器检索全库？**
- embedding 是 bi-encoder：query 和文档分别编码，文档向量可离线预算，检索快。
- reranker 是 cross-encoder：query 和文档拼在一起过模型，能逐词交互，更准，但每对都要推理一次。
- 全库 N 条就要推理 N 次，不可行；所以两阶段：先检索 top-150，再重排留 top-20。

**4. 讲一下 Anthropic 的 Contextual Retrieval，它的成本怎么控制？**
- 入库时让 Claude 根据整篇文档给每块写 50~100 token 上下文，拼在块前，再建向量 + BM25 索引。
- 失败率：+上下文向量 -35%，+BM25 -49%，+重排 -67%。
- 成本：整篇文档是公共前缀，用 prompt caching，后续块读缓存打 1 折，约 $1.02/百万文档 token，一次性入库成本。

**5. 场景题：车机语音助手问"那个像发动机的黄灯一直亮着还能开吗"，检索总找不到对应条目。你怎么排查和改进？**
- 先诊断：看 recall@k——正确条目是否在候选里。
- 不在：问题口语化 → 加查询改写，把"像发动机的黄灯"改成"发动机故障灯 常亮"；手册用词和用户用词差异大 → 试 HyDE 或多查询；条目缺上下文 → Contextual Retrieval。
- 在但排得靠后：加重排。
- 安全相关：答案必须有来源支撑（Groundedness），不确定时提示到店检查，不能编造"可以继续开"。
- 延迟：车载对延迟敏感，改写和重排用小模型，或只在首轮检索得分低时触发。

**6. 追问：GraphRAG 什么时候值得用？代价是什么？**
- 值得：需要跨文档关系推理、或对整个语料做全局总结的问题（Microsoft GraphRAG 的 Global Search）。
- 代价：索引阶段大量 LLM 调用（抽实体关系 + 社区摘要），贵且慢；需要 prompt 调优；语料更新要重建。
- 大多数"查某个事实"的问答，混合检索 + 重排性价比更高。

## 常见误区

1. **"技巧全堆上去效果最好"**：每一步都加延迟和成本。HyDE 的假答案可能把检索带偏；重排条数太多拖慢响应。按 recall/MRR 诊断结果逐个加，并做消融对比。
2. **中文直接用默认 BM25**：默认按空格分词，中文整句变成一个 token，BM25 基本失效。要配分词（jieba 或 n-gram）。
3. **把 RRF 当成"分数平均"**：RRF 丢掉了原始分数，只用排名——这正是它稳健的原因，但也意味着"某一路分数特别高"的信息会丢失；需要保留分数信息时要用归一化后加权求和，并认真标定。

## 手写练习（禁用 AI，约 20 分钟）

> 交互版（关闭粘贴、逐行核对、自动计时）：在 Claude 里打开「手写练习 Day 11」页面。

**今日片段：加权 RRF 融合**（15 行）

```python
def weighted_rrf(rank_lists, weights, c=60):
    # 每一路检索结果都要有一个对应的权重，数量不一致就报错
    if len(rank_lists) != len(weights):
        raise ValueError("rank_lists 和 weights 数量必须相同")
    # 准备一个字典，记录每个文档的累计融合分数
    scores = {}
    # 同时遍历每一路结果和它的权重
    for ranked, w in zip(rank_lists, weights):
        # 排名从 1 开始数，第一名的 rank 是 1
        for rank, doc_id in enumerate(ranked, start=1):
            # 这一路贡献 w / (c + rank)，同一文档在多路出现就累加
            scores[doc_id] = scores.get(doc_id, 0.0) + w / (c + rank)
    # 按融合分数从高到低排序，返回文档 id 列表
    return sorted(scores, key=scores.get, reverse=True)

print(weighted_rrf([[2, 7, 6], [6, 2, 7]], [1, 1], c=1))  # 期望 [2, 6, 7]
```

**三步走**
1. 看着上面的注释和代码，先手敲一遍全部注释，再手敲一遍代码（不复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对，并跑一下最后的 `print`。

**容易写错的地方**
- `enumerate` 忘了 `start=1`：第一名 rank 变成 0，和 RRF 定义对不上。
- 用 `=` 而不是累加：同一文档在两路都出现时，后一路覆盖前一路。
- `sorted` 漏了 `reverse=True`：结果变成从低到高。

**进阶片段（可选）**：两阶段检索 `retrieve()`（改写 → 混合检索 → `setdefault` 去重 → 用原问题 `q` 重排），见上面代码示例。

**昨日复习（只做第③步）**：Day 10 的 `recall_at_k` 与 `mrr`。注意 MRR 找到第一个命中后要 `break`，recall 的分母是正确文档总数而不是 k。

## 动手练习（30 分钟）

在上面代码基础上：① 写 6 个问题（3 个含故障码/数字等精确词，3 个纯口语），标注每题应命中的 `id`；② 分别计算 **BM25-only、向量-only、混合（0.5/0.5）、混合（0.2/0.8）** 四种配置的 recall@2 和 MRR；③ 再打开 `rewrite()` 和 `rerank()`，看 MRR 提升多少、单次查询延迟增加多少（`time.perf_counter()`），写一句结论："在我的数据上，最值得加的是 ___"。

## 延伸学习

- Claude Academy · Building with the Claude API：[BM25 lexical search](https://academy.claude.com/courses/building-with-the-claude-api/bm25-lexical-search) → [A Multi-Index RAG pipeline](https://academy.claude.com/courses/building-with-the-claude-api/a-multi-index-rag-pipeline)（Day 10 留下的最后两课，手写 BM25 索引和 RRF）
- Claude Cookbook · [Enhancing RAG with contextual retrieval](https://platform.claude.com/cookbook/capabilities-contextual-embeddings-guide)：完整的 Contextual Embeddings + Elasticsearch BM25 + Cohere 重排代码和评测
- LangChain · [Cross encoder reranker](https://docs.langchain.com/oss/python/integrations/document_transformers/cross_encoder_reranker)

## 参考资料

- Anthropic Engineering · Contextual Retrieval：https://www.anthropic.com/engineering/contextual-retrieval
- Claude Cookbook · Enhancing RAG with contextual retrieval：https://platform.claude.com/cookbook/capabilities-contextual-embeddings-guide
- Claude Academy · BM25 lexical search：https://academy.claude.com/courses/building-with-the-claude-api/bm25-lexical-search
- Claude Academy · A Multi-Index RAG pipeline：https://academy.claude.com/courses/building-with-the-claude-api/a-multi-index-rag-pipeline
- LangChain Reference · EnsembleRetriever：https://reference.langchain.com/python/langchain-classic/retrievers/ensemble/EnsembleRetriever
- LangChain Reference · BM25Retriever：https://reference.langchain.com/python/langchain-community/retrievers/bm25/BM25Retriever
- LangChain Reference · MultiQueryRetriever：https://reference.langchain.com/python/langchain-classic/retrievers/multi_query/MultiQueryRetriever
- LangChain Docs · Cross encoder reranker：https://docs.langchain.com/oss/python/integrations/document_transformers/cross_encoder_reranker
- LangChain Docs · HyDE retriever：https://docs.langchain.com/oss/javascript/integrations/retrievers/hyde
- LangChain Docs · Retriever integrations：https://docs.langchain.com/oss/python/integrations/retrievers
- Microsoft GraphRAG（非 LangChain/Claude 来源）：https://microsoft.github.io/graphrag/
