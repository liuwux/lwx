# Day 12 · Agentic RAG（第二轮：多跳要串行、检索要有预算、「没找到」要说出来）（2026-11-09）

## 今日 AI 动态（2026-11-09）

过去 24 小时关注的厂商没有新的官方发布。

---

> 第二轮说明：第一轮 Day 12 讲过三种 RAG 架构、「检索作为工具」的最小改动、多跳检索的概念、何时检索、just-in-time 检索和 `search_result` 引用块。今天不再重复这些，只讲把检索交给 Agent 之后才会出现的问题：多跳之间的依赖、检索停不下来、工具怎么告诉模型「没有」。

## 读前三问

1. 用户问「我这台车该加什么机油」，Agent 需要先查车型、再查手册。这两次检索能不能并行发出？什么情况下可以并行？
2. 知识库里根本没有答案时，Agentic RAG 最常见的失败是什么？在 LangChain 里怎么给它设上限？
3. 检索工具找不到内容时，应该返回空字符串、抛异常，还是返回别的？为什么？

## 从一个场景开始

上一轮你把车机「用车手册助手」从固定流程的 2-step RAG 改成了 Agentic RAG：模型自己决定什么时候调用 `search_manual`。

演示很顺利，上线后出了两件事。

第一件：司机问「我这台车该加什么机油？」，助手直接回答「5W-30」。可这台车是混动版，手册写的是 0W-20。模型没先查车型，用一个笼统的问题检索，拿到了燃油版的内容。

第二件：司机问「拖车钩怎么装？」，手册里没有这部分。日志显示助手换着说法搜了 11 次，语音一直卡在「正在查询」，最后因为超时被系统打断。

把检索交给模型以后，「查什么、查几次、什么时候停」都变成了模型的决定。今天讲怎么给这些决定加约束。

## 一句话总结

Agentic RAG 让模型自己决定检索，但多跳之间有依赖要串行，检索次数要设上限，工具找不到时要明确说「没有」，模型才能及时停下来走兜底。

## 先跑起来

用 LangChain 的 `create_agent` 搭一个有两个工具的 Agent，配上 `ToolCallLimitMiddleware` 限制检索次数。模型是按剧本输出的假模型，用来稳定复现「两跳检索」和「停不下来」两种轨迹。

```python
# agentic_hops.py —— 两跳检索要串行、检索要有预算（脚本化假模型，无需 API Key）
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

MANUAL = {"A7 混动 机油": "A7 混动版：0W-20 全合成，容量 4.2 L", "座椅加热": "长按座椅图标 3 秒"}

@tool
def lookup_vehicle(vin: str) -> str:
    """按 VIN 查车型。用户说“我这台车”时先调用它。"""
    return "A7 混动" if vin.startswith("LA7") else "未知车型"

@tool
def search_manual(query: str) -> str:
    """检索用车手册。query 要包含车型名，例如“A7 混动 机油”。"""
    hits = [v for k, v in MANUAL.items() if all(w in query for w in k.split())]
    return hits[0] if hits else "NO_RESULT：手册中没有相关内容"

class ScriptedModel(GenericFakeChatModel):        # 假模型：按剧本依次输出，忽略 bind_tools
    def bind_tools(self, tools, **kw):
        return self

call = lambda name, **a: AIMessage("", tool_calls=[{"name": name, "args": a, "id": str(id(a))}])

script = iter([
    call("lookup_vehicle", vin="LA7X2026"),        # 第 1 跳：先查车型
    call("search_manual", query="A7 混动 机油"),     # 第 2 跳：查询词用到了第 1 跳的结果
    AIMessage("您的 A7 混动用 0W-20 全合成机油，约 4.2 L。"),
    call("search_manual", query="拖车钩 安装"),      # 库里没有的问题：模型不停换词重搜
    call("search_manual", query="A7 拖车钩"),
    call("search_manual", query="A7 混动 拖挂 安装"),
    AIMessage("手册里没有拖车钩安装说明，建议联系服务站。"),
])
agent = create_agent(ScriptedModel(messages=script), tools=[lookup_vehicle, search_manual],
                     middleware=[ToolCallLimitMiddleware(tool_name="search_manual", run_limit=2)])

for q in ["我这台车（VIN LA7X2026）该加什么机油？", "拖车钩怎么装？"]:
    print(f"\n== {q}")
    for m in agent.invoke({"messages": [("human", q)]})["messages"][1:]:
        tc = getattr(m, "tool_calls", None)
        print("  调用", tc[0]["name"], tc[0]["args"]) if tc else print("  结果" if m.type == "tool" else "  回答", m.content)
```

