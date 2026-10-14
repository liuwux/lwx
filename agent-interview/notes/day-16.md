# Day 16 · 多智能体架构（2026-10-14）

## 读前三问

1. 什么时候该把一个 Agent 拆成多个？拆了要付出什么代价？
2. 「子 Agent 当工具调用」（orchestrator-worker）和「handoff 转交控制权」有什么区别，各适合什么场景？
3. 为什么说多数编码任务、强依赖的流水线任务不适合用多 Agent？

## 从一个场景开始

车机语音助手上线半年，能力越加越多：导航、音乐、空调、车辆说明书问答、充电预约……所有工具都塞进一个 Agent，system prompt 写了三千字，工具列表有四十多个。

问题接连出现：

- 用户说「电量低了，找个充电站，顺便放点轻松的歌」，模型先搜充电站，返回一大段地点列表，再去放歌，上下文被搜索结果占满，后面开始答非所问。
- 两件互不相关的事只能一件一件做，用户要多等一倍时间。
- 导航组和媒体组改 prompt 时互相影响，谁改坏了都说不清。

一个自然的想法是：让一个「主管」听懂用户意图，把找充电站交给导航专家，把放歌交给媒体专家，两边同时开工，最后由主管汇总成一句话。这就是今天的主题：**多智能体架构**。

## 一句话总结

多智能体就是把任务拆给多个各有专长、各管一块上下文的 Agent 协作完成。它换来上下文隔离、并行和分团队开发，代价是 token 翻倍、协调变难；能用单 Agent 解决就不要拆。

## 先跑起来

下面一个文件就是上面的场景：主管 Agent（`boss`）把两个子 Agent（`nav`、`media`）包装成工具，一轮同时调用两个。

为了不依赖 API Key、让结果可复现，代码里用 LangChain 自带的假模型按「剧本」吐出预设回复，代替 Claude 做决策。Agent 循环、工具执行、并行调度都是 LangChain 的真实代码。换成真模型时，把 `model(...)` 换成 `init_chat_model("anthropic:<模型名>")` 并去掉剧本即可。

```python
# car_supervisor.py —— 车机「主管 Agent」把任务并行委派给两个子 Agent（langchain 1.4.5）
import time
from langchain.agents import create_agent
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI

# 没有 API Key：用「剧本模型」按顺序吐出预设回复，代替 Claude 做决策
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self
def model(*replies): return Scripted(messages=iter(replies))
def call(name, **args): return {"name": name, "args": args, "id": name}
@tool
def search_poi(keyword: str) -> str:
    """按关键词搜索附近地点"""
    time.sleep(1); return "A站 2.1km 空闲3桩；B站 4.5km 空闲0桩；C站 5km 空闲6桩"
@tool
def play_playlist(mood: str) -> str:
    """按心情播放歌单"""
    time.sleep(1); return "正在播放：轻松驾驶（32首）"

nav = create_agent(model(AI("", tool_calls=[call("search_poi", keyword="充电站")]),
                         AI("最近可用：A站，2.1km，空闲3桩")), tools=[search_poi])
media = create_agent(model(AI("", tool_calls=[call("play_playlist", mood="轻松")]),
                           AI("已播放「轻松驾驶」歌单")), tools=[play_playlist])

def delegate(agent, request):  # 子 Agent 从空白上下文开始，只把最后一句结论交回去
    out = agent.invoke({"messages": [{"role": "user", "content": request}]})
    print(f"   [子Agent内部产生 {len(out['messages'])} 条消息，全部丢弃]")
    return out["messages"][-1].text

@tool
def ask_nav(request: str) -> str:
    """导航子Agent：找地点、规划路线"""
    return delegate(nav, request)
@tool
def ask_media(request: str) -> str:
    """媒体子Agent：音乐、电台"""
    return delegate(media, request)

boss = create_agent(model(AI("", tool_calls=[call("ask_nav", request="找最近有空桩的充电站"),
                                             call("ask_media", request="放点轻松的歌")]),
                          AI("已为你找到A站（2.1km，3个空桩），并播放轻松歌单。")), tools=[ask_nav, ask_media])
t0 = time.time(); res = boss.invoke({"messages": [{"role": "user", "content": "电量低了，找个充电站，顺便放点轻松的歌"}]})
for m in res["messages"]: print(f"{m.type:>5}: {m.text or [c['name'] for c in m.tool_calls]}")
print(f"主管上下文共 {len(res['messages'])} 条，耗时 {time.time()-t0:.1f}s（两个子任务各需 1s）")
```

**运行**：`pip install langchain`（实测版本 langchain 1.4.5 / langgraph 1.2.15），然后 `python car_supervisor.py`。

**实测输出**（2026-10-14，连续运行 3 次结果一致）：

