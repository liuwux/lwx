# Day 17 · LangGraph（第二轮：并行写要挂 reducer、重跑要幂等、回溯不撤销副作用）（2026-11-14）

## 今日 AI 动态（2026-11-14）

### 趋势观察

- Anthropic 和 Google 同一天都在讲「Agent 的外围架构」：一个讲常驻 Agent 的事件收件箱、定时器、持久记忆，一个讲多 Agent 和人类专家协作攻克数学与系统难题。重点已经从「模型会不会」转向「Agent 怎么长期、可靠地跑」。
- DeepSeek 把联网搜索、读网页、文件解析、内容安全做成同一个 API 的开关。推测：国内模型厂商正在从「卖模型」转向「卖一整套应用底座」，开发者自己搭的 RAG/搜索管线会越来越薄。

### 今日要点

- Anthropic 发布「常驻 Agent」构建指南：五个基础组件，事件要去重、一个收件箱同时只跑一个 worker。
- Google Research 公布 Antigravity /teamwork 多 Agent 系统的科研成果，并向 1 万名学术研究者开放半年免费使用。
- DeepSeek 为 `deepseek-flash` 上线全套应用 API：联网搜索、URL 读取、文件输入、安全审查，另有独立的搜索、OCR、ASR、TTS 接口。

### 分公司动态

#### 1. Anthropic / Claude

