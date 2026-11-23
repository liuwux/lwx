# Day 26 · 部署与生产化（第二轮：重放语义、超时与重试叠加、优雅停机、并发消息）（2026-11-23）

## 今日 AI 动态（2026-11-23）

### 趋势观察

这个周末各家几乎没有产品发布，唯一的新官方内容是 OpenAI 首席科学家写的一篇治理文章。它提议用一个可量化的预训练指标（困惑度）来协调各家的安全上限。推测：前沿实验室之间的讨论正从「各自承诺」转向「用共同指标划线」，这会影响未来大模型的发布节奏。

### 今日要点

- OpenAI 发文提议把预训练困惑度作为全行业共用的进展指标，并据此设定「共同上限」。

### 分公司动态

#### 1. OpenAI

**[Perplexity and Intelligence](https://openai.com/index/perplexity-and-intelligence/)**（2026-11-22，作者 Jakub Pachocki）

- 文章首次公开了 OpenAI 自 GPT-3.5 以来在内部留出代码数据集上测得的 log-perplexity，并把它和能力出现的节点对应起来：GPT-3.5 约 0.99，对话变得流畅；GPT-4 约 0.91，思维链推理开始有效；约 0.76 时，模型能在开放数学问题上产出研究级结果。
- 文章预计在 0.68–0.7 区间会出现更大的能力跃迁。它提议各家共享评测规范、对安全论证做红队审查，并设定与困惑度挂钩的「共同上限」：在监控或理解能力达到对应里程碑之前，各家都不越过这条线。
- 文章也承认这个指标有局限：各家数据集不同，指标和实际任务能力不直接对应，也反映不出思维链这类预训练之外的提升。
- **为什么重要**：如果头部实验室真的按同一个指标设「刹车点」，那么 Agent 开发者在规划产品时，对下一代模型何时出现、能力有多强的预期就要跟着调整。

### 动态来源

- [OpenAI · Perplexity and Intelligence](https://openai.com/index/perplexity-and-intelligence/)

---

## 读前三问

1. LangGraph 从检查点恢复时，是从「中断的那一行」继续吗？节点里的扣款为什么可能执行两次？
2. 你给节点配了 3 次重试，节点里又用了默认配置的 Anthropic SDK。高峰期 API 返回 529，最坏情况下会发出几次请求？怎么改？
3. 发版要滚动重启 Worker，正在跑的 Agent 任务怎么办？用户在任务没跑完时又发来一条消息，又怎么办？

## 从一个场景开始

第一轮我们用重试策略加 SQLite 检查点，做了一个「远程预约充电」的图，进程崩溃后可以续跑。上线一周后，客服收到投诉：有车主被扣了两次 45 元。

查日志发现：支付网关其实第一次就扣款成功了，但回执晚了 5 秒才回来。节点超时后被重试，又扣了一次。同一周还出了两件事：一是发版时 Kubernetes 发出 SIGTERM，跑到一半的任务直接被杀；二是车主等不及，连发了两次「帮我约充电」，后台启动了两个 run，抢着写同一个 thread。

第一轮解决的是「崩溃后能接着跑」。第二轮要解决的是**接着跑的时候会发生什么**：哪些代码会重跑，重试会叠加成多少次，怎么体面地停机，以及同一个会话里的并发消息怎么处理。

## 一句话总结

恢复是从上一个存档点重放，不是从断点续跑，所以有副作用的操作必须幂等。重试只放在一层，并且要设上限。停机时先排空再退出，同一会话的并发消息要明确排队、拒绝还是打断。

## 先跑起来

一个 4 步的图：报价 → 支付 → 推送 → 归档。支付节点设了 1 秒超时和 3 次重试。第 1 次支付会「扣款成功但回执卡住」。推送完成后模拟收到 SIGTERM，然后在新实例上续跑。

```python
# charge_prod.py —— 远程预约充电：超时+重试+幂等键，停机排空后从检查点续跑（langgraph 1.2.24）
import asyncio, uuid
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import RetryPolicy
from langgraph.runtime import RunControl
from langgraph.errors import GraphDrained
from langgraph.checkpoint.memory import InMemorySaver
LEDGER, RAW, ctl = {}, [], RunControl()       # 网关账本：按幂等键去重 / 对照组：不去重
class S(TypedDict, total=False):
    order_id: str
    receipt: str
async def quote(s: S):                        # 随机值在节点里生成并写进 state，恢复时不会变
    return {"order_id": uuid.uuid4().hex[:6]}
async def pay(s: S):
    key = f"charge:{s['order_id']}"           # 幂等键取自持久化的 state，每次重试都相同
    LEDGER.setdefault(key, 45.0); RAW.append(45.0)
    print(f"  pay 第 {len(RAW)} 次：网关已扣款")
    if len(RAW) == 1:
        await asyncio.sleep(5)                # 第 1 次：扣款成功，但回执迟迟不回来
    return {"receipt": key}
async def notify(s: S):
    print("  推送车主：已扣款", LEDGER[s["receipt"]], "元")
    ctl.request_drain("SIGTERM")              # 模拟此刻收到停机信号：跑完本步就停
    return {}
g = StateGraph(S)
g.add_node("quote", quote)
g.add_node("pay", pay, timeout=1.0, retry_policy=RetryPolicy(max_attempts=3, initial_interval=0.1))
g.add_node("notify", notify)
g.add_node("archive", lambda s: print("  归档完成") or {})
for a, b in [(START, "quote"), ("quote", "pay"), ("pay", "notify"), ("notify", "archive"), ("archive", END)]:
    g.add_edge(a, b)
app = g.compile(checkpointer=InMemorySaver())
cfg = {"configurable": {"thread_id": "car-42"}}
async def main():
    try:
        await app.ainvoke({}, cfg, control=ctl, durability="sync")
    except GraphDrained as e:
        print("被排空:", e.reason, "| 待执行:", app.get_state(cfg).next)
    await app.ainvoke(None, cfg)              # 新实例上线：传 None 从检查点续跑
    print("实际扣款:", sum(LEDGER.values()), "元 | 不加幂等键会扣:", sum(RAW), "元")
asyncio.run(main())
```

**运行**：`pip install langgraph && python charge_prod.py`

**实测输出**（2026-11-23，Python 3.13，langgraph 1.2.24）：

```
  pay 第 1 次：网关已扣款
  pay 第 2 次：网关已扣款
  推送车主：已扣款 45.0 元
被排空: SIGTERM | 待执行: ('archive',)
  归档完成
实际扣款: 45.0 元 | 不加幂等键会扣: 90.0 元
```

**该看什么**：

- `pay` 确实执行了两次：第 1 次超时被取消，重试时整个节点从头再跑。幂等键保证只扣了一次钱，而对照组 `RAW` 扣了两次。
- 幂等键能保持不变，是因为 `order_id` 在上一步就写进了 state。如果在 `pay` 里现场生成 uuid，每次重试的键都不一样，去重就失效了。
- 排空后 `next` 是 `archive`，续跑时 `quote`、`pay`、`notify` 都没有重跑。生产环境要把 `InMemorySaver` 换成 Postgres 这类持久化的检查点，新进程才能读到存档。

## 核心概念

### 1. 恢复是「重放」，不是「续跑」

- **是什么**：LangGraph 只在每一步（superstep）结束时存档，不会在函数中间存。恢复和重试时，都是把整个节点函数从头再跑一遍，所以副作用至少执行一次，可能执行多次。
- **没有它会怎样**：你会以为「崩溃后从断点继续」，于是把扣款、发短信直接写在节点里，结果一恢复就重复执行。
- **对应你熟悉的**：Activity 被系统回收后重建。`onCreate` 会整个重新执行，只有 `savedInstanceState` 里的东西能保留下来。

三条规则：

- 随机数、当前时间这类每次结果不同的值，要先算出来写进 state，或者放进 `@task`。task 的结果会存档，恢复时直接取回，不会重新计算。
- 每个副作用用一个从 state 推出来的幂等键，交给下游去重。
- `interrupt()` 之前的代码在每次恢复时都会重跑，所以那段代码也要幂等。

### 2. 超时与重试：只在一层重试，并且要有上限

- **是什么**：LangGraph 可以给节点设 `timeout`（只支持 async 节点），超时会抛出 `NodeTimeoutError`，默认会被重试。`RetryPolicy` 默认最多尝试 3 次（包括第一次），退避时带随机抖动，对 `ValueError` 这类代码 bug 不重试。
- **没有它会怎样**：会出现重试风暴。Anthropic Python SDK 默认会对 429、5xx、连接错误自动重试 2 次，也就是每次调用最多发 3 个请求。外面再套 3 次节点重试，就是 3×3=9 次。服务过载（529）时，大家一起放大流量，只会让过载更严重。
- **对应你熟悉的**：OkHttp 开了 `retryOnConnectionFailure`，业务层又写了一层 retry，两层叠在一起就会成倍放大。

做法：

- 指定一层负责重试，另一层关掉，比如 `Anthropic(max_retries=0)`，只用节点重试；或者反过来，节点只对超时重试。
- 遵守 `retry-after` 响应头，不要固定间隔重试。
- 按错误类型区分：`enforced_spend_limit_reached`（达到月度消费上限）这种 429 重试多少次都不会好，应该直接告警。

### 3. 优雅停机：先排空，再退出

- **是什么**：收到 SIGTERM 后调用 `RunControl.request_drain()`。图会跑完当前这一步，在步与步之间停下，存好检查点，再抛出 `GraphDrained`。新实例用同一个 `thread_id` 传入 `None`，就能接着跑。
- **没有它会怎样**：进程被直接杀掉，正在执行的那一步白做了。如果那一步有副作用，还要靠幂等去兜底。
- **对应你熟悉的**：`onStop` 里先保存草稿、释放资源，而不是等系统强杀。

托管的 Agent Server 也是同样的思路：实例收到 SIGINT 后不再接新任务，正在跑的任务有一小段时间收尾，没跑完的会重新入队。Worker 会定期写心跳，一个清扫任务每 2 分钟检查一次，把丢了心跳的任务重新入队。

### 4. 同一会话的并发消息（double texting）

- **是什么**：一个 run 还没跑完，用户又发来新消息。LangSmith Deployment 提供四种策略：`enqueue`（默认，排队等前一个跑完）、`reject`（拒绝新消息）、`interrupt`（停掉当前 run，保留进度，带着新消息继续）、`rollback`（停掉当前 run 并撤销它的进度，用新消息重新开始）。
- **没有它会怎样**：两个 run 同时写一个 thread，状态互相覆盖。
- **对应你熟悉的**：`Flow` 的 `collectLatest` 对应 interrupt / rollback，`buffer` 对应 enqueue，`conflate` 加丢弃策略对应 reject。

注意：`interrupt` 停下来时，可能有一个工具调用已经发出去但还没返回。所以涉及钱的流程不要用 `interrupt` 或 `rollback`，而应该用 `enqueue` 或 `reject`。

## 高频面试题

**1. LangGraph 的三种 durability 模式有什么区别？涉及支付的流程选哪个？**

- `exit`：只在运行结束时存档，最快，但中途崩溃就无法恢复。
- `async`：在执行下一步的同时异步存档，进程崩溃时有小概率丢掉最后一个存档。Agent Server 默认用这个。
- `sync`：存档写完才执行下一步，最可靠，代价是一些性能。
- 支付流程选 `sync`。官方文档特别提到：如果幂等键刚生成、副作用还没执行时进程崩溃，要用 `sync`，才能保证恢复后用的还是同一个键。

**2.（追问）上线后新版本给图加了一个节点、改了一个节点名，还在等人工审批的旧 thread 恢复时会怎样？**

- 改名或删除节点最危险：恢复时按存档里的旧名字找不到节点，run 直接失败。
- 增删边是安全的，因为边的结构不存进检查点。
- state 的新字段用 `NotRequired` 添加；要改名，就先加新字段，等旧 thread 跑完再删旧字段。
- 发版前先查还有哪些 thread 处于 interrupted 或 busy 状态。必要时加一个版本标记，用条件边分流，等 v1 的 thread 全部结束后再删掉旧逻辑。

**3.（场景题）高峰期 Claude 返回大量 529，你的车载助手 Agent 怎么扛住？**

- 只保留一层重试，指数退避加抖动，遵守 `retry-after`，设好总次数和总时长的上限。
- 降级：交互请求切到备用模型（回退），非实时任务（夜间行程报告）改用 Batch API，异步跑，价格是一半。
- 削峰：任务进队列，由 Worker 按限额消费，用户端先返回「处理中」，用流式结果减少等待感。
- 观察 `anthropic-ratelimit-*` 响应头和 529 比例，提前限流，不要等到报错。

**4.（追问）用户连发两条「帮我约充电」，你选哪种 multitask 策略？为什么？**

- 有扣款这种副作用，不选 `interrupt` 或 `rollback`，因为它们可能让工具调用停在一半。
- 默认用 `enqueue`。第二条消息排队，等第一单完成后，Agent 看到「已预约」，就不会再下单。
- 如果要防止误触，可以用 `reject`，前端提示「上一条还在处理」。
- 下游扣款仍然用幂等键兜底，策略选对了也不能省掉这一层。

## 常见误区

1. **「有了检查点就不会重复执行」**：检查点只保证已经完成的步骤不重跑。正在执行的那一步，在重试、恢复、超时之后都会从头再跑，副作用仍然要幂等。
2. **「重试次数越多越稳」**：每一层都开重试，次数会成倍放大，过载时等于自己在攻击服务。只在一层重试，设上限，加抖动，对不可恢复的错误直接失败。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：幂等支付节点 + 超时重试 + 排空续跑**（取自 `charge_prod.py`）

```python
# 支付节点：用 state 里已有的订单号拼出幂等键
async def pay(s: S):
    key = f"charge:{s['order_id']}"
    # 网关按键去重：同一个键只记一笔；对照组每次都记
    LEDGER.setdefault(key, 45.0); RAW.append(45.0)
    # 第 1 次调用时模拟回执卡住，让节点超时
    if len(RAW) == 1:
        await asyncio.sleep(5)
    # 把幂等键当作回执写回 state
    return {"receipt": key}
# 推送节点：通知车主后，模拟收到停机信号，请求排空
async def notify(s: S):
    print("  推送车主：已扣款", LEDGER[s["receipt"]], "元")
    ctl.request_drain("SIGTERM")
    return {}
# 注册支付节点：1 秒超时，最多尝试 3 次，首次重试间隔 0.1 秒
g.add_node("pay", pay, timeout=1.0, retry_policy=RetryPolicy(max_attempts=3, initial_interval=0.1))
# 主流程：带上运行控制，每步同步存档
async def main():
    try:
        await app.ainvoke({}, cfg, control=ctl, durability="sync")
    # 被排空时，打印原因和下一步要跑的节点
    except GraphDrained as e:
        print("被排空:", e.reason, "| 待执行:", app.get_state(cfg).next)
    # 新实例用同一个 thread，传 None 从检查点续跑
    await app.ainvoke(None, cfg)
```

**三步走**

1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**

- 幂等键必须来自 state（`s['order_id']`），不能在 `pay` 里现场生成 `uuid`，否则每次重试的键都不一样。
- `timeout` 只对 `async def` 节点有效。同步节点加 `timeout` 会在编译时报错，节点里也要用 `await asyncio.sleep`，不能用 `time.sleep`。
- 续跑传的是 `None`，不是 `{}`。传 `{}` 会被当成一次新输入，从头开始跑。

**昨日复习（只做第③步）**：第二轮 Day 25 的「缓存计费循环」。只看你自己写的注释重写，重点核对：`keys` 的 `range(1, len(prompt) + 1)`、`max(..., default=-1)`、写价只算 `count(prompt) - read` 这一部分。

## 动手练习（30 分钟）

在 `charge_prod.py` 上改：

1. 把 `InMemorySaver` 换成 `SqliteSaver`（或 `AsyncSqliteSaver`），把「排空」和「续跑」拆成两次独立运行的进程，确认第二次运行不会重新扣款。
2. 把 `order_id` 的生成挪进 `pay` 节点，重新运行，观察 `LEDGER` 里出现两笔不同的键。然后再改回来。
3. 让 `pay` 每次都卡住，观察 3 次尝试都失败后抛出什么异常，再用 `retry_on` 让它只对 `NodeTimeoutError` 重试。

## 读后自测

1. LangGraph 从检查点恢复时，是从「中断的那一行」继续吗？节点里的扣款为什么可能执行两次？见第 6 节「先跑起来」和第 7 节「核心概念 1」
2. 你给节点配了 3 次重试，节点里又用了默认配置的 Anthropic SDK。高峰期 API 返回 529，最坏情况下会发出几次请求？怎么改？见第 7 节「核心概念 2」
3. 发版要滚动重启 Worker，正在跑的 Agent 任务怎么办？用户在任务没跑完时又发来一条消息，又怎么办？见第 7 节「核心概念 3」「核心概念 4」

## AI 历史知识点：Building Effective Agents（2024，Anthropic）

**是什么**：Anthropic 发表的一篇工程文章，总结了和几十个团队合作构建 Agent 的经验。它把「智能体系统」分成两类：**工作流**（LLM 和工具按预先写好的代码路径编排）和 **Agent**（LLM 自己决定流程和用哪个工具）。文中列出了五种常用的工作流模式：提示链、路由、并行化、编排者-工作者、评估者-优化者。

**何时由谁提出**：2024 年 12 月 19 日发布，作者是 Anthropic 的 Erik S. 和 Barry Zhang。

**为什么重要**：在各种 Agent 框架层出不穷的时候，它给出了一个很克制的建议：先用最简单的方案，确实需要时才加复杂度；先直接调 API，用框架也要搞懂底层。文中提出的三条原则（设计保持简单、让规划步骤透明可见、认真设计 Agent 与计算机的接口 ACI）后来成了面试和工程讨论里常引用的说法。

**和今天的趋势有什么联系**：今天讲的超时、重试、幂等，都在为「Agent 会犯错、出错会累积」这一点兜底。文章本身就提醒，Agent 成本更高，错误会层层叠加，需要在沙箱里充分测试并加上护栏。页面上的编者按也指出，文中提到的很多工具已经过时，并指向更新的 Managed Agents 文章：理念还在，部署形态在往托管方向走。

来源：[Building effective agents - Anthropic](https://www.anthropic.com/research/building-effective-agents)

## 参考区（用时查，不必通读）

### 细节速查

**LangGraph（1.2.x）**

- `durability="exit" | "async" | "sync"`，可以传给 `invoke` / `stream` / `ainvoke`。开源版文档没写默认值，Agent Server 的默认值是 `"async"`。
- `RetryPolicy(initial_interval=0.5, backoff_factor=2.0, max_interval=128.0, max_attempts=3, jitter=True, retry_on=default_retry_on)`。
- 默认 `retry_on` 不重试这些异常（含子类）：`ValueError`、`TypeError`、`ArithmeticError`、`ImportError`、`LookupError`、`NameError`、`SyntaxError`、`RuntimeError`、`ReferenceError`、`StopIteration`、`StopAsyncIteration`、`OSError`。`requests` / `httpx` 的错误只在 5xx 时重试。`NodeTimeoutError` 默认会重试。
- `add_node(..., timeout=秒数 | timedelta | TimeoutPolicy(run_timeout, idle_timeout, refresh_on="auto"|"heartbeat"))`。只支持 async 节点；超时靠 asyncio 取消实现，阻塞调用会让超时生效得更晚。`idle_timeout` 可以用 `runtime.heartbeat()` 续期。可以用 `g.set_node_defaults(retry_policy=..., timeout=...)` 给所有节点设默认值，`add_node` 里的设置优先。
- `runtime.execution_info.node_attempt`：当前是第几次尝试，从 1 开始。错误处理器在重试用完后才执行；`interrupt()` 不经过重试，也不经过错误处理器。
- 排空：`RunControl()` → `invoke(..., control=ctl)` → `ctl.request_drain(reason)` → 抛出 `GraphDrained(reason)`；每次 run 都要新建一个 `RunControl`。
- 函数式 API：task 的结果会存档，恢复时直接取回；重放从 entrypoint 开头开始。增删或调换恢复点之前的 `@task` / `interrupt` 调用顺序，会让缓存的结果对错位置。

**LangSmith Deployment（Agent Server）**

- 创建 run 的参数：`multitask_strategy`（reject / rollback / interrupt / enqueue，默认 enqueue）、`durability`（默认 async）、`if_not_exists`、`after_seconds`、`webhook`、`stream_resumable`。
- 架构：Postgres 存所有需要持久化的数据（run、thread、检查点、store）；Redis 只放临时数据（信号、取消、流式消息）。并发数 = 队列 Worker 数 × `N_JOBS_PER_WORKER`（默认 10）。同一个 thread 同一时间最多只有一个 run 在跑。
- 韧性：Worker 写心跳；清扫任务每 2 分钟把丢了心跳的 run 重新入队；Postgres 的瞬时错误最多重试 3 次。

**Claude API**

- 错误码：400 `invalid_request_error`、401 `authentication_error`、402 `billing_error`、403 `permission_error`、404 `not_found_error`、409 `conflict_error`、413 `request_too_large`、429 `rate_limit_error`、500 `api_error`、504 `timeout_error`、529 `overloaded_error`。
- Python SDK 默认值：`max_retries=2`，退避从 0.5 秒起、最长 8 秒，遵守 `retry-after`；超时 600 秒，其中连接超时 5 秒。预计超过 10 分钟的请求要用流式或 Batch。
- 限流响应头：`retry-after`、`anthropic-ratelimit-{requests,tokens,input-tokens,output-tokens}-{limit,remaining,reset}`。429 的具体原因看 `error.details.error_code`：`enforced_spend_limit_reached`（不要重试）、`ramp_limit_exceeded`（每分钟最多升一档）。
- Message Batches：价格五折；大多数批次 1 小时内完成，最长 24 小时；`processing_status` 有 `in_progress` / `canceling` / `ended` 三种；每批最多 10 万条请求或 256 MB；结果保留 29 天。

### 来源说明

- 示例代码已在 langgraph 1.2.24 上实测。`RunControl` / `GraphDrained` / `TimeoutPolicy` 的名称已在本地安装包的源码中核对过。
- LangGraph 的 durable-execution 页面这次没能打开，相关内容取自 checkpointers、fault-tolerance、functional-api、backward-compatibility 几个页面。
- 今日动态里，blog.google AI 板块、docs.bigmodel.cn、Qoder IDE 更新日志没能打开；Hugging Face 页面被代理拒绝访问。

### 延伸学习

- Claude Academy · Building with the Claude API：「Agents and workflows」模块
- Claude Academy · Claude Platform 101：「What are managed agents?」「Building your first managed agent」

### 参考资料

- [Persistence / Checkpointers - LangChain Docs](https://docs.langchain.com/oss/python/langgraph/checkpointers)
- [Fault tolerance - LangChain Docs](https://docs.langchain.com/oss/python/langgraph/fault-tolerance)
- [Functional API - LangChain Docs](https://docs.langchain.com/oss/python/langgraph/functional-api)
- [Interrupts - LangChain Docs](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [Backward compatibility - LangChain Docs](https://docs.langchain.com/oss/python/langgraph/backward-compatibility)
- [Double texting - LangSmith Docs](https://docs.langchain.com/langsmith/double-texting)
- [Agent Server scale - LangSmith Docs](https://docs.langchain.com/langsmith/agent-server-scale)
- [Scalability & resilience - LangSmith Docs](https://docs.langchain.com/langgraph-platform/scalability-and-resilience)
- [Errors - Claude Platform Docs](https://platform.claude.com/docs/en/api/errors)
- [Rate limits - Claude Platform Docs](https://platform.claude.com/docs/en/api/rate-limits)
- [Batch processing - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [anthropic-sdk-python `_constants.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/_constants.py)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