**运行**：`pip install -U langchain langgraph && python agentic_hops.py`

**实测输出**（2026-11-09，Python 3.13，langchain 1.4.9，langchain-core 1.7.6，langgraph 1.2.20）：

```
== 我这台车（VIN LA7X2026）该加什么机油？
  调用 lookup_vehicle {'vin': 'LA7X2026'}
  结果 A7 混动
  调用 search_manual {'query': 'A7 混动 机油'}
  结果 A7 混动版：0W-20 全合成，容量 4.2 L
  回答 您的 A7 混动用 0W-20 全合成机油，约 4.2 L。

== 拖车钩怎么装？
  调用 search_manual {'query': '拖车钩 安装'}
  结果 NO_RESULT：手册中没有相关内容
  调用 search_manual {'query': 'A7 拖车钩'}
  结果 NO_RESULT：手册中没有相关内容
  调用 search_manual {'query': 'A7 混动 拖挂 安装'}
  结果 Tool call limit exceeded. Do not call 'search_manual' again.
  回答 手册里没有拖车钩安装说明，建议联系服务站。
```

**该看什么**：
- 第一段：第二次检索的查询词「A7 混动 机油」里的「A7 混动」来自第一次工具的结果。第二跳必须等第一跳回来才能发出。
- 第二段：第三次检索没有真正执行，中间件直接返回了一条错误消息作为工具结果。模型读到它，才改成兜底回答。
- 工具找不到时返回的是带 `NO_RESULT` 前缀的明确文字，而不是空字符串。

## 核心概念

### 1. 多跳检索：有依赖就串行，无依赖才并行

- **是什么**：多跳是指答案需要好几次检索拼起来。如果后一次的查询词要用到前一次的结果（先查车型，再按车型查机油），就只能一次一次来；如果几个子问题互不相关（「前排和后排座椅加热分别怎么开」），可以在同一轮里并行发出多个工具调用。
- **没有它会怎样**：模型跳过第一跳，用笼统问题直接检索，拿到另一个车型的内容，就是场景里的「5W-30」。反过来，把独立子问题一个个串行查，语音场景的等待时间会成倍增加。
- **对应你熟悉的**：协程里 `async/await` 的用法。有依赖的请求按顺序 `await`；互不依赖的用 `async` 并发发出，再一起 `awaitAll`。

在工具描述里写清楚调用顺序（如「用户说『我这台车』时先调用它」），比在系统提示词里写一大段规则更有效，因为模型选工具时正好在读这段描述。Anthropic 的研究系统里，子代理并行调用 3 个以上工具，复杂问题的研究时间最多缩短了 90%。

### 2. 检索预算：给「查几次」设上限

- **是什么**：Agentic RAG 的循环是「想一想 → 检索 → 再想一想」，什么时候停由模型决定。预算就是从外部加一个硬上限，比如单个问题最多检索 2 次。
- **没有它会怎样**：知识库里没有答案时，模型倾向于换着说法反复搜，越搜越久。Anthropic 介绍过他们早期的研究 Agent：为一个简单问题启动了 50 个子代理，还会不停搜索根本不存在的资料。
- **对应你熟悉的**：OkHttp 的重试和超时。重试必须有最大次数，否则弱网下请求会一直挂着。

