# Day 10 · RAG 基础（第二轮：索引要跟着源更新、删除比新增难、过滤就是权限）（2026-11-07）

## 今日 AI 动态（2026-11-07）

### 趋势观察

本期最有意思的一条来自 Google：代码里过期的 TODO 注释会把编程 Agent「带偏」，每次带偏平均多花 70% 的步数、77% 的 token。Qoder CLI 同一天也在改进会话压缩时保留哪些工具结果。两件事讲的是同一个问题：Agent 读到的上下文有没有过期、该不该在那里。推测：「上下文卫生」会和模型能力一样，成为 Agent 效果的关键变量。另一条是 Claude Enterprise 支持按成本代码统计项目花费，企业开始把 AI 成本算到具体业务头上。

### 今日要点

- Google：开发者博客分析了数亿步 Agent 轨迹，发现 Agent 被过期 TODO 带偏时，步数、token、等待时间都显著上升。
- Anthropic：Claude Enterprise 支持给项目打成本代码标签（beta），按标签查看花费。
- 阿里巴巴 / Qoder：Qoder CLI 1.1.84 优化会话压缩，交接状态会保留最近的响应和工具结果。

### 分公司动态

#### 1. Anthropic / Claude

**[Claude 应用更新日志：Track spend by cost code with project tags (beta)](https://support.claude.com/en/articles/12138966-release-notes)**（2026-11-06）

符合条件的 Enterprise 组织可以给项目打上已有的计费代码（比如费用代码、客户事项编号），然后按这些标签查看大部分项目的花费。

**为什么重要**：企业落地 Agent 时最常被问的是「这笔钱花在哪个业务上」，按项目归集成本是做 ROI 核算的前提。

#### 2. Google

**[Agent Distraction](https://developers.googleblog.com/agent-distraction/)**（2026-11-06）

Google 工程师用代码里的 `TODO(bug/...)` 注释做观察指标，分析了数千名工程师的 Agent 轨迹。13% 的读文件会话碰到过这类注释，94% 的情况下 Agent 会忽略；但在采取了行动的情况里，83% 是用户没要求的，其中 41% 追的是早已关闭的 bug。控制变量后，每次这样的「绕路」带来多 70% 的步数、多 77% 的 token、多 105% 的等待时间。团队又做了一个清理 Agent，半年删掉了数千条过期引用，人工审核通过率超过 95%。

**为什么重要**：这是「过期上下文会伤害 Agent」的大规模实证，和今天讲的「知识库里留着旧版本」是同一类问题。

#### 3. 阿里巴巴 / Qoder

**[Qoder CLI 1.1.84](https://changelog.qoder.com/zh/cli)**（2026-11-06）

优化会话压缩：压缩后的交接状态会保留最近的响应和工具结果；同时修复了 `/mcp` 列表里待认证服务的状态显示错误。

**为什么重要**：压缩时保留什么直接决定长任务能不能接着做，各家编程 Agent 都在细调这一步。

### 动态来源

- [Claude 应用更新日志（Release notes）](https://support.claude.com/en/articles/12138966-release-notes)
- [Agent Distraction - Google Developers Blog](https://developers.googleblog.com/agent-distraction/)
- [Qoder CLI 更新日志](https://changelog.qoder.com/zh/cli)

---

> 第二轮说明：第一轮 Day 10 讲过 Load → Split → Embed → Store、四种切分策略、query/document 两种输入、Precision/Recall/MRR 和四类 LLM 评审。混合检索、重排归 Day 11，检索作为工具归 Day 12。今天只讲「上线之后」的基础问题：知识库变了，索引怎么跟着变。

## 读前三问

1. 用车手册 OTA 改了一处胎压值，重新入库后检索同时返回新旧两个值。原因是什么？怎么做到「只改变的那一块」？
2. 一款车型停售了，要把它的手册从知识库删掉。增量同步为什么删不掉？换成全量同步又有什么风险？
3. 按车型、按权限过滤，应该在检索前做还是检索后做？为什么？

## 从一个场景开始

你给车企做的「用车手册助手」上线了，A 车型手册写着「胎压 2.3 bar」。

一个月后 OTA 升级，手册改成 2.4 bar。运营同学把新手册重新跑了一遍入库脚本。第二天客服反馈：司机问胎压，助手回答「2.3 或 2.4 bar」。

你一查，向量库里新旧两块都在。脚本每次都是 `add_documents`，从来没删过旧的。更麻烦的是，你想写个脚本「先全删再全加」，又发现一次全量重建要几个小时，还要重算所有 embedding，花钱。

再后来，B 车型停售，产品说「把 B 的手册下架」。你又想到另一个问题：A 车主问胎压时，检索会不会把 B 车型的胎压也捞上来？

第一轮学的是「怎么建索引」，今天讲的是「索引怎么活下去」。

## 一句话总结

向量库是源文档的派生数据，必须跟源同步：靠内容哈希只重算变化的块并删掉旧块，下架的来源要用全量对账才删得掉；按车型和权限的过滤要在检索时做，评测标签要绑在稳定的来源位置上。

## 先跑起来

用 LangChain 的索引 API（`index` + `RecordManager`）做增量同步，再按车型过滤检索。为了不依赖 API Key，用一个「字符袋」玩具向量代替真实 embedding（共有的字越多越相似）。

```python
# rag_sync.py —— 手册更新后索引同步 + 按车型过滤检索（字符袋向量代替真实 embedding，无需 API Key）
import hashlib, math
from langchain_core.documents import Document
from langchain_core.embeddings import Embeddings
from langchain_core.indexing import InMemoryRecordManager, index
from langchain_core.vectorstores import InMemoryVectorStore

class CharBag(Embeddings):                      # 玩具 embedding：共有的字越多越相似
    def embed_query(self, text):
        v = [0.0] * 256
        for ch in text:
            v[int(hashlib.md5(ch.encode()).hexdigest(), 16) % 256] += 1
        n = math.sqrt(sum(x * x for x in v)) or 1
        return [x / n for x in v]
    def embed_documents(self, texts):
        return [self.embed_query(t) for t in texts]

def manual(model, tire):                        # 一本手册切成两块，metadata 记来源和车型
    meta = {"source": f"manual-{model}.md", "model": model}
    return [Document(f"{model} 车型胎压标准：前后轮 {tire} bar", metadata=meta),
            Document(f"{model} 车型雨刮更换：拉起雨刮臂按下卡扣", metadata=meta)]

store = InMemoryVectorStore(CharBag())
rm = InMemoryRecordManager(namespace="car-manuals")
rm.create_schema()
sync = lambda docs: index(docs, rm, store, cleanup="incremental", source_id_key="source", key_encoder="sha256")

print("首次入库:", sync(manual("A", 2.3) + manual("B", 2.5)))
print("原样重跑:", sync(manual("A", 2.3)))
print("A 车 OTA 后:", sync(manual("A", 2.4)))   # 只改了胎压那一块

naive = InMemoryVectorStore(CharBag())          # 对照组：每次 add_documents，不做同步
naive.add_documents(manual("A", 2.3)); naive.add_documents(manual("A", 2.4))
q = "A 车胎压是多少"
print("对照组命中:", [d.page_content for d in naive.similarity_search(q, k=2)])

only_a = lambda d: d.metadata["model"] == "A"   # InMemoryVectorStore 的 filter 是函数
for d, s in store.similarity_search_with_score(q, k=2, filter=only_a):
    print(f"同步+过滤: {s:.2f} {d.page_content}")
```

**运行**：`pip install -U langchain-core && python rag_sync.py`

**实测输出**（2026-11-07，Python 3.13，langchain-core 1.7.6）：

```
首次入库: {'num_added': 4, 'num_updated': 0, 'num_skipped': 0, 'num_deleted': 0}
原样重跑: {'num_added': 0, 'num_updated': 0, 'num_skipped': 2, 'num_deleted': 0}
A 车 OTA 后: {'num_added': 1, 'num_updated': 0, 'num_skipped': 1, 'num_deleted': 1}
对照组命中: ['A 车型胎压标准：前后轮 2.3 bar', 'A 车型胎压标准：前后轮 2.4 bar']
同步+过滤: 0.47 A 车型胎压标准：前后轮 2.4 bar
同步+过滤: 0.29 A 车型雨刮更换：拉起雨刮臂按下卡扣
```

**该看什么**：
- 「原样重跑」全是 skipped：内容没变就不重算 embedding，这在真实系统里省的是钱和时间。
- 「OTA 后」是加 1、跳过 1、删 1：只有胎压那一块变了，旧块被删掉。对照组则新旧两块同时命中，正是场景里的事故。
- 过滤后结果里只有 A 车型，B 的 2.5 bar 不会出现。
- 不传 `key_encoder` 会看到 SHA-1 的警告，生产里建议换成 sha256 等更强的哈希。

## 核心概念

### 1. 索引是派生数据，要靠「记账」来同步

- **是什么**：RecordManager 是一本账，记着每一块的内容哈希、来自哪个来源、什么时候写入的。再次入库时先查账：哈希一样就跳过，来源相同但哈希不同就写新块、删旧块。
- **没有它会怎样**：只能二选一，要么一直追加（新旧并存），要么每次全删全建（慢、贵、重建期间检索不可用）。
- **对应你熟悉的**：RecyclerView 的 DiffUtil，比较新旧列表，只刷新变化的那几项。

哈希算的是「内容 + metadata」，所以只改了车型标签也会被当作变化。`source_id_key` 指定哪个 metadata 字段代表「来源」，没有它，系统就不知道新块该替换谁。

### 2. 删除比新增难：增量 vs 全量

- **是什么**：`incremental` 只清理「这次出现过的来源」里的旧块；`full` 把这次没出现的来源全部当作已删除。
- **没有它会怎样**：我实测过，用 incremental 模式传入空列表，什么都不会删，所以停售车型的手册会一直留在库里。反过来，用 full 模式却只传了 A 的手册，B 的两块会被全部删掉。官方文档的警告是：full 模式下加载器必须返回完整数据集，否则会误删。
- **对应你熟悉的**：同步通讯录。增量同步只更新改过的联系人；全量对账要以服务端为准，本地多出来的就删，前提是服务端列表必须是完整的。

工程做法：日常用 incremental 处理更新；下架走单独的删除流程（按来源删）；定期用一次 full 对账，并在执行前检查「这次的来源数量和上次差太多就停下来」，防止加载器出错时把库清空。

### 3. 过滤就是权限和版本

- **是什么**：每一块带上 metadata（车型、年款、OTA 版本、可见范围），检索时按条件过滤，只在符合条件的块里找最相似的。
- **没有它会怎样**：A 车主拿到 B 车型的胎压；内部维修手册被普通用户检索到。如果先取 top-k 再过滤，k 个结果可能全被过滤掉，用户得到「没找到」。
- **对应你熟悉的**：Room 查询里的 `WHERE model = :model`，你不会先把整张表读到内存再筛。

LangChain 文档提醒：各家向量库对 metadata 过滤的支持程度不同，要看具体实现。`InMemoryVectorStore` 的 filter 是一个函数，pgvector、Pinecone 等用字典条件。过滤条件要由后端根据登录态生成，不能让模型自己填。

### 4. 评测标签绑在稳定的位置上

- **是什么**：评测集里每道题标注「应该命中哪些块」。Claude cookbook 的 RAG 评测集用文档 URL 加章节锚点（如 `...#creating-test-cases`）作为块的标识。
- **没有它会怎样**：如果标注写的是「第 37 块」，改了切块参数或插了一段新内容，序号全变，评测集一夜作废，你也就不敢再调切分参数。
- **对应你熟悉的**：UI 自动化测试用 `resource-id` 定位控件，而不是用屏幕坐标。

同理，换 embedding 模型时不要原地覆盖：新建一个索引，跑完评测再切流量，旧索引留着回滚。不同模型的向量不能混在一个库里比较。

## 高频面试题

**Q1：知识库文档更新了，向量库怎么同步？**
- 每块记内容哈希和来源 ID，入库前对账。
- 哈希相同跳过，来源相同、哈希不同则写新删旧。
- 只有变化的块需要重算 embedding，省成本。
- LangChain 的实现是 `index()` 加 `RecordManager`。

**Q2（追问）：那文档整个删除了怎么办？增量模式能处理吗？**
- 增量模式只清理本次出现过的来源，删不掉整篇消失的文档。
- 要么走显式的按来源删除，要么定期跑全量对账。
- 全量对账要求输入是完整数据集，否则会误删。
- 加保护：来源数量骤降时中止，并保留快照可回滚。

**Q3（场景题）：一个助手要服务十几款车型、多个 OTA 版本，还有只给 4S 店看的维修手册，检索怎么设计？**
- metadata 带车型、年款、OTA 版本、可见范围。
- 过滤条件由后端按车辆 VIN 和用户角色生成，在检索时生效。
- 用户手册和维修手册分开命名空间或索引，权限最敏感的物理隔离。
- 评测集按车型分组，确认不会跨车型串答。

**Q4（追问）：为什么不能先检索 top-k 再过滤？**
- 过滤后可能剩不到 k 条，甚至为空，召回下降。
- 被过滤掉的敏感块已经被读取过，日志或缓存里可能泄露。
- 过滤放在检索里，相当于在合格集合里找最相似的。
- 如果向量库不支持某种过滤，宁可按维度拆索引。

## 常见误区

- **「重跑一遍入库脚本就是更新」**：只追加不删除，新旧版本会同时被检索到，模型给出自相矛盾的答案。
- **「metadata 只是用来显示出处的」**：它同时是权限、版本和同步的依据。少了来源 ID，增量同步都做不了。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带来源 metadata 的切块、增量同步、按车型过滤检索**（取自 `rag_sync.py`）

```python
# 生成一本手册的两块，每块都带上来源和车型
def manual(model, tire):
    # 来源用于同步对账，车型用于检索过滤
    meta = {"source": f"manual-{model}.md", "model": model}
    # 返回两个 Document：正文放 page_content，标签放 metadata
    return [Document(f"{model} 车型胎压标准：前后轮 {tire} bar", metadata=meta),
            Document(f"{model} 车型雨刮更换：拉起雨刮臂按下卡扣", metadata=meta)]

# 建一个内存向量库，指定用哪个 embedding
store = InMemoryVectorStore(CharBag())
# 建一本记账本，命名空间区分不同知识库
rm = InMemoryRecordManager(namespace="car-manuals")
# 第一次用之前先建表
rm.create_schema()
# 同步函数：增量清理，按 source 字段认来源，用 sha256 算哈希
sync = lambda docs: index(docs, rm, store, cleanup="incremental", source_id_key="source", key_encoder="sha256")

# 首次入库两款车的手册
print(sync(manual("A", 2.3) + manual("B", 2.5)))
# A 车 OTA 后重新同步，只会替换变化的那一块
print(sync(manual("A", 2.4)))

# 过滤函数：只保留 A 车型的块
only_a = lambda d: d.metadata["model"] == "A"
# 在过滤后的范围里取最相似的两块，并带上分数
for d, s in store.similarity_search_with_score("A 车胎压是多少", k=2, filter=only_a):
    # 打印分数和正文
    print(f"{s:.2f} {d.page_content}")
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `index()` 的前三个参数顺序是「文档、记账本、向量库」；`cleanup`、`source_id_key` 是关键字参数，不传 `source_id_key` 时 incremental 模式会报错。
- `RecordManager` 用之前要先 `create_schema()`。
- `similarity_search_with_score` 返回的是 `(Document, 分数)` 元组，要拆成两个变量；`InMemoryVectorStore` 的 `filter` 接收的是函数，换成 pgvector 等要改成字典条件。

**昨日复习（只做第③步）**：第二轮 Day 9 的「带作用域检查、按司机隔离、同主题覆盖的记忆工具」（`memory_scope.py`）。只看你自己写的注释重写，重点核对：命名空间是元组 `("drivers", id, "prefs")`；`old.value['value']` 取旧值；创建时传 `store=` 和 `context_schema=`，调用时传 `context=`。

## 动手练习（30 分钟）

在 `rag_sync.py` 的基础上：

1. B 车型停售：先用 `cleanup="incremental"` 传入只含 A 的列表，观察 B 是否还在；再改用 `cleanup="full"`，看 `num_deleted`。
2. 给 `full` 模式加一道保护：如果这次的来源数量比上次少一半以上，就打印警告并不执行。
3. 给每块加 `audience` 字段（`owner` / `dealer`），把维修内容标成 `dealer`，再写一个过滤函数，让车主身份检索不到维修内容。

## 读后自测

1. 用车手册 OTA 改了一处胎压值，重新入库后检索同时返回新旧两个值。原因是什么？怎么做到「只改变的那一块」？——见「核心概念」第 1 节和「先跑起来」
2. 一款车型停售了，增量同步为什么删不掉？换成全量同步又有什么风险？——见「核心概念」第 2 节
3. 按车型、按权限过滤，应该在检索前做还是检索后做？为什么？——见「核心概念」第 3 节和 Q4

## AI 历史知识点：Faiss 与十亿级向量检索（2017，Facebook AI Research）

**是什么**：论文《Billion-scale similarity search with GPUs》提出在 GPU 上做大规模近邻搜索的方法，并开源了代码库 Faiss。核心技术有两个：一个单次扫描、状态放在寄存器里的 GPU top-k 选择算法（WarpSelect）；以及基于乘积量化（把向量压缩成很短的编码）和倒排索引（先粗分桶、只在少数桶里找）的近似搜索。

**何时由谁提出**：Facebook AI Research 的 Jeff Johnson（纽约）、Matthijs Douze 和 Hervé Jégou（巴黎），2017 年 2 月提交到 arXiv。

**为什么重要**：在 SIFT1B 十亿向量数据集上，比此前最好的 GPU 方法快 8.5 倍；用 4 块 GPU 不到 12 小时就能为 10 亿向量建出近邻图。它让「十亿级向量、毫秒级查询」从研究变成了一个可以 `pip install` 的工具，后来的很多向量数据库都直接用它或借鉴了它的索引结构。

**和今天的趋势有什么联系**：RAG 里的「向量库」本质就是近似近邻搜索，近似意味着用一点召回换速度，这也是为什么检索要单独评测。但 Faiss 只管「找得快」，不管「数据是不是最新的」。今天讲的增量同步、删除、过滤，正是向量检索从算法走向生产系统后要补上的一层。

**原始来源**：
- [Billion-scale similarity search with GPUs - arXiv:1702.08734](https://arxiv.org/pdf/1702.08734)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `index()` 签名 | `index(docs_source, record_manager, vector_store, *, batch_size=100, cleanup=None, source_id_key=None, cleanup_batch_size=1000, force_update=False, key_encoder="sha1", upsert_kwargs=None)`，返回 `num_added / num_updated / num_skipped / num_deleted` |
| cleanup 模式 | `None`：不删；`incremental`：边写边清理本批出现过的来源，同一来源跨多批会重复工作，batch 尽量大；`full`：没出现的来源全删，加载器必须返回全集；`scoped_full`（langchain-core 0.3.25 起）：在内存里记录来源 ID，适合加载器不能一次给全集的情况，千万级以上需并行 |
| `key_encoder` | `sha1`（默认，会警告不抗碰撞）/ `sha256` / `sha512` / `blake2b`，或自定义函数 |
| 本次实测 | incremental 传空列表：删 0；full 只传 A：跳过 2、删 2（B 被删） |
| 向量库接口 | `add_documents(docs, ids=...)`、`delete(ids=...)`、`similarity_search(query, k, filter=...)`；`InMemoryVectorStore` 支持按 ID 删除、metadata 过滤、带分数搜索、异步，不支持多租户 |
| Voyage embedding | `voyage-4-large` / `voyage-4` / `voyage-4-lite` / `voyage-code-4` / `voyage-4-nano`（开源权重）；默认 1024 维，可选 256/512/2048；已归一化，点积等于余弦；文档用 `input_type="document"`，问题用 `"query"` |
| cookbook 评测集 | 100 题，字段 `id / question / correct_chunks（URL+锚点列表）/ correct_answer`；基线 Precision 0.43、Recall 0.66、MRR 0.74、端到端 71%；加摘要索引和重排后 MRR 0.87、端到端 81% |
| Deep Agents RAG 教程 | `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)`；每块写 `# Source:` 头；生产建议持久化向量库并在文档变化时定期刷新 |

### 来源说明

- 演示代码的 embedding 是玩具实现，只用来验证同步、删除和过滤逻辑，分数没有语义意义。
- 今日 AI 动态：OpenAI 官网本次无法打开，未核实，没有收录；Meta、DeepSeek、智谱、Qwen 在时间范围内没有找到官方新发布。
- Faiss 来自 arXiv 原始论文（非 LangChain/Claude 来源）；「先 top-k 再过滤会漏召回」是根据检索原理的推导，不是某篇官方文档的原话。

### 延伸学习

- Claude Academy《Building with the Claude API》的「The full RAG flow」一课：用两维玩具向量讲清预处理和查询两个阶段，以及余弦相似度和余弦距离的区别。
- LangChain Deep Agents 的 RAG 教程：把检索结果写成带来源头的文件，再交给子 Agent 分析。

### 参考资料

- [index() - LangChain Reference](https://reference.langchain.com/python/langchain-core/indexing/api/index)
- [Vector store integrations - Docs by LangChain](https://docs.langchain.com/oss/python/integrations/vectorstores)
- [Retrieval Augmented Generation (RAG) with Deep Agents - Docs by LangChain](https://docs.langchain.com/oss/python/deepagents/rag)
- [Embeddings - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/embeddings)
- [Retrieval augmented generation - Claude Cookbook](https://platform.claude.com/cookbook/capabilities-retrieval-augmented-generation-guide)
- [The full RAG flow - Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/the-full-rag-flow)
