# Day 28 · 端侧与移动端 Agent（2026-10-26）

## 今日 AI 动态（2026-10-26）

过去 24 小时关注的厂商没有新的官方发布。

---

## 读前三问

1. 怎么把一个 Android App 的能力（如调空调、开导航）变成 Agent 能调用的工具？
2. 车载语音 Agent 的「端云协同」怎么设计？什么走端侧、什么走云端？
3. 断网、弱网、模型选错工具时，车上的 Agent 怎么兜底？

## 从一个场景开始

你负责车机语音助手。现在的做法是：每一句话都上传云端大模型，模型决定调哪个功能，再下发指令。

上线后问题来了：

- 车开进地库，网断了，车主说「有点冷」，助手毫无反应。
- 网络正常时，「打开车窗」这种一句话指令也要等 1.5 秒，车主觉得比按按钮还慢。
- 每个 App 团队都要单独给云端写一套接口，导航、空调、音乐各写各的。

你需要三样东西：**让 App 自己把能力声明成工具**（不再每家写接口），**把简单高频的指令留在车机本地处理**，**网断了能降级而不是罢工**。这就是今天的主题：端侧与移动端 Agent。

## 一句话总结

把 App 能力声明成带描述的工具让 Agent 发现和调用；短而高频的指令交给端侧小模型，复杂请求交给云端大模型；断网时降级到端侧，危险操作仍需确认。

## 先跑起来

用 LangChain 的 `create_agent` 搭一个车载 Agent：两个「App 能力」作为工具，一个中间件按「指令简不简单、在不在线」把请求路由给端侧或云端模型。为了不依赖 API Key 和本地模型，两个模型都换成按剧本出牌的假模型，只看路由和工具调用流程。

```python
# car_edge_agent.py —— 车载语音 Agent：App 能力当工具 + 端云路由（剧本模型代替真实 LLM）
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage, ToolMessage
from langchain_core.outputs import ChatGeneration, ChatResult

@tool
def set_ac(temp: int) -> str:
    """把车内空调设到 temp 摄氏度，范围 16-30。"""
    return f"空调已设为 {temp}℃" if 16 <= temp <= 30 else "错误：温度需在 16-30 之间"
@tool
def navigate(dest: str) -> str:
    """开始导航到 dest。"""
    return f"已开始导航到{dest}"
class Scripted(GenericFakeChatModel):  # 剧本模型：先调一次工具，拿到结果后回复
    tag: str
    def bind_tools(self, tools, **kw): return self
    def _generate(self, messages, stop=None, run_manager=None, **kw):
        last = messages[-1]
        if isinstance(last, ToolMessage): msg = AIMessage(f"[{self.tag}] {last.content}")
        elif "冷" in last.content: msg = AIMessage("", tool_calls=[{"name": "set_ac", "args": {"temp": 24}, "id": "c1"}])
        else: msg = AIMessage("", tool_calls=[{"name": "navigate", "args": {"dest": "顺路充电站"}, "id": "c2"}])
        return ChatResult(generations=[ChatGeneration(message=msg)])
local, cloud = Scripted(messages=iter([]), tag="端侧小模型"), Scripted(messages=iter([]), tag="云端大模型")
SIMPLE = ("冷", "热", "打开", "关闭", "音量")

@dataclass
class Ctx:
    online: bool = True

@wrap_model_call
def edge_cloud_router(request, handler):
    q = request.messages[0].content
    simple = len(q) <= 8 and any(k in q for k in SIMPLE)   # 短、且命中高频指令词
    use_local = simple or not request.runtime.context.online  # 离线时一律走端侧
    print("  路由 ->", "端侧" if simple else ("端侧（离线降级）" if use_local else "云端"))
    return handler(request.override(model=local if use_local else cloud))
agent = create_agent(model=cloud, tools=[set_ac, navigate], middleware=[edge_cloud_router], context_schema=Ctx)
for q, online in [("有点冷", True), ("先送孩子再找个顺路的充电站", True), ("先送孩子再找个顺路的充电站", False)]:
    print(f"用户：{q}（在线={online}）")
    out = agent.invoke({"messages": [("user", q)]}, context=Ctx(online=online))
    print("  回复：", out["messages"][-1].content)
```

**运行**：`pip install -U langchain langgraph && python car_edge_agent.py`

**实测输出**（2026-10-26，Python 3.13，langchain 1.4.7）：