LangChain 里用 `ToolCallLimitMiddleware` 实现。两个关键参数：
- `run_limit`：一次提问内最多调用几次，下一个问题重新计数；
- `thread_limit`：整段对话累计上限，需要配 checkpointer 才能跨轮记住次数。

默认行为是 `continue`：超出的调用被拦下，返回一条错误消息给模型，模型自己决定怎么收尾（输出里的第三次调用就是这样）。上限多少合适，Anthropic 给过一个参考：简单的事实查询，1 个 Agent、3 到 10 次工具调用就够了。车机语音要更紧。

### 3. 「没找到」要明确说出来

- **是什么**：检索工具找不到内容时，返回一句模型能看懂的话，比如「NO_RESULT：手册中没有相关内容」，必要时附上建议（「试试只用车型名检索」）。
- **没有它会怎样**：返回空字符串，模型会以为是工具出错了，于是继续重试；直接抛异常，整个 Agent 可能中断。两种情况都会让模型停不下来，或者干脆用自己的记忆编一个答案。
- **对应你熟悉的**：接口返回 `200 + 空列表` 和 `404 + 错误说明` 的区别。客户端能根据明确的状态决定显示「暂无数据」，而不是一直转圈。

这一条和第 2 条是配合的：明确的「没有」让模型更早停下来，预算是它没停时的兜底。上限触发时返回的也是一条文字消息，所以最终回答仍由模型生成，语气可以保持一致。

### 4. 查询先宽后窄

- **是什么**：第一次检索用短而宽的查询，看看知识库里有什么，再根据结果逐步收窄。
- **没有它会怎样**：Anthropic 发现模型常常一上来就写很长、很具体的查询，结果命中很少，接着又换一个同样具体的说法，白白消耗次数。
- **对应你熟悉的**：在 Logcat 里排查问题，先按 TAG 过滤看大概，再加关键字缩小范围。

可以把这条写进检索工具的描述，或者在 `NO_RESULT` 的提示里引导模型「换更短的关键词」。

## 高频面试题

**Q1：Agentic RAG 和固定流程的 2-step RAG 各适合什么场景？**
- 2-step 每次必检索一次，延迟稳定、可控，适合 FAQ、文档问答。
- Agentic 由模型决定是否检索、检索几次，灵活，但延迟不可预测。
- 中间方案是在固定流程里加查询改写、结果评审等校验步骤，LangChain 文档称为 Hybrid RAG。
- 车机语音对延迟敏感，常见做法是高频问题走固定流程，复杂问题再交给 Agent。

**Q2（追问）：交给 Agent 之后，你怎么防止它一直检索停不下来？**
- 工具找不到时返回明确的「没有」信号和下一步建议，让模型主动停。
- 外部加硬上限，比如 LangChain 的 `ToolCallLimitMiddleware`，按单次提问或整段对话计数。
- 再给模型调用次数也设上限，防止「不调工具、只空转」。
- 监控每个问题的检索次数分布，次数异常多往往意味着知识库缺内容。

**Q3（场景题）：用户问「我这台车该加什么机油」，Agent 却回答了另一个车型的机油，怎么排查和修？**
- 看轨迹：是否先调用了查车型的工具，第二次检索的查询词里有没有车型名。
- 在查车型工具的描述里写清调用时机，在检索工具的描述里要求查询包含车型。
- 也可以在工具内部按车型过滤，不让模型漏传。
- 评测集里加入依赖车型的问题，检查轨迹而不只是最终答案。

**Q4（追问）：多跳检索能不能都并行发出，加快速度？**
- 不能都并行。后一跳的查询依赖前一跳的结果时，只能串行。
- 互不依赖的子问题可以在同一轮里并行调用多个工具。
- 可以先让模型列出子问题和依赖关系，再决定哪些一起发。
- 并行会让上下文一次涌入很多结果，要控制每次返回的长度。

## 常见误区