```
   [子Agent内部产生 4 条消息，全部丢弃]
   [子Agent内部产生 4 条消息，全部丢弃]
human: 电量低了，找个充电站，顺便放点轻松的歌
   ai: ['ask_nav', 'ask_media']
 tool: 最近可用：A站，2.1km，空闲3桩
 tool: 已播放「轻松驾驶」歌单
   ai: 已为你找到A站（2.1km，3个空桩），并播放轻松歌单。
主管上下文共 5 条，耗时 1.0s（两个子任务各需 1s）
```

**该看什么**：
- 第 4 行：主管在**一条**消息里同时发出两个调用，两个子任务各要 1 秒，总耗时却是 1.0 秒，说明它们是并行跑的。
- 前两行：每个子 Agent 内部都走了一遍「调工具→看结果→总结」，共 4 条消息，但这些全部丢弃。主管只收到一句结论，所以它的上下文只有 5 条，那串充电站列表从没进入过主管的上下文。
- 子 Agent 收到的只有 `request` 这一句话，不知道用户原话。这是一个风险，见「常见误区」第 1 条。

## 核心概念

### 1. 先问：真的需要多 Agent 吗？

- **是什么**：LangChain 文档总结了拆分的三个理由：**上下文管理**（每个 Agent 只装自己那一块知识）、**分团队开发**（各组独立维护自己的 Agent）、**并行**（独立子任务同时跑）。
- **没有它会怎样**：单 Agent 工具太多、prompt 太长时，选错工具、上下文被塞满的概率都会上升。
- **代价**：Anthropic 的数据是，普通 Agent 消耗的 token 约为聊天的 4 倍，多 Agent 系统约为 15 倍。

Anthropic 的建议是先用最简单的方案，只有效果确实提升了才加复杂度。面试时先说「单 Agent 优先」，再说拆分理由，会显得很稳。

### 2. Orchestrator-Worker：子 Agent 当工具用

- **是什么**：一个主管（orchestrator）分析任务、动态决定拆成哪几件事，交给工人（worker / subagent）去做，最后由主管汇总。在代码里，子 Agent 就是一个普通工具，参数是一句任务描述，返回值是一段结论。
- **没有它会怎样**：所有中间过程（搜索结果、读过的文件）都堆在一个上下文里。Claude Academy 的例子是：为了回答「哪个服务处理退款」要读 15 个文件，交给子 Agent 去读，主上下文里只留下问题和答案。
- **对应你熟悉的**：主线程把耗时工作丢给 `Dispatchers.IO` 上的协程，只拿回结果；协程内部的局部变量不会污染主线程。

它和固定的「并行」工作流不同：并行的子任务是事先写死的；orchestrator 根据每次的输入**临时决定**拆什么。

**委派说明要写清楚**：Anthropic 的多 Agent 研究系统要求每次委派都写明目标、输出格式、可用工具和来源、任务边界。说明写得模糊，子 Agent 会理解错，或者几个子 Agent 重复做同一件事。

### 3. Handoff：把控制权交出去

- **是什么**：当前 Agent 调用一个「转交」工具，修改状态里的 `active_agent`，此后由另一个 Agent **直接和用户对话**，而不是做完再交回主管。
- **没有它会怎样**：如果用 orchestrator，用户每说一句话都要先经过主管再转给专家，多一跳调用，专家也没法直接追问用户。
- **对应你熟悉的**：Activity 跳转。从首页 `startActivity` 进入导航页后，用户直接和导航页交互，首页不再参与。

| | 子 Agent 当工具 | Handoff | Router |
|---|---|---|---|
| 谁和用户说话 | 始终是主管 | 当前接手的 Agent | 汇总结果的那一步 |
| 能否并行 | 能 | 不适合 | 能 |
| 多轮重复请求 | 每次都要重新走一遍主管 | 状态保留，省调用 | 每次都要重新分类 |
| 典型场景 | 研究、多领域并行任务 | 客服按阶段转人工、转专家 | 知识库按来源分发 |

LangChain 文档里的一组对比：同一个请求第二次再问时，子 Agent 模式要 4 次模型调用，handoff 只要 2 次；但在需要同时查三个领域时，子 Agent 模式约 9K token，handoff 约 14K。

### 4. 并行与它的边界

- **是什么**：主管一次派出多个子 Agent 同时干活。Anthropic 的研究系统一次派 3–5 个子 Agent，每个子 Agent 再并行调用多个工具，复杂问题的耗时最多减少 90%。
- **不适合的情况**：子任务之间强依赖、需要共享大量上下文、需要实时协调。Anthropic 明确说多数编码任务属于这一类。Claude Academy 也把「复现→调试→修复」这种前后依赖的流水线列为反模式，因为信息在每次交接时都会丢失一部分。
- **还有一个瓶颈**：主管目前要等这一批子 Agent 全部完成才能继续，中途无法调整方向。改成异步能更快，但结果合并、状态一致、错误传递都会变难。