```
用户：有点冷（在线=True）
  路由 -> 端侧
  路由 -> 端侧
  回复： [端侧小模型] 空调已设为 24℃
用户：先送孩子再找个顺路的充电站（在线=True）
  路由 -> 云端
  路由 -> 云端
  回复： [云端大模型] 已开始导航到顺路充电站
用户：先送孩子再找个顺路的充电站（在线=False）
  路由 -> 端侧（离线降级）
  路由 -> 端侧（离线降级）
  回复： [端侧小模型] 已开始导航到顺路充电站
```

**该看什么**：
- 每个问题都打印了**两次**路由：第一次模型决定调工具，第二次模型读工具结果再回复。中间件包住的是**每一次模型调用**，不是每个问题。
- 同一句复杂请求，在线走云端，离线降级到端侧，工具调用流程完全一样。**换模型不换工具**，是端云协同的前提。
- 工具本身（`set_ac`）始终在车机上执行，模型只负责「说要调什么」。

## 核心概念

### 1. App 能力即工具

- **是什么**：App 把自己能做的事（设温度、开导航）声明成「名字 + 描述 + 参数」，Agent 读描述后决定调哪个，由 App 自己执行。
- **没有它会怎样**：每个 App 都要给语音助手单独开一套接口，助手每接一个新 App 都要改代码；或者只能靠模拟点击屏幕，又慢又脆。
- **对应你熟悉的**：像 `ContentProvider` / AIDL 把能力暴露给别的进程，只是调用方从另一个 App 变成了 Agent。

Android 16 起官方提供了 **AppFunctions**：给 `AppFunctionService` 里的方法加 `@AppFunction` 注解，KDoc 自动成为 Agent 读到的描述，调用方需要 `EXECUTE_APP_FUNCTIONS` 权限。官方文档直接称它为「设备上的 MCP 工具」。今天代码里 `@tool` + docstring 就是同一个思路的 Python 版。

### 2. 工具在端侧执行，模型可以在任何地方

- **是什么**：Claude 文档把工具分为两类：**客户端工具**由你的程序执行（模型返回 `tool_use`，你执行后回传 `tool_result`）；**服务端工具**在 Anthropic 那边执行，比如网页搜索。
- **没有它会怎样**：会误以为「用了云端模型，车就被云端控制了」，安全边界画错。
- **对应你熟悉的**：服务端下发「该做什么」的指令，真正操作硬件的永远是车机上的 Service。

所以车控类工具一定是客户端工具：参数校验、权限检查、确认弹窗都在车机代码里做。官方课程也强调：工具要做参数校验，并返回**清楚的错误信息**（如「温度需在 16-30 之间」），模型读到后会自己修正重试。

### 3. 端云协同：按难度和网络路由

- **是什么**：短、高频、低风险的指令（调温度、音量）交给车机上的小模型或规则；需要推理、多步规划、联网查询的请求交给云端大模型。
- **没有它会怎样**：全上云，延迟高、断网不可用、费用随调用量线性涨；全在端侧，复杂请求答不好。
- **对应你熟悉的**：图片加载的「内存缓存 → 磁盘缓存 → 网络」，能在近处解决就不走远处。

LangChain 里用 `@wrap_model_call` 中间件实现，核心就一行 `handler(request.override(model=...))`。Claude 文档的建议方向一致：延迟敏感、量大的简单任务优先用最快的模型，复杂任务用更强的模型，也可以组合使用让大部分 token 走便宜模型。

### 4. 离线降级与确认闸门

- **是什么**：网络不可用时，路由强制落到端侧，能力打折但不罢工；涉及安全的动作（开车门、行驶中改设置）无论端云都要二次确认或直接拒绝。
- **没有它会怎样**：地库、隧道里助手失灵；或者端侧小模型理解错了，直接执行了危险操作。
- **对应你熟悉的**：App 的离线模式 + 运行时权限弹窗。

降级时要告诉用户「当前离线，只支持基础指令」，复杂请求可以先排队，联网后再交给云端。

## 高频面试题

**1. 怎么让车机上的多个 App 被同一个语音 Agent 调用？**
- 每个 App 把能力声明成工具：名字、描述、参数 schema，由 App 自己执行。
- Android 16+ 可用 AppFunctions；跨平台可用 MCP；自研系统可定义统一的工具注册表。
- 描述写清用途和取值范围，Agent 靠描述选工具。
- 调用方要有权限，敏感能力单独授权。