- **「设了调用上限就不用管工具返回了」**：上限只是兜底。工具不明确说「没有」，模型每次都会把预算用完，延迟和费用照样高。
- **「多跳检索的效果只看最终答案对不对」**：最终答案对了，轨迹可能是凑巧的（比如碰巧检索到了正确车型）。评测要同时看工具调用的顺序和参数。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：两个检索工具 + 检索预算**（取自 `agentic_hops.py`）

```python
# 把函数注册成工具，函数说明会作为工具描述交给模型
@tool
# 查车型工具：输入 VIN，返回车型名
def lookup_vehicle(vin: str) -> str:
    # 工具描述：写清楚什么时候该先调用它
    """按 VIN 查车型。用户说“我这台车”时先调用它。"""
    # 演示逻辑：VIN 以 LA7 开头就是 A7 混动
    return "A7 混动" if vin.startswith("LA7") else "未知车型"

# 再注册一个工具
@tool
# 手册检索工具：输入查询词，返回命中的一段
def search_manual(query: str) -> str:
    # 工具描述：要求查询里带上车型名
    """检索用车手册。query 要包含车型名，例如“A7 混动 机油”。"""
    # 某个条目的所有关键词都出现在查询里，才算命中
    hits = [v for k, v in MANUAL.items() if all(w in query for w in k.split())]
    # 有命中返回第一条；没有就明确说“没有”，不要返回空串
    return hits[0] if hits else "NO_RESULT：手册中没有相关内容"

# 创建 Agent：传入模型、工具列表
agent = create_agent(ScriptedModel(messages=script), tools=[lookup_vehicle, search_manual],
                     # 中间件：单次提问里，手册检索最多调用 2 次
                     middleware=[ToolCallLimitMiddleware(tool_name="search_manual", run_limit=2)])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `@tool` 读的是函数的 docstring 当工具描述，docstring 必须写在函数体第一行，写成注释就没用了。
- `all(w in query for w in k.split())` 是「条目的关键词都在查询里」，方向别写反成 `query in k`。
- `middleware=` 接的是列表；`tool_name` 是字符串，要和函数名完全一致。

**昨日复习（只做第③步）**：第二轮 Day 11 的「混合检索 + 追问改写链」（`hybrid_followup.py`）。只看你自己写的注释重写，重点核对：向量检索的条数在 `as_retriever(search_kwargs={"k": 2})` 里，BM25 直接写 `k=2`；`EnsembleRetriever` 的 `weights` 个数和检索器一致；改写链的输入是字典 `{"q": ...}`。

## 动手练习（30 分钟）

在 `agentic_hops.py` 的基础上：

1. 把 `run_limit` 改成 1，再加一个 `ToolCallLimitMiddleware(run_limit=3)`（不填 `tool_name`，限制所有工具合计）。重新排剧本，观察第一段的两跳是否还能完成。
2. 把 `search_manual` 找不到时的返回值改成空字符串 `""`，看看输出里「结果」一行变成什么样，想一想真实模型读到它会怎么做。
3. 在剧本里加一轮：同一个 `AIMessage` 里同时发出两个 `search_manual` 调用（「前排座椅加热」「后排座椅加热」），观察 `create_agent` 是否两个都执行了，结果顺序是什么。

## 读后自测

1. 用户问「我这台车该加什么机油」，Agent 需要先查车型、再查手册。这两次检索能不能并行发出？什么情况下可以并行？——见第 1 节和 Q4
2. 知识库里根本没有答案时，Agentic RAG 最常见的失败是什么？在 LangChain 里怎么给它设上限？——见第 2 节和 Q2
3. 检索工具找不到内容时，应该返回空字符串、抛异常，还是返回别的？为什么？——见第 3 节

## AI 历史知识点：多智能体研究系统 Claude Research（2025，Anthropic）

**是什么**：Anthropic 公开的一篇工程文章，讲 Claude「Research」功能背后的架构：一个主 Agent 负责规划，按需启动多个子 Agent 并行检索，各自在独立的上下文里搜索、筛选，再把精简后的结果交回主 Agent 汇总。

**何时由谁提出**：2025 年 6 月 13 日，由 Anthropic 工程团队的 Jeremy Hadfield、Barry Zhang、Kenneth Lien 等人发表在 Anthropic Engineering 博客。

**为什么重要**：它给了 Agentic 检索少有的一手数据。用 Claude Opus 4 做主 Agent、Sonnet 4 做子 Agent 的多智能体系统，在内部研究评测上比单个 Opus 4 高 90.2%；在 BrowseComp 评测上，token 用量一项就解释了约 80% 的成绩差异。代价也写得很清楚：多智能体系统消耗的 token 大约是普通对话的 15 倍。文章还总结了检索的经验，比如「先宽后窄」的查询策略、按问题难度规定检索次数，以及早期版本「为简单问题启动 50 个子代理」的过度检索问题。

**和今天的趋势有什么联系**：今天讲的多跳依赖、并行检索、检索预算，在这篇文章里都有对应的做法。业界的「深度研究」类产品基本都沿用了「主 Agent 规划 + 子 Agent 并行检索 + 预算约束」这一套结构。

**原始来源**：
- [How we built our multi-agent research system - Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `ToolCallLimitMiddleware` | `langchain.agents.middleware`；参数 `tool_name`（不填则所有工具合计）、`thread_limit`（整段对话，需 checkpointer）、`run_limit`（单次提问，下次提问重置），两者至少填一个；`exit_behavior`：`continue`（默认，拦下超出的调用并给模型返回错误消息，模型决定何时结束）、`error`（抛 `ToolCallLimitExceededError`）、`end`（立即结束，只能用于限制单个工具，其他工具还有待执行调用时报 `NotImplementedError`）；可同时挂多个实例，分别做全局和单工具限制 |
| `ModelCallLimitMiddleware` | 限制模型调用次数，`thread_limit` / `run_limit`，`exit_behavior` 默认 `end`（正常结束），可选 `error` |
| 实测拦截消息 | `Tool call limit exceeded. Do not call 'search_manual' again.`（以 ToolMessage 形式返回） |
| LangChain 三种 RAG 架构 | 2-Step（控制高、灵活低、延迟稳定，FAQ / 文档机器人）；Agentic（控制低、灵活高、延迟不定，多工具研究助手）；Hybrid（中 / 中，带查询改写、检索校验、答案校验，可回环） |
| Anthropic 检索投入参考 | 简单事实查询：1 个 Agent、3–10 次工具调用；直接比较：2–4 个子 Agent、各 10–15 次；复杂研究：10 个以上子 Agent |
| Anthropic 研究系统数据 | 多智能体比单 Agent Opus 4 高 90.2%（内部评测）；BrowseComp 上 token 用量解释约 80% 差异，加上工具调用次数和模型选择共约 95%；Agent 约为对话 4 倍 token，多智能体约 15 倍；并行把复杂问题研究时间最多缩短 90%；上下文超 20 万 token 会被截断，主 Agent 先把计划存入记忆 |
| 假模型 | `GenericFakeChatModel(messages=iterator)` 按顺序输出 `AIMessage`，可带 `tool_calls`；它本身不实现 `bind_tools`，`create_agent` 里使用要像示例那样子类化返回 `self` |

### 来源说明

- 演示用的模型是按剧本输出的假模型，只用来展示中间件和工具返回值的行为；换成 `ChatAnthropic` 等真实模型时，是否先查车型、何时停止由模型决定，本次没有 API Key，真实模型的轨迹未实测。
- 今日 AI 动态：各家官方页面在时间范围内（2026-11-08 美国时间 / 11-08 至 11-09 北京时间）未找到新发布；openai.com 新闻页本次无法打开，改查了 OpenAI 开发者更新日志，最近一条为 11-05。

### 延伸学习

- LangChain「Build a custom RAG agent with LangGraph」教程：检索、评审、改写的完整图（第二轮 Day 11 已介绍）。
- Claude Academy「Building with the Claude API」课程中的 RAG 与工具使用章节。

### 参考资料

- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Retrieval - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/retrieval)
- [How we built our multi-agent research system - Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
