# Day 17 · LangGraph（2026-10-15）

## 读前三问

1. LangGraph 的状态图由哪三样东西组成？「条件边」解决什么问题？
2. checkpoint 是什么？为什么人工中断（interrupt）离不开它？
3. 用 `interrupt()` 暂停后再恢复，节点是从哪一行继续跑的？这会带来什么坑？

## 从一个场景开始

车机助手新增「电量低自动预约充电桩」：搜索充电站 → 判断要不要车主确认 → 下单预约。

用 Day 16 的写法，整个流程交给一个 Agent 循环，问题来了：

- 预约要花钱。你希望「超过 20 元必须车主点确认」是一条**写死的规则**，而不是指望模型每次都记得问。
- 车主看到确认弹窗时正在开车，十分钟后才点。这期间进程可能被系统回收，原来的 Python 调用栈早就没了，恢复时从哪儿接着跑？
- 出了问题想回看「第二步时状态是什么」，单个循环里什么记录都没有。

需要的是一张**流程图**：哪些步骤固定、哪里分岔、哪里停下等人，图上画清楚；每走一步自动存档，随时能从存档恢复。这就是 **LangGraph**。

## 一句话总结

LangGraph 把 Agent 写成一张状态图：节点干活、边决定下一步、共享状态在节点间流动。每走一步自动存档，所以能暂停等人确认、断点续跑、回看历史。

## 先跑起来

下面这张图就是上面的场景。为了专注于图本身，三个节点都不调模型（真实项目里 `search` 或路由会调 LLM），所以不需要 API Key。

```python
# car_charge_graph.py —— 车机「预约充电」状态图：条件路由 + checkpoint + 人工确认（langgraph 1.2.15）
from typing import Annotated, TypedDict
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command

class State(TypedDict):
    battery: int                              # 当前电量（覆盖更新）
    station: str                              # 选中的充电站
    fee: float                                # 预约费
    log: Annotated[list[str], operator.add]   # 执行日志（追加更新）

def search(state: State):
    return {"station": "A站", "fee": 30.0, "log": ["search: 找到A站，预约费30元"]}

def route(state: State) -> str:  # 条件边：费用超过20元要车主确认
    return "confirm" if state["fee"] > 20 else "book"

def confirm(state: State):
    print("   >> confirm 节点开始执行")
    ok = interrupt(f"预约{state['station']}需付{state['fee']}元，是否确认？")
    return Command(goto="book" if ok else END, update={"log": [f"confirm: 车主回答 {ok}"]})

def book(state: State):
    return {"log": [f"book: 已预约{state['station']}"]}

g = StateGraph(State)
g.add_node("search", search); g.add_node("confirm", confirm); g.add_node("book", book)
g.add_edge(START, "search")
g.add_conditional_edges("search", route, ["confirm", "book"])
g.add_edge("book", END)  # confirm 之后去哪，由它返回的 Command 决定
app = g.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "car-001"}}
r = app.invoke({"battery": 15, "log": []}, cfg)
print("1) 暂停，问车主：", r["__interrupt__"][0].value)
print("2) 存档里下一步：", app.get_state(cfg).next)
r = app.invoke(Command(resume=True), cfg)   # 车主点了「确认」，同一个 thread_id 续跑
print("3) 恢复后日志：", r["log"])
print("4) 存档快照数：", len(list(app.get_state_history(cfg))), "下一步：", app.get_state(cfg).next)
```

**运行**：`pip install langgraph`（实测 langgraph 1.2.15），然后 `python car_charge_graph.py`。

**实测输出**（2026-10-15，连续运行 2 次结果一致）：

```
   >> confirm 节点开始执行
1) 暂停，问车主： 预约A站需付30.0元，是否确认？
2) 存档里下一步： ('confirm',)
   >> confirm 节点开始执行
3) 恢复后日志： ['search: 找到A站，预约费30元', 'confirm: 车主回答 True', 'book: 已预约A站']
4) 存档快照数： 5 下一步： ()
```