**2.（追问）端侧小模型选错了工具怎么办？**
- 工具内做参数校验，错误信息写清楚，让模型能自我修正。
- 端侧置信度低或连续失败时升级到云端（在线时）。
- 高风险工具加确认闸门，不靠模型自觉。
- 记录日志，作为后续优化端侧模型的数据。

**3.（场景题）设计一个车载语音 Agent 的端云协同方案。**
- 第一层：本地规则/关键词，处理最高频的固定指令，几十毫秒。
- 第二层：端侧小模型，处理短指令和离线场景。
- 第三层：云端大模型，处理多步规划、联网查询，流式返回降低等待感。
- 统一工具层在车机执行；断网降级、危险操作确认、全链路日志。

**4. 为什么不全部放在端侧，或者全部放在云端？**
- 全端侧：车机算力和内存有限，小模型复杂推理差，模型更新慢。
- 全云端：延迟高、断网不可用、调用成本随用户量增长、隐私数据要出车。
- 端云结合：高频简单留本地，难题上云。
- 路由策略要用评测数据调，不能拍脑袋定阈值。

## 常见误区

1. **「云端模型在控制车」**：模型只返回要调用哪个工具，执行在车机代码里，权限和安全责任也在车机这边。
2. **「端侧就是把大模型裁小塞进车机」**：端侧的价值在于低延迟、离线可用和隐私，它负责的是「简单且高频」那部分，而不是替代云端。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带校验的工具 + 端云路由中间件**（取自 `car_edge_agent.py`）