**[Building always-on agents](https://claude.com/resources/articles/building-always-on-agents)**（2026-11-13）
这是一篇官方最佳实践，讲怎么做「有状态、会主动干活」的常驻 Agent。核心循环和普通 Agent 一样，区别在外围：Agent 大部分时间空闲，被事件唤醒后跑一小段（run），处理完就存档。文中列了五个组件：事件收件箱（去重、同一收件箱只允许一个 worker）、定时器、文件式持久记忆、权限（凭据代理、高风险操作暂停等用户批准）、发件箱。
**为什么重要**：这几条和今天讲的 LangGraph 持久执行是同一件事：每一步要存档，重跑要去重，危险操作前要停下来等人。

#### 2. Google

**[Advancing science & engineering with Antigravity's /teamwork](https://research.google/blog/advancing-science-engineering-with-antigravitys-teamwork-feature/)**（2026-11-13）
Google Research 公布了 /teamwork 多 Agent 系统和人类专家合作的成果：完整证明并用 Lean 4 形式化了 2013 年提出的 Courtade–Kumar 猜想；30 个 Agent 组成的循环从核磁数据推断分子结构，首选答案完全正确率 95%；用 Rust 重写 AV1 解码器 libgav1，文中称比 C++ 原版快 20–30%。带 /teamwork、/boost、Deep Review 的 Gemini 4 模型已可用，另开放 1 万个学术研究者名额，免费使用半年。
**为什么重要**：多 Agent 协作开始在可验证的硬问题（形式化证明、性能可测的系统代码）上拿出成果，不再只是 demo。

#### 3. DeepSeek

**[Change Log：Releasing the full suite of application APIs](https://api-docs.deepseek.com/updates/)**（2026-11-13）
`deepseek-flash` 在 Messages、Responses、Chat Completions 三种接口里都能用新能力：打开 `deepseek_web_search` 工具就能联网搜索并带引用回答，配合思考模式可以多轮搜索；`deepseek_read_url` 自动抓取用户消息里的链接；支持 PDF、Office、图片等文件输入；新增安全审查参数，命中风险时返回拒答标记。另外上线了独立的搜索、OCR、ASR、TTS 接口，并开源了一个演示聊天客户端。
**为什么重要**：做车机或 App 助手时，联网搜索、语音识别和合成可以直接用同一家 API，不用自己拼一套管线。

### 动态来源

- https://claude.com/resources/articles/building-always-on-agents
- https://research.google/blog/advancing-science-engineering-with-antigravitys-teamwork-feature/
- https://api-docs.deepseek.com/updates/

---

> 第二轮说明：第一轮 Day 17 讲过 State、节点与边、checkpoint、interrupt 恢复时节点从头重跑。今天只讲进阶：一次并行执行的规则（super-step）、失败时谁来处理、「至少执行一次」带来的重复副作用，以及时间回溯的回放和分叉。

## 读前三问

1. 三个并行节点同时写同一个字段，LangGraph 会怎样？其中一个失败了，另外两个的结果还算不算数？
2. 为什么说 LangGraph 的节点是「至少执行一次」？预约、扣费这类节点该怎么写？
3. 用时间回溯改掉历史里的一个选择再跑一遍，之前已经发生的外部操作会被撤销吗？

## 从一个场景开始

车机助手帮车主规划长途：沿途三个服务区，同时询价，挑最便宜的预约充电位。

第一轮学的那些已经够把图画出来。真上线后，问题出在「出错」和「重来」上：

- 三个询价并行跑，结果都往 `quotes` 里写。上线第一天就报错 `InvalidUpdateError`。
- 服务区 B 的接口偶尔超时。你希望只重试 B，A 和 C 不要重新请求。
- 进程在「预约」这一步跑到一半被杀，恢复后又跑了一次，车主收到两张订单。
- 车主到了半路说「改去 C 吧」。你用时间回溯改了选择重跑，C 订上了，可 B 的订单还在。

这四个坑都来自同一件事：LangGraph 按「一步一步、每步存档」执行。只要摸清每一步的规则，这四个坑都能提前避开。

## 一句话总结

LangGraph 把执行切成一个个并行步，每步结束统一合并、存档。所以并行写要定合并规则，失败只重跑没完成的节点，但节点可能被执行多次，外部操作必须幂等；回溯只改存档，不会替你撤销外部操作。

## 先跑起来

不调模型、不需要 API Key。B 的接口前两次故意超时，`calls` 记录每个服务区被请求了几次，`booked` 模拟外部订单系统。

```python
# trip_graph.py —— 长途行程：并行询价(Send) + 失败重试(RetryPolicy) + 幂等预约 + 时间回溯分叉（langgraph 1.2.21）
import operator
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Send, RetryPolicy

class Trip(TypedDict):
    trip_id: str
    stations: list[str]
    quotes: Annotated[list[tuple], operator.add]   # 并行写入的字段必须挂 reducer
    choice: str

PRICE, calls, booked = {"服务区A": 1.6, "服务区B": 1.2, "服务区C": 1.4}, {}, {}

def fan_out(s: Trip):                       # 运行时决定开几个并行分支
    return [Send("quote", {"station": st}) for st in s["stations"]]

def quote(s: dict):
    st = s["station"]; calls[st] = calls.get(st, 0) + 1
    if st == "服务区B" and calls[st] < 3: raise ConnectionError(f"{st} 超时")  # B 前两次超时
    return {"quotes": [(st, PRICE[st])]}

def pick(s: Trip):
    return {"choice": min(s["quotes"], key=lambda q: q[1])[0]}

def book(s: Trip):                          # 幂等键来自业务 ID，重跑不会重复下单
    key = f"{s['trip_id']}:{s['choice']}"
    booked.setdefault(key, len(booked) + 1)
    print(f"   book 执行，幂等键={key}，订单号={booked[key]}"); return {}

g = StateGraph(Trip)
g.add_node("quote", quote, retry_policy=RetryPolicy(max_attempts=3, initial_interval=0.1))
g.add_node("pick", pick); g.add_node("book", book)
g.add_conditional_edges(START, fan_out, ["quote"])
g.add_edge("quote", "pick"); g.add_edge("pick", "book"); g.add_edge("book", END)
app = g.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "trip-001"}}
r = app.invoke({"trip_id": "T1", "stations": list(PRICE), "quotes": []}, cfg, durability="sync")
print("1) 报价:", sorted(r["quotes"]), "选中:", r["choice"], "各站调用次数:", calls)
before_book = next(h for h in app.get_state_history(cfg) if h.next == ("book",))
app.invoke(None, before_book.config); print("2) 回放 book 后订单数:", len(booked))
fork = app.update_state(before_book.config, {"choice": "服务区C"})   # 车主改选 C
print("3) 分叉后:", app.invoke(None, fork)["choice"], "订单:", booked, "调用次数不变:", calls)
```

**运行**：`pip install langgraph`（实测 1.2.21），然后 `python trip_graph.py`。

**实测输出**（2026-11-14）：

```
   book 执行，幂等键=T1:服务区B，订单号=1
1) 报价: [('服务区A', 1.6), ('服务区B', 1.2), ('服务区C', 1.4)] 选中: 服务区B 各站调用次数: {'服务区A': 1, '服务区B': 3, '服务区C': 1}
   book 执行，幂等键=T1:服务区B，订单号=1
2) 回放 book 后订单数: 1
   book 执行，幂等键=T1:服务区C，订单号=2
3) 分叉后: 服务区C 订单: {'T1:服务区B': 1, 'T1:服务区C': 2} 调用次数不变: {'服务区A': 1, '服务区B': 3, '服务区C': 1}
```

**该看什么**：
- 第 1 行：B 被请求了 3 次，A、C 只有 1 次。重试只针对失败的分支。
- 第 2 行：回放时 `book` 确实**又执行了一次**，但幂等键相同，订单数仍是 1。
- 第 3 行：分叉后订上了 C，**B 的订单还在**。时间回溯只改存档，外部世界要你自己去撤销。

## 核心概念

### 1. super-step：一次并行步的三条规则

- **是什么**：LangGraph 把执行切成一步一步。同一步里的节点并行跑，各自的输出先不生效，等全部跑完才统一合并进状态，然后存档，再进入下一步。
- **没有它会怎样**：并行节点互相读到对方的半成品，结果取决于谁跑得快。
- **对应你熟悉的**：Compose 的一帧。一帧内多处修改 `State`，到帧末统一提交、统一重组，不会读到一半的值。

由此有三条规则：
1. **多个节点同步写同一个字段，必须挂 reducer**，否则报 `InvalidUpdateError`（去掉上例的 `operator.add` 即可复现）。并行结果的先后顺序也不保证，要排序就自己带排序键。
2. **一步是一个事务**：任何一个分支抛异常，这一步的更新都不会合并进状态。
3. **已成功的分支会单独存下来**（pending writes）：恢复时只重跑失败的那个。

`Send` 用来做「数量运行时才知道」的扇出：每个 `Send` 带一份独立输入，像上例每个服务区一份。

### 2. 失败分层：按「谁能修」选处理方式

- **是什么**：官方按「谁能修这个错」分五类：系统能修的（瞬时错误）→ `RetryPolicy`；重试耗尽后要补偿的 → 节点的 `error_handler`；模型能修的 → 错误写进状态、路由回去重来；人能修的 → `interrupt`；意外错误 → 直接抛出。
- **没有它会怎样**：所有异常都 `try/except` 后让模型重试，参数错误也重试三遍，白白烧 token。
- **对应你熟悉的**：OkHttp 的 `retryOnConnectionFailure` 只管网络层；业务错误交给上层弹窗或回退，两层不混用。

`RetryPolicy` 默认不重试 `ValueError`、`TypeError` 这类代码错误；对 HTTP 错误只重试 5xx。

### 3. 至少执行一次：副作用要幂等

- **是什么**：重试、崩溃恢复、`interrupt` 恢复、回放，都会把**整个节点**从头再跑一遍。存档只在每步结束时写，节点里已经发出去的请求，存档不知道。
- **没有它会怎样**：预约节点调完接口、还没来得及存档时进程被杀，恢复后再调一次，车主收到两张订单。
- **对应你熟悉的**：WorkManager 的 Worker 也可能被重复执行，所以上传接口要带请求 ID 让服务端去重。

写法：用**业务 ID 拼出稳定的幂等键**（`trip_id + 选择`），交给下游去重。键不能在节点里随机生成，否则重跑时是新键。要确保「下单前一定已存档」，就用 `durability="sync"`：每步写完存档才进入下一步。Anthropic 的常驻 Agent 指南也要求事件收件箱去重，道理相同。

### 4. 时间回溯：回放和分叉

- **是什么**：回放是从某个历史存档按原路再跑一遍：`invoke(None, 历史快照.config)`。分叉是先用 `update_state` 在旧存档上改值，生成一个新分支再跑。两者都不会重跑存档之前的节点。
- **没有它会怎样**：车主中途改主意，只能从头询价，三个接口全部重新请求。
- **对应你熟悉的**：Git。回放像在旧提交上重新执行 CI；分叉像 `checkout` 旧提交后开新分支提交，原来的历史还在。

两个要点：
1. `update_state` **不会回滚**线程，只是在旧存档上新增一个分支。
2. 存档之后的节点会**真的再执行**（模型调用、外部接口、`interrupt` 都会再来一遍），外部已发生的事不会被撤销。上例里 B 的订单就得自己写取消逻辑。

## 高频面试题

**1. LangGraph 并行节点同时写同一个字段会发生什么？怎么解决？**
- 同一步内无 reducer 的字段收到多个值，报 `InvalidUpdateError`。
- 给字段挂 reducer（如 `operator.add`），或让每个分支写不同的字段。
- 并行结果顺序不保证，需要稳定顺序就带排序键再排序。

**2.（追问）三个并行询价里一个失败了，恢复时另外两个会重跑吗？**
- 不会。同一步内成功的节点输出会单独存下来（pending writes）。
- 这一步整体不合并进状态（事务性），恢复后只重跑失败的分支。
- 若配了 `RetryPolicy`，只有失败分支按退避重试。

**3. 节点里调用了扣费接口，怎么保证崩溃恢复后不重复扣费？**
- 认清节点是「至少执行一次」：重试、恢复、回放都会重跑整个节点。
- 用业务 ID 生成稳定幂等键，交给下游去重，并确认下游真的支持幂等键。
- 下单前用 `durability="sync"` 保证前面的步骤已存档；没收到确认时先查询状态再决定补偿。
- 无法幂等的操作放进 `error_handler` 或单独的去重 worker。

**4.（场景题）车主半路要求「回到选服务区那一步，改成 C」，你怎么实现？要注意什么？**
- 用 `get_state_history` 找到 `next == ("book",)` 的快照，`update_state` 改 `choice` 后分叉重跑。
- 询价节点在快照之前，不会重新请求。
- B 的预约不会自动消失：分叉前或在分叉分支里先取消 B，再订 C。
- 若流程里有确认弹窗，分叉后 `interrupt` 会重新触发，需要车主再确认一次。

## 常见误区

1. **「时间回溯 = 撤销」**：它只在存档里开新分支，原历史和外部世界都不变。已下的单、已发的消息，要靠补偿逻辑处理。
2. **「挂了 checkpointer 就不会重复执行」**：存档只在每步结束时写。节点内部发出的请求、写的文件，恢复时会再来一遍，所以必须幂等。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：并行扇出 + 重试 + 幂等预约 + 分叉**（取自 `trip_graph.py`，略有精简）

```python
# 状态：报价字段挂上「相加」规则，允许并行分支同时写
class Trip(TypedDict):
    # 行程 ID，用来拼幂等键
    trip_id: str
    # 沿途服务区列表
    stations: list[str]
    # 各分支的报价，按列表相加合并
    quotes: Annotated[list[tuple], operator.add]
    # 选中的服务区，后写覆盖
    choice: str

# 扇出函数：每个服务区派一个独立的询价任务
def fan_out(s: Trip):
    # 每个 Send 指定目标节点和这份任务自己的输入
    return [Send("quote", {"station": st}) for st in s["stations"]]

# 预约节点：可能被执行多次，所以要幂等
def book(s: Trip):
    # 用业务 ID 拼稳定的幂等键，重跑时键不变
    key = f"{s['trip_id']}:{s['choice']}"
    # 同一个键只下一次单
    booked.setdefault(key, len(booked) + 1)
    # 不更新状态
    return {}

# 询价节点挂重试策略：最多试 3 次
g.add_node("quote", quote, retry_policy=RetryPolicy(max_attempts=3))
# 从 START 出发，由扇出函数决定开几个分支
g.add_conditional_edges(START, fan_out, ["quote"])

# 在历史里找到「下一步是预约」的那个存档
before_book = next(h for h in app.get_state_history(cfg) if h.next == ("book",))
# 在这个存档上改选择，得到新分支的配置
fork = app.update_state(before_book.config, {"choice": "服务区C"})
# 输入传 None，从新分支继续跑
app.invoke(None, fork)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `h.next == ("book",)`：`next` 是元组，单元素元组要带逗号，写成 `("book")` 就成了字符串，永远匹配不上。
- 回放和分叉的输入都是 `None`；传了字典就成了「提交新输入」，会从入口重新开始。
- `Send` 的第二个参数是这个分支自己的输入字典，不是整个状态。

**昨日复习（只做第③步）**：第二轮 Day 16 的「带失败隔离的派活工具」（`car_trip.py` 里的 `task`）。只看你自己写的注释重写，重点核对：`Annotated[dict, operator.or_]`、`ToolMessage` 带 `tool_call_id`、失败分支 `update` 为 `{}`。

## 动手练习（30 分钟）

在 `trip_graph.py` 的基础上改：

1. 把 `quotes` 的 `Annotated[..., operator.add]` 去掉，记录报错信息；再改回来。
2. 把 B 改成前 5 次都超时，观察 `RetryPolicy` 用完后抛出的异常。然后改回 3 次以内成功，用 `app.invoke(None, cfg)` 从失败处恢复，确认 A、C 的调用次数仍是 1。
3. 写一个 `cancel` 逻辑：分叉改选 C 之前，先把 `booked` 里 `T1:服务区B` 删掉。用一句话写下：这段取消逻辑应该放在分叉流程的哪里、为什么。

## 读后自测

1. 三个并行节点同时写同一个字段，LangGraph 会怎样？其中一个失败了，另外两个的结果还算不算数？→ 见第 7 节「核心概念 1」
2. 为什么说 LangGraph 的节点是「至少执行一次」？预约、扣费这类节点该怎么写？→ 见第 7 节「核心概念 3」
3. 用时间回溯改掉历史里的一个选择再跑一遍，之前已经发生的外部操作会被撤销吗？→ 见第 7 节「核心概念 4」

## AI 历史知识点：Pregel 大规模图计算（2010，Google）

**是什么**：Google 为处理网页图、社交网络这类数十亿顶点的大图而做的计算系统。程序只需从「单个顶点」的视角写：读上一轮收到的消息、更新自己、给邻居发消息。计算按一轮轮的 superstep 推进，每轮之间有全局同步栅栏；所有顶点都「投票停止」且没有在途消息时结束。

**何时由谁提出**：Grzegorz Malewicz、Matthew Austern 等 7 位 Google 工程师，发表于 SIGMOD 2010。同步轮次的结构来自 Leslie Valiant 提出的 BSP（整体同步并行）模型。

**为什么重要**：在它之前，每种图算法要么自建一套分布式设施，要么用 MapReduce 在每个阶段搬运整张图。Pregel 只在机器间传消息，并在每个 superstep 开始时做 checkpoint，机器故障后从最近存档重算丢失的轮次。PageRank 这类算法因此能稳定跑在超大规模图上。

**和今天的联系**：LangGraph 的底层运行时就叫 Pregel，官方文档明确说它按 Pregel / BSP 模型分步执行：计划、并行执行、统一更新。今天讲的「一步内互不可见、步末统一合并、按步存档、故障后从存档恢复」，正是 16 年前这篇论文的思路搬到了 Agent 上。

**原始来源**：[Pregel: a system for large-scale graph processing（Google Research）](https://research.google/pubs/pub37252/) · [论文 PDF](https://www.cs.princeton.edu/courses/archive/spring16/cos598F/pregel_paper.pdf) · [LangGraph runtime 文档](https://docs.langchain.com/oss/python/langgraph/pregel)

---

## 参考区（用时查，不必通读）

### 细节速查

- **durability 三档**（在 `invoke` / `stream` 上传）：`"exit"` 只在结束、出错或中断时存档，最快但中途崩溃无记录；`"async"` 下一步运行时后台存档，有小概率丢档；`"sync"` 每步存完再继续，最可靠。
- **恢复方式**：`invoke(None, cfg)` 从最新存档继续未完成的工作；`invoke(新输入, cfg)` 按 reducer 合并后从入口开始；`invoke(Command(resume=x), cfg)` 回答中断；`invoke(None, 快照.config)` 回放。
- **`RetryPolicy` 参数与默认值**：`max_attempts=3`（含首次）、`initial_interval=0.5`、`backoff_factor=2.0`、`max_interval=128.0`、`jitter=True`、`retry_on=default_retry_on`。默认不重试 `ValueError`、`TypeError`、`ArithmeticError`、`ImportError`、`LookupError`、`NameError`、`SyntaxError`、`RuntimeError`、`ReferenceError`、`StopIteration`、`StopAsyncIteration`、`OSError` 及其子类；`requests`/`httpx` 只重试 5xx；`NodeTimeoutError` 默认可重试。自定义：`retry_on=函数`，内部可调用 `default_retry_on(exc)`。
- **`error_handler`**：`add_node(..., error_handler=handler)`，签名 `handler(state, error: NodeError) -> Command`；`NodeError` 有 `node`、`error` 两个字段；重试耗尽后才执行；`interrupt()` 不经过重试和 handler。
- **超时**（langgraph>=1.2，只支持 async 节点）：`timeout=60` 或 `TimeoutPolicy(run_timeout=..., idle_timeout=...)`，超时抛 `NodeTimeoutError` 并交给重试策略。
- **`update_state(config, values, as_node=None)`**：返回新分支的 config；多个节点同一步写过状态、线程无历史、想跳过某节点时显式传 `as_node`。
- **时间回溯与中断**：回放、分叉都会重新触发 `interrupt`，需要新的 `Command(resume=...)`。
- **扇入**：`add_edge(["b", "c"], "d")` 等列出的节点都完成才跑 d；`add_node(d, defer=True)` 让 d 等其他待执行任务都结束。
- **recursion_limit**：限制 super-step 数，超了抛 `GraphRecursionError`；写在 config 顶层。
- **Functional API**：`@task` 包裹副作用和非确定性操作（读时钟、随机数），恢复时直接取存档结果，不重新执行。
- **Channel 类型**（运行时层）：`LastValue`（默认，覆盖）、`Topic`（发布订阅，可累积/去重）、`BinaryOperatorAggregate`（reducer 对应的聚合通道）。
- **错误码页面**：`INVALID_CONCURRENT_GRAPH_UPDATE`。

### 来源说明

- 「AI 历史知识点」的 Pregel 论文 PDF 为普林斯顿课程页存档的原文，非 LangChain/Claude 来源。
- Claude Academy 暂无 LangGraph 对应课程；Claude 侧观点引自官方文章 Building always-on agents。

### 延伸学习

- [Fault tolerance](https://docs.langchain.com/oss/python/langgraph/fault-tolerance)：官方的失败分类表和幂等写法，面试前建议通读一遍。
- [Use time-travel](https://docs.langchain.com/oss/python/langgraph/use-time-travel)：回放与分叉的完整示例。

### 参考资料

- [Fault tolerance - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/fault-tolerance)
- [Checkpointers - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/checkpointers)
- [Use time-travel - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/use-time-travel)
- [Build sequences, branches, and loops - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/graph-api-control-flow)
- [Build checklist for the Graph API - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/use-graph-api)
- [Functional API overview - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/functional-api)
- [LangGraph runtime - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/pregel)
- [Building always-on agents - Claude](https://claude.com/resources/articles/building-always-on-agents)