**该看什么**：
- 第 2 行：第一次 `invoke` 没有报错也没有跑完，而是把问题交回给调用方；存档里记着「下一步是 confirm」。
- 「confirm 节点开始执行」打印了**两次**：恢复时节点从头重跑，不是从 `interrupt` 那一行接着跑（见核心概念 4）。
- 第 3 行：`log` 三条是累加的，`station` 是覆盖的——同一个状态，不同字段合并规则不同（见核心概念 1）。

## 核心概念

### 1. State：所有节点共享的一份数据

- **是什么**：一个 `TypedDict`，每个节点读它、返回**部分更新**。每个字段可以挂一个 reducer（合并函数），决定新值怎么并进去：不挂就是覆盖，挂 `operator.add` 就是追加。
- **没有它会怎样**：节点之间只能靠参数层层传递，加一个字段要改一串函数签名。
- **对应你熟悉的**：MVI 里的 `UiState` + reducer。每个 Intent 产出一个局部变化，reducer 合成新状态。

聊天场景用 `add_messages` 作 reducer：新消息追加，ID 相同则替换。官方建议状态里存**原始数据**（搜索结果、分类结果），prompt 在节点内部临时拼，这样改 prompt 不用改状态结构。

### 2. 节点与边：谁干活、下一步去哪

- **是什么**：节点是普通函数；边决定执行顺序。普通边固定走向；**条件边**在节点跑完后调用一个路由函数，按它的返回值选下一个节点。`START`、`END` 是虚拟的起点和终点。
- **没有它会怎样**：「超过 20 元必须确认」只能写进 prompt 求模型遵守。写成条件边，就是代码保证的硬规则。
- **对应你熟悉的**：Jetpack Navigation 的导航图。节点是 destination，边是 action，条件边像在代码里根据登录状态决定 `navigate` 到哪。

节点也可以自己返回 `Command(goto=..., update=...)`，把「改状态」和「跳转」合在一步，适合 `confirm` 这种跳转取决于节点内部结果的情况。官方建议一个节点的出口二选一：要么静态边，要么动态路由，别混用。

### 3. Checkpoint 与 thread：每一步自动存档

- **是什么**：编译时传入 checkpointer，图每走完一个 super-step（一轮并行执行的节点）就存一份快照。快照按 `thread_id` 分组，一个 thread 就是一次会话。上面的例子存了 5 份。
- **没有它会怎样**：进程一挂，进度全丢；也不可能「暂停十分钟后继续」。
- **对应你熟悉的**：`onSaveInstanceState` + `SavedStateHandle`。进程被杀后凭 Bundle 恢复，`thread_id` 就像 Bundle 的 key。

它一次性带来四个能力：多轮记忆、人在回路、时间回溯（回到旧快照换条路跑）、容错（从最后一个成功的步骤重跑）。`InMemorySaver` 只用于实验，生产用 Postgres 等持久化实现。

### 4. interrupt：停下来等人

- **是什么**：节点里调用 `interrupt(问题)`，图立刻暂停、存档，把问题交回调用方。人回答后，用**同一个 `thread_id`** 调 `invoke(Command(resume=答案))`，答案就成了 `interrupt()` 的返回值。
- **没有它会怎样**：要么让模型自己决定花钱，要么开个线程死等用户，进程一重启就全丢了。
- **对应你熟悉的**：`registerForActivityResult`。发起后当前流程挂起，结果回来再继续，中间 Activity 可能已经被重建。

关键细节：**恢复时节点从头重跑**。所以 `interrupt()` 之前不要放有副作用的操作（扣款、写库），要么放在之后，要么拆成单独的节点。

## 高频面试题

**1. LangGraph 和直接写一个 while 循环的 Agent 有什么区别？**
- 流程显式化：固定步骤、分支、循环都画在图上，能做硬约束。
- 每步自动存档，免费得到暂停恢复、断点续跑、历史回溯。
- 代价是多一层抽象；Anthropic 建议先直接用 API，用框架就要搞清楚底层在做什么。