```python
# 用 @tool 把函数声明成 Agent 可调用的工具
@tool
# 定义设空调温度的工具，参数是整数温度，返回字符串
def set_ac(temp: int) -> str:
    # docstring 就是给模型看的工具描述，写清取值范围
    """把车内空调设到 temp 摄氏度，范围 16-30。"""
    # 在范围内就返回成功，否则返回清楚的错误信息
    return f"空调已设为 {temp}℃" if 16 <= temp <= 30 else "错误：温度需在 16-30 之间"

# 定义运行时上下文：记录当前是否在线
@dataclass
class Ctx:
    # 默认在线
    online: bool = True

# 用 wrap_model_call 包住每一次模型调用
@wrap_model_call
# 中间件函数接收请求和真正执行调用的 handler
def edge_cloud_router(request, handler):
    # 取第一条消息，即用户原话
    q = request.messages[0].content
    # 判断是否简单指令：长度不超过 8 且命中高频词
    simple = len(q) <= 8 and any(k in q for k in SIMPLE)
    # 简单指令或离线时走端侧
    use_local = simple or not request.runtime.context.online
    # 复制请求、换上选中的模型，再交给 handler 执行
    return handler(request.override(model=local if use_local else cloud))
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 在线标志在 `request.runtime.context.online`，不是 `request.state`；而且 `create_agent` 要传 `context_schema=Ctx`，调用时传 `context=Ctx(...)`。
- 最后一定是 `return handler(...)`，忘了 `return` 中间件就吞掉了模型结果。
- `request.override(model=...)` 返回的是新请求，不会修改原来的 `request`。

**昨日复习（只做第③步）**：Day 27 的「从日志造 SFT / DPO 数据 + DPO 损失」。只看你自己写的注释重写，重点核对：`margin` 里两个括号都是「策略减参考」、sigmoid 的负号在 `exp` 里、DPO 推导式要有 `r["q"] in ups` 过滤。

## 动手练习（30 分钟）

在 `car_edge_agent.py` 上改：

1. 加一个工具 `open_window(percent: int)`，`percent` 超出 0-100 时返回错误信息。
2. 给 `Ctx` 加字段 `driving: bool`。在中间件里：如果正在行驶且用户原话含「车门」，不调用模型，直接返回 `AIMessage("行驶中不能开车门")`（提示：中间件可以不调用 `handler`，直接 `return ModelResponse(result=[AIMessage(...)])`，`ModelResponse` 从 `langchain.agents.middleware` 导入）。
3. 统计一次运行中端侧和云端各被调用了几次，打印出来。

## 读后自测

1. 怎么把 App 能力变成 Agent 能调用的工具？→ 见核心概念 1、2，面试题 1
2. 端云协同怎么设计？→ 见先跑起来、核心概念 3，面试题 3、4
3. 断网、选错工具怎么兜底？→ 见核心概念 4，面试题 2

---

## AI 历史知识点：Claude 3 模型家族（2024，Anthropic）

**是什么**：Anthropic 一次发布三个档位的模型：Haiku（最快最便宜）、Sonnet（速度与能力平衡）、Opus（最强）。三者都支持 200K token 上下文和图像输入。

**何时由谁提出**：2024 年 3 月 4 日，Anthropic 在官方博客《Introducing the next generation of Claude》中发布。

**为什么重要**：它让「按任务选档位」成为行业惯例。官方给出的定位很具体：Haiku 读一篇约 1 万 token、带图表的 arXiv 论文不到 3 秒，价格是 Opus 的约六十分之一（输入 $0.25 对 $15 / 百万 token），适合实时客服和内容审核；Opus 用于研究、策略和复杂任务自动化。三档同时支持视觉，也让「看图表、看截图」成为通用能力。

**和今天的趋势有什么联系**：今天端云协同的核心思想就是「便宜快速的模型处理大量简单请求，强模型只处理难题」。从 Claude 3 的三档到现在官方文档建议的「先用最快模型，有能力缺口再升级」「多模型组合」，再到车机上的端侧小模型 + 云端大模型，是同一条路线的延伸。

原始来源：[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family)

---

## 参考区（用时查，不必通读）

### 细节速查

**LangChain 动态选模型（当前写法）**
- 导入：`from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse`
- 签名：`def mw(request: ModelRequest, handler: Callable[[ModelRequest], ModelResponse]) -> ModelResponse`
- 读消息：`request.messages`；读状态：`request.state["messages"]`；读上下文：`request.runtime.context`
- 换模型：`handler(request.override(model=m))`，返回副本，不改原请求
- 注意：已经 `bind_tools` 过的模型配合结构化输出时不支持，要传未绑定的模型
- 本地模型：`pip install langchain-ollama`，`ChatOllama(model="gpt-oss:20b", temperature=0).bind_tools([...])`，需先 `ollama pull`，且模型要支持工具调用

**Android AppFunctions（非 LangChain/Claude 来源）**
- 要求 Android 16（API 36）+，实验性 API；Jetpack 库 `androidx.appfunctions`
- 服务：继承 `AppFunctionService`，加 `@AppFunctionServiceEntryPoint(serviceName=..., appFunctionXmlFileName=...)`
- 函数：`@AppFunction(isDescribedByKDoc = true)`；数据类：`@AppFunctionSerializable(isDescribedByKDoc = true)`
- 错误：`AppFunctionInvalidArgumentException`、`AppFunctionElementNotFoundException` 等
- 调用方：`AppFunctionManager`，`isAppFunctionEnabled(packageName, functionId)`，权限 `android.permission.EXECUTE_APP_FUNCTIONS`
- 调试：`adb shell cmd app_function list-app-functions`
- 与 MCP：官方称其为 MCP 工具的移动端对应物；Agent 可同时考虑远程 MCP 工具和本地 AppFunctions

**Claude 工具执行位置**

| | 客户端工具 | 服务端工具 |
|---|---|---|
| 谁执行 | 你的程序 | Anthropic 基础设施 |
| 例子 | 自定义工具、`bash`、`text_editor`、`computer` | `web_search`、`web_fetch`、`code_execution` |
| 你要做 | 执行后回传 `tool_result`（`tool_use_id` 要对应） | 直接读结果 |
| 停止原因 | `stop_reason: "tool_use"` | — |

**降延迟（Claude 官方）**：选更快的模型（当前为 Claude Haiku 5.5，`claude-haiku-5-5`）、调低 effort（`output_config={"effort": "low"}`）、缩短输入输出、流式输出；关注首 token 时间（TTFT）。

### 来源说明

- AppFunctions 内容来自 Android 官方文档（非 LangChain/Claude 来源）。
- 代码用剧本模型代替真实 LLM，只演示路由和工具流程；已在本地实测。

### 延伸学习

- Claude Academy · Building with the Claude API → Tool functions（工具命名、参数校验、错误信息）
- Docs by LangChain · Custom middleware → Dynamic model selection

### 参考资料

- [Tool use with Claude - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Choosing the right model - Claude Platform Docs](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
- [Reducing latency - Claude Platform Docs](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-latency)
- [Tool functions - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/tool-functions)
- [Custom middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [Models - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/models)
- [ChatOllama integration - Docs by LangChain](https://docs.langchain.com/oss/python/integrations/chat/ollama)
- [Overview of AppFunctions - Android Developers](https://developer.android.com/ai/appfunctions)