一个实用的判断标准（来自 Claude Academy）：**如果你不关心中间过程，只要结果，就委派出去；如果需要边看边调整，就留在主线程。**

## 高频面试题

**1. 什么场景适合用多 Agent？什么场景不适合？**
- 适合：可以拆成互相独立的子任务、信息量超过一个上下文窗口、工具很多且分属不同领域、任务价值足以覆盖成本。
- 不适合：子任务强依赖、需要共享大量上下文，比如大部分编码任务。
- 成本：多 Agent 约为聊天的 15 倍 token，所以先尝试单 Agent，加上工具和 Skills 能解决就不拆。

**2.（追问）子 Agent 返回的结果太简略，主管拿到「测试失败」四个字没法继续，怎么办？**
- 在委派说明里规定输出格式：必须包含具体位置、关键数据、失败原因。
- 大块结果（日志、文件）写到外部存储，只回传路径或 ID。
- 如果主管需要边看边判断（比如跑测试），就不要委派，留在主线程。

**3.（追问）orchestrator-worker 和 handoff 怎么选？**
- 需要并行、需要隔离大量中间过程 → 子 Agent 当工具。
- 需要专家直接和用户多轮对话、按阶段推进（如客服售前转售后）→ handoff。
- 两者可以组合：handoff 接手的 Agent 内部再派子 Agent。

**4.（场景题）给车机设计一个多 Agent 语音助手，你会怎么拆？**
- 主管负责意图理解和汇总；导航、媒体、车控、说明书问答各为一个子 Agent，各带自己的工具。
- 互不相关的请求（找充电站 + 放歌）并行派发；有依赖的（先查路况再改路线）留在同一个 Agent 内顺序执行。
- 车控这类高风险操作不交给子 Agent 自由发挥，由主管确认后执行；简单指令（调音量）走规则或单 Agent，不经过多 Agent。
- 子 Agent 用更小更快的模型，主管用强模型，控制成本和延迟。

## 常见误区

1. **子 Agent 知道用户说了什么**：子 Agent 从空白上下文开始，只看得到主管写的那句任务描述。主管写得含糊，子 Agent 就会做错。委派说明要把目标、格式、边界写全。
2. **拆得越细越专业**：给子 Agent 设一个「你是 Python 专家」的人设并不会增加能力（Claude Academy 列为反模式）。每多一层交接就多一次信息损失和一次模型调用，拆分要有上下文、并行或团队上的实际理由。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：把子 Agent 包装成工具，交给主管**（取自上面的 `car_supervisor.py`，假模型部分已去掉，换成真模型变量 `llm`）