**2.（追问）`interrupt()` 恢复后，节点里 `interrupt` 前面的代码会再执行一遍吗？会出什么问题？**
- 会，节点从头重跑。
- 前面若有扣款、创建订单、往列表追加，就会重复执行。
- 解决：副作用放到 `interrupt` 之后或单独节点，并做成幂等。
- 不要用裸 `try/except` 包住 `interrupt()`，它靠抛异常实现暂停，会被吞掉。

**3.（追问）不传 `thread_id` 或换了 `thread_id` 恢复，会怎样？**
- checkpointer 靠 `thread_id` 找存档；没有它无法恢复中断。
- 换了 ID 就是一个全新会话，之前的进度和记忆都看不到。
- 车机上可以用「车辆 ID + 会话 ID」组合作 thread_id。

**4.（场景题）车机助手要执行「开车窗、调空调、预约付费服务」，怎么用 LangGraph 设计确认流程？**
- 意图识别节点（LLM）→ 条件边按风险分级：低风险直接执行，高风险进确认节点。
- 确认节点用 `interrupt` 把操作详情推给 HMI 弹窗，车主点击后 `Command(resume=...)` 续跑。
- checkpointer 用持久化实现，熄火重启后仍能从确认步骤恢复；执行节点做幂等。
- 风险规则写在路由函数里，不依赖模型自觉。

## 常见误区

1. **「用了 LangGraph 就有记忆」**：不传 checkpointer 时，每次 `invoke` 都从空状态开始。记忆、中断、回溯全部依赖 checkpointer + `thread_id`。
2. **「列表字段直接返回就会追加」**：没挂 reducer 的字段是**覆盖**。上例如果 `log` 不写 `Annotated[..., operator.add]`，最后只剩一条日志，并行节点同时写还会报冲突。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：定义状态、写确认节点、连图、编译**（取自 `car_charge_graph.py`，状态字段和提问文字略有精简）

```python
# 定义所有节点共享的状态
class State(TypedDict):
    # 充电站名：后写入的覆盖前面的
    station: str
    # 日志：挂上「相加」规则，各节点的输出会追加
    log: Annotated[list[str], operator.add]

# 确认节点：暂停等车主回答
def confirm(state: State):
    # 把问题交给调用方并暂停；恢复时这里拿到车主的答案
    ok = interrupt(f"预约{state['station']}，是否确认？")
    # 同时更新日志并决定跳去哪：同意去下单，拒绝就结束
    return Command(goto="book" if ok else END, update={"log": [f"车主回答 {ok}"]})

# 用状态类型创建图
g = StateGraph(State)
# 注册三个节点：名字 + 函数
g.add_node("search", search); g.add_node("confirm", confirm); g.add_node("book", book)
# 入口：从 START 先进入搜索
g.add_edge(START, "search")
# 搜索后由路由函数决定去确认还是直接下单
g.add_conditional_edges("search", route, ["confirm", "book"])
# 下单后结束
g.add_edge("book", END)
# 编译，并挂上内存存档器
app = g.compile(checkpointer=InMemorySaver())

# 会话配置：thread_id 决定读写哪份存档
cfg = {"configurable": {"thread_id": "car-001"}}
# 车主确认后，用同一个 thread_id 续跑
r = app.invoke(Command(resume=True), cfg)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `thread_id` 要放在 `{"configurable": {...}}` 里面，不是直接放在 config 顶层。
- 恢复用 `Command(resume=...)`，跳转用 `Command(goto=...)`，参数名别混。
- `add_conditional_edges` 的第二个参数传函数本身 `route`，不是 `route(state)`；`compile()` 之前图不能调用。

**昨日复习（只做第③步）**：Day 16 的「把子 Agent 包装成工具，交给主管」（`delegate` + `ask_nav` / `ask_media`）。只看你自己写过的注释，重写代码，重点核对：`invoke` 传字典、取 `[-1].text`、docstring 在函数体第一行。

## 动手练习（30 分钟）

在 `car_charge_graph.py` 的基础上改：

1. 把第二次 `invoke` 改成 `Command(resume=False)`，确认日志里没有 `book`，且最后 `next` 为空。
2. 改 `route`：电量 `battery < 10` 时视为紧急，跳过确认直接 `book`。分别用电量 5 和 15 跑一遍。
3. 在 `confirm` 的 `interrupt` **之前**加一行「往 `log` 写一条 `已扣费`」，观察恢复后是否出现两次。然后把它挪到 `interrupt` 之后，再跑一次，用一句话写下结论。

## 读后自测

1. LangGraph 的状态图由哪三样东西组成？「条件边」解决什么问题？→ 见第 6 节「核心概念 1、2」
2. checkpoint 是什么？为什么人工中断离不开它？→ 见第 6 节「核心概念 3」
3. 用 `interrupt()` 暂停后再恢复，节点是从哪一行继续跑的？这会带来什么坑？→ 见第 6 节「核心概念 4」

---

## 参考区（用时查，不必通读）

### 细节速查

- **导入**：`StateGraph, START, END` 来自 `langgraph.graph`；`interrupt, Command, Send` 来自 `langgraph.types`；`InMemorySaver` 来自 `langgraph.checkpoint.memory`。
- **State 类型**：`TypedDict`（常用）、dataclass、Pydantic 模型；预置 `MessagesState`（带 `messages` + `add_messages`），可继承加字段。
- **`add_conditional_edges(源节点, 路由函数, 映射)`**：映射可以是 dict（返回值 → 节点名），也可以是可能目标的列表。
- **`Command` 参数**：`update`（更新状态）、`goto`（跳转）、`graph`（如 `Command.PARENT` 跳到父图）、`resume`（恢复中断）。返回类型可标注 `Command[Literal["节点名"]]` 以便画图。注意：同一节点上的静态边会和 `goto` 同时生效。
- **`Send`**：map-reduce，从条件边返回 `[Send("节点", 子状态), ...]`，数量运行时决定。
- **recursion_limit**：限制 super-step 数，超了抛 `GraphRecursionError`；1.0.6 起默认 1000；写在 config 顶层：`invoke(x, config={"recursion_limit": 5})`。节点内可用 `RemainingSteps` 提前收尾。
- **StateSnapshot 字段**：`values`、`next`（空元组表示已结束）、`config`（含 `thread_id`、`checkpoint_id`）、`metadata`（`source` 为 input/loop/update，`step`、`writes`）、`created_at`、`parent_config`、`tasks`。
- **读存档**：`get_state(cfg)` 最新；cfg 里加 `checkpoint_id` 取指定快照；`get_state_history(cfg)` 由新到旧。
- **Checkpointer 实现**：`InMemorySaver`（实验）、`SqliteSaver`（`langgraph-checkpoint-sqlite`，本地）、`PostgresSaver`（`langgraph-checkpoint-postgres`，生产）；异步用 `AsyncSqliteSaver` / `AsyncPostgresSaver`；另有 MongoDB、Cosmos DB。
- **pending writes**：同一 super-step 里已成功的节点输出会被保存，失败恢复时不再重跑。
- **interrupt 规则**：payload 要可 JSON 序列化；同一节点多个 `interrupt` 按顺序（下标）匹配恢复值，不要在不确定的循环里调用；校验重试用「一次 interrupt + 条件边回环」而不是 `while True`。新代码可用 `stream_events(..., version="v3")` 读取中断。
- **错误分类处理（Thinking in LangGraph）**：瞬时错误 → 重试策略；LLM 可修复 → 错误写进状态、回环；用户可修复 → interrupt；意外错误 → 直接抛出。

### 来源说明

本篇未使用非 LangChain/Claude 来源。Claude Academy 暂无 LangGraph 对应课程；Anthropic 对框架的态度引自 Building Effective AI Agents。

### 延伸学习

- [Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph)（官方设计方法教程，用客服邮件 Agent 贯穿）
- Day 24「护栏与人在回路」会在今天的 interrupt 基础上讲审批与可逆设计。

### 参考资料

- [Graph API overview - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [Interrupts - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [Checkpointers - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/checkpointers)
- [Thinking in LangGraph - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph)
- [Building Effective AI Agents - Anthropic](https://www.anthropic.com/research/building-effective-agents)