```python
# 创建导航子 Agent：有自己的模型和工具
nav = create_agent(llm, tools=[search_poi])
# 创建媒体子 Agent
media = create_agent(llm, tools=[play_playlist])

# 通用委派函数：把一句任务交给某个子 Agent
def delegate(agent, request):
    # 子 Agent 从空白上下文开始，只收到这一句任务
    out = agent.invoke({"messages": [{"role": "user", "content": request}]})
    # 中间过程全部丢弃，只交回最后一条消息的文字
    return out["messages"][-1].text

# 声明为工具，主管靠函数名和文档字符串决定何时调用
@tool
def ask_nav(request: str) -> str:
    # 这句文档字符串就是给主管看的工具说明
    """导航子Agent：找地点、规划路线"""
    # 把任务转交给导航子 Agent
    return delegate(nav, request)

# 同样包装媒体子 Agent
@tool
def ask_media(request: str) -> str:
    # 工具说明：负责音乐和电台
    """媒体子Agent：音乐、电台"""
    # 把任务转交给媒体子 Agent
    return delegate(media, request)

# 创建主管：它只看得见两个子 Agent 工具
boss = create_agent(llm, tools=[ask_nav, ask_media])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `invoke` 的参数是字典 `{"messages": [...]}`，不是直接传字符串。
- 取结果是 `out["messages"][-1].text`，`-1` 表示最后一条；写成 `[0]` 拿到的是你自己发的任务。
- 工具函数必须写文档字符串（docstring），而且要放在函数体第一行。`@tool` 把它当工具说明，没有说明主管就不知道什么时候该调。

**昨日复习（只做第③步）**：Day 15 的「酒店 Agent 的任务生命周期」（`HotelExecutor`）。只看你自己写过的注释，重写代码，重点核对：续接任务时不能再 `enqueue_event(task)`；`TaskUpdater` 三个参数的顺序；`requires_input` 之后要 `return`。

## 动手练习（30 分钟）

在 `car_supervisor.py` 的基础上改：

1. 新增一个「车控子 Agent」`ask_climate`，带一个工具 `set_ac(temp: int)`（内部 `time.sleep(1)`）。让主管一次并行调用三个子 Agent，确认总耗时仍约为 1 秒。
2. 把 `boss` 换成顺序调用：剧本里拆成三条 `AI` 消息，每条只带一个 tool call。对比总耗时，用一句话写下「并行」到底省在哪里。
3. 思考题：用户接着说「就去 B 站吧」，主管把这句话原样转给导航子 Agent，会发生什么？（提示：子 Agent 是无状态的，它不记得上一轮的列表。）你会怎么改委派说明？

## 读后自测

1. 什么时候该把一个 Agent 拆成多个？拆了要付出什么代价？→ 见第 6 节「核心概念 1」
2. 「子 Agent 当工具调用」和「handoff」有什么区别，各适合什么场景？→ 见第 6 节「核心概念 2、3」
3. 为什么说多数编码任务、强依赖的流水线任务不适合用多 Agent？→ 见第 6 节「核心概念 4」

---

## 参考区（用时查，不必通读）

### 细节速查

- **LangChain 五种多 Agent 模式**：Subagents（子 Agent 当工具）、Handoffs（转交）、Skills（单 Agent 按需加载专业 prompt）、Router（先分类再分发，可多路后汇总）、Custom workflow（用 LangGraph 自定义，可以嵌入以上任意模式）。
- **LangChain 文档里的调用次数对比**：

| 场景 | Subagents | Handoffs | Skills | Router |
|---|---|---|---|---|
| 单次请求（模型调用次数） | 4 | 3 | 3 | 3 |
| 同一请求问两轮（总调用） | 8 | 5 | 5 | 6 |
| 三个领域对比（调用 / token） | 5 / ~9K | 7+ / ~14K+ | 3 / ~15K | 5 / ~9K |

- **Handoff 的写法**：转交工具返回 `Command(update={"messages": [ToolMessage(...)], "active_agent": "xxx"})`（`from langgraph.types import Command`）。必须带一条对应 `tool_call_id` 的 `ToolMessage`，否则消息历史不合法；在子图间转交用 `Command(goto=..., graph=Command.PARENT)`，并同时传带工具调用的 `AIMessage` 和 `ToolMessage` 这一对；不要把子 Agent 的完整历史转过去。
- **Anthropic 工作流模式**（Building effective agents）：提示链、路由、并行（分片 sectioning / 投票 voting）、orchestrator-workers、evaluator-optimizer（一个生成、一个评审，循环改进）。课程表里的「辩论」可以理解为投票 + 评审的组合：多个 Agent 独立给出答案，再由裁判或互评收敛。
- **Anthropic 研究系统的数据**：Opus 4 主管 + Sonnet 4 子 Agent，比单个 Opus 4 在内部研究评测上高 90.2%；BrowseComp 上 token 用量解释了约 80% 的表现差异。按复杂度分配资源：简单查事实 1 个 Agent、3–10 次工具调用；对比类 2–4 个子 Agent；复杂研究 10 个以上。另有一个 CitationAgent 专门给结论补引用。
- **Claude Code 子 Agent**：每个子 Agent 有独立上下文、自定义 system prompt 和工具权限，只把摘要返回主对话；内置 General purpose、Explore、Plan 三种。适合调研、代码审查（独立上下文更客观）；不适合跑测试（摘要会丢细节，课程实测效果最差）。
- **剧本模型**：`GenericFakeChatModel` 默认不支持 `bind_tools`，本篇通过子类把它改成直接返回自身，仅用于演示和测试。

### 来源说明

本篇未使用非 LangChain/Claude 来源。LangChain 的子 Agent 页面和 benchmark 博客正文本次没能打开，子 Agent 的写法以「Build a personal assistant with subagents」教程为准。

### 延伸学习

- [Introduction to subagents](https://academy.claude.com/courses/introduction-to-subagents)（Claude Academy，4 课）
- [Build a personal assistant with subagents](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents-personal-assistant)（LangChain 教程）
- Day 17 LangGraph 会讲 handoff 背后的状态图与 checkpoint。

### 参考资料

- [Multi-agent - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent)
- [Handoffs - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs)
- [Build a personal assistant with subagents - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents-personal-assistant)
- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Building Effective AI Agents - Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- [What are subagents? - Claude Academy](https://academy.claude.com/courses/introduction-to-subagents/what-are-subagents)
- [Using subagents effectively - Claude Academy](https://academy.claude.com/courses/introduction-to-subagents/using-subagents-effectively)
