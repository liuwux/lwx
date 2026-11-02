# Day 5 · 工具设计（第二轮：评测驱动、返回详略与错误分层）（2026-11-02）

## 今日 AI 动态（2026-11-02）

过去 24 小时关注的厂商没有新的官方发布。

---

> 第二轮说明：第一轮 Day 5 讲过粒度、命名、描述、返回值、幂等和权限边界。今天不再重复，改讲进阶的四件事：怎么用评测而不是凭感觉改工具；怎么让模型自己决定要多详细的返回；参数校验失败该怎么报；协议去掉会话以后，跨调用的状态放在哪里。

## 读前三问

1. 工具上线后，模型总在重复调用、参数也老填错。你怎么知道该改哪里，而不是凭感觉改描述？
2. 参数校验失败（比如预约时间早于现在），应该报协议错误还是工具执行错误？为什么？
3. 同一个搜索工具，有时只要一句概览，有时要拿 ID 去做下一步。返回值怎么设计？

## 从一个场景开始

车机助手有两个工具：`charging_search` 搜充电站，`charging_book` 预约充电桩。用户说「帮我约一个上海今天 10 点的快充」。

日志里出现了三个问题：

- 搜索每次返回 300 个站点的全部字段，一次就占掉几千 token，可用户只关心最近的两个。
- 模型把年份填成了去年。工具抛出异常，框架层把它当成「请求格式不对」报给客户端，模型只看到调用失败，不知道原因，于是换个写法又填了一次去年。
- 有人提议「在描述里加一句：年份必须是今年」。改完谁也说不清是变好了还是变差了。

这些不是第一轮那几条原则能解决的。要解决的是：怎么验证改动有没有用，返回多少才合适，错误怎么让模型看得见。

## 一句话总结

工具是写给模型用的接口：用评测集衡量改动，让模型按需选择返回的详细程度，把校验失败当作执行结果回给模型让它自己改，跨调用的状态用工具返回的句柄传递。

## 先跑起来

一个返回详略可选的搜索工具，加一个会做时间校验的预约工具。用 `@wrap_tool_call` 中间件把异常转成模型能读到的错误结果。用脚本化的假模型，不需要 API Key。

```python
# tool_design_demo.py —— 返回详略可选 + 校验错误回给模型自我纠正（脚本化假模型，无需 API Key）
from datetime import datetime
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain.messages import AIMessage, ToolMessage
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.utils.function_calling import convert_to_openai_tool

STATIONS = [("sh-01", "人民广场站", 2, "120kW", 1.2), ("sh-02", "外滩站", 0, "60kW", 2.5)]
NOW = datetime(2026, 11, 2, 9, 0)  # 固定“当前时间”，让输出可复现

@tool
def charging_search(city: str, response_format: Literal["concise", "detailed"] = "concise") -> str:
    """搜索城市内的充电站。默认 concise 只返回名称和空闲桩数；需要 station_id 去预约时用 detailed。"""
    if response_format == "concise":
        return "；".join(f"{n} 空闲{f}" for _, n, f, _, _ in STATIONS)
    return "\n".join(f"station_id={i} {n} 空闲{f} 功率{p} 距离{d}km" for i, n, f, p, d in STATIONS)
@tool
def charging_book(station_id: str, start: str) -> str:
    """预约充电桩。start 格式 YYYY-MM-DD HH:MM，必须晚于当前时间。"""
    if datetime.strptime(start, "%Y-%m-%d %H:%M") <= NOW:
        raise ValueError(f"start 必须晚于当前时间 {NOW:%Y-%m-%d %H:%M}，请换一个更晚的时间重试")
    return f"已预约 {station_id}，{start} 开始，预约号 R-1024"
@wrap_tool_call
def errors_to_model(request, handler):  # 把异常变成模型能读到的错误结果
    try:
        return handler(request)
    except Exception as e:
        return ToolMessage(f"错误：{e}", tool_call_id=request.tool_call["id"], status="error")

class ScriptedModel(GenericFakeChatModel):
    def bind_tools(self, tools, **kw):
        return self
calls = [("charging_search", {"city": "上海", "response_format": "detailed"}),
         ("charging_book", {"station_id": "sh-01", "start": "2025-11-02 10:00"}),  # 年份填错
         ("charging_book", {"station_id": "sh-01", "start": "2026-11-02 10:00"})]
script = [AIMessage("", tool_calls=[{"name": n, "args": a, "id": f"c{i}"}]) for i, (n, a) in enumerate(calls)]
model = ScriptedModel(messages=iter(script + [AIMessage("已为你预约人民广场站 10:00，预约号 R-1024。")]))
print("模型看到的 charging_search 参数:", convert_to_openai_tool(charging_search)["function"]["parameters"])
agent = create_agent(model, tools=[charging_search, charging_book], middleware=[errors_to_model])
out = agent.invoke({"messages": [{"role": "user", "content": "帮我约一个上海今天 10 点的快充"}]})
for m in out["messages"]:
    print(f"{m.type:>5}{'[' + m.status + ']' if m.type == 'tool' else ''}: {m.content or m.tool_calls[0]['args']}")
```

**运行**：`pip install -U langchain langgraph && python tool_design_demo.py`

**实测输出**（2026-11-02，Python 3.13，langchain 1.4.7）：

```
模型看到的 charging_search 参数: {'properties': {'city': {'type': 'string'}, 'response_format': {'default': 'concise', 'enum': ['concise', 'detailed'], 'type': 'string'}}, 'required': ['city'], 'type': 'object'}
human: 帮我约一个上海今天 10 点的快充
   ai: {'city': '上海', 'response_format': 'detailed'}
 tool[success]: station_id=sh-01 人民广场站 空闲2 功率120kW 距离1.2km
station_id=sh-02 外滩站 空闲0 功率60kW 距离2.5km
   ai: {'station_id': 'sh-01', 'start': '2025-11-02 10:00'}
 tool[error]: 错误：start 必须晚于当前时间 2026-11-02 09:00，请换一个更晚的时间重试
   ai: {'station_id': 'sh-01', 'start': '2026-11-02 10:00'}
 tool[success]: 已预约 sh-01，2026-11-02 10:00 开始，预约号 R-1024
   ai: 已为你预约人民广场站 10:00，预约号 R-1024。
```

**该看什么**：
- 第一行：`Literal` 变成了 schema 里的 `enum`，默认值是 `concise`。模型因为下一步要预约、需要 ID，所以主动选了 `detailed`。
- `tool[error]` 那一行：错误进了对话，内容写清了「当前时间是几点、该怎么改」，所以下一轮能改对。真实模型读到它才会改；假模型的三步是预先写好的。
- 把 `middleware=[errors_to_model]` 删掉再跑一次：`ValueError` 会直接中断整个 Agent，模型根本没有机会纠正。

## 核心概念

### 1. 用评测驱动改工具

- **是什么**：准备一批真实任务（每个任务配一个可验证的结果），让 Agent 在循环里跑完，统计准确率、工具调用次数、token 消耗和工具报错数，再读对话记录找原因。改完工具再跑一遍对比。
- **没有它会怎样**：改描述全凭感觉。Anthropic 举过一个例子：模型总给搜索词自动加上「2025」，搜索结果因此有偏差，是读对话记录才发现的，改一句描述就修好了。
- **对应你熟悉的**：Android 的性能基准测试（Macrobenchmark）：先测出基线，改完再测，用数字说话。

指标和改法大致对应：重复调用多，说明分页或返回上限不合适；参数错误多，说明描述不清，或者缺少示例。官方还建议把对话记录交给 Claude 分析、让它批量改工具，同时留一份没参与优化的测试集，防止只对练习题有效。

### 2. 让模型选择返回多详细

- **是什么**：给工具加一个 `response_format` 枚举参数，`concise` 只给结论，`detailed` 附上下一步要用的 ID。官方的 Slack 例子里，同样的信息 detailed 用了 206 个 token，concise 只用了 72 个。
- **没有它会怎样**：要么永远返回全量字段，浪费上下文；要么永远精简，模型想做下一步时又拿不到 ID，只好再调一次。
- **对应你熟悉的**：同一个 REST 接口用 `fields=` 参数选择返回哪些字段，列表页和详情页各取所需。

另外两条配套做法：数据量大就分页、过滤、截断，截断时告诉模型怎么缩小范围（例如「结果太多，请加上区县再搜」）；返回人能读懂的名字，不返回一长串 UUID，可以减少模型编造 ID。Claude Code 默认把单次工具返回限制在 25,000 token。

### 3. 错误分两层：协议错误和执行错误

- **是什么**：MCP 规范把工具错误分成两类。协议错误走 JSON-RPC 的 `error`，例如工具名不存在、请求结构不合法，客户端一般不会转给模型。执行错误放在正常结果里，标上 `isError: true`，例如外部 API 失败、业务规则不通过、参数校验不通过，客户端应该转给模型。
- **没有它会怎样**：参数校验失败时，如果当成协议错误返回，模型只知道「失败了」。规范的改动提案 SEP-1303 里举的就是这种情况：模型连续三次发出同样的过去日期，因为它看不到原因。
- **对应你熟悉的**：HTTP 的 400 和 200 两种响应。400 是请求格式不对，交给调用方的代码处理；200 里带 `{"ok": false, "msg": "余额不足"}`，是业务失败，界面要把原因展示给用户。

为什么 schema 校验之外还要在工具里校验？因为 JSON Schema 只能约束格式（比如日期写成 `YYYY-MM-DD`），表达不了「必须晚于现在」这种规则。错误信息要写成能直接照着改的话：错在哪里、正确的值是什么、下一步该做什么。

### 4. 跨调用的状态用句柄传递

- **是什么**：MCP 2026-07-28 版规范去掉了协议层的会话，每个请求都是独立的。需要跨调用保存状态的服务端，改为返回自己生成的句柄（handle），再由模型把句柄当普通参数传给下一个工具。
- **没有它会怎样**：如果状态藏在服务端的连接或会话里，换一台实例、断线重连，状态就丢了；模型也看不到当前进行到哪一步。
- **对应你熟悉的**：`PendingIntent` 或订单号：先拿到一个凭证，之后凭它继续操作，不依赖当前进程还活着。

在车机上，它最适合「先准备、再确认」的流程：`nav_plan_route` 返回 `route_id` 和摘要，让用户确认，再调 `nav_start(route_id)`。确认这一步就有了明确的边界，句柄还可以设置过期时间，并和车辆身份绑定。

## 高频面试题

**Q1（场景题）：工具上线后，模型调用次数很多、经常填错参数。你怎么改进？**
- 先建评测集：真实任务配可验证的结果，测出准确率、调用次数、token 和报错数作为基线。
- 读对话记录：调用多就看分页和返回上限，参数错就补描述和 `input_examples`。
- 每次只改一处，重跑对比；另留一份测试集，防止只对练习题有效。

**Q2（追问）：参数校验失败，为什么不返回 JSON-RPC 的 `-32602`？**
- 协议错误一般不转给模型，模型看不到原因，会盲目重试。
- 规范要求校验失败作为执行错误返回（`isError: true`），客户端应把它转给模型。
- 错误文本要写清楚错在哪里、合法范围是什么、建议怎么改。
- 工具名不存在、请求结构不合法，这些才是协议错误。

**Q3（场景题）：充电站搜索可能返回几百条，返回值怎么设计？**
- 默认只给最相关的前几条，并加上过滤参数（距离、功率、是否空闲）。
- 提供 `response_format`：概览用 concise，要预约时用 detailed 带上 `station_id`。
- 截断时说明总数和缩小范围的办法，或者告诉模型下一页怎么取。
- 返回站名而不是内部 UUID，ID 只放在 detailed 里。

**Q4（追问）：协议没有会话了，「先规划路线、用户确认后再导航」怎么设计工具？**
- 拆成两个工具：`nav_plan_route` 返回 `route_id` 和摘要，`nav_start` 接收 `route_id`。
- 句柄由服务端生成，带过期时间，并校验是否属于当前车辆。
- 用户确认放在两次调用之间，由应用层的审批节点控制，不靠模型自觉。

## 常见误区

- **「工具抛了异常，框架会自动转给模型」**：不一定。本例实测，`create_agent` 不加中间件时，`ValueError` 会直接中断 Agent。要用 `@wrap_tool_call` 之类的机制把异常转成错误结果，并且只转那些模型改得了的错误。
- **「加一个 `reasoning` 参数，让模型说明为什么调用」**：Claude 官方文档提醒，要求模型输出思考过程的参数可能触发 `reasoning_extraction` 拒答。需要的话，改成要一句简短说明或依据。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：会做业务校验的工具，加上把异常转成错误结果的中间件**（取自 `tool_design_demo.py`）

```python
# 声明预约工具：模型填站点 ID 和开始时间
@tool
def charging_book(station_id: str, start: str) -> str:
    """预约充电桩。start 格式 YYYY-MM-DD HH:MM，必须晚于当前时间。"""
    # 业务校验：schema 只管格式，「必须晚于现在」要在工具里判断
    if datetime.strptime(start, "%Y-%m-%d %H:%M") <= NOW:
        # 校验失败就抛异常，信息里写清当前时间和改法
        raise ValueError(f"start 必须晚于当前时间 {NOW:%Y-%m-%d %H:%M}，请换一个更晚的时间重试")
    # 校验通过，返回预约结果
    return f"已预约 {station_id}，{start} 开始，预约号 R-1024"

# 声明一个包裹每次工具调用的中间件
@wrap_tool_call
def errors_to_model(request, handler):
    # 正常情况：直接执行工具并返回结果
    try:
        return handler(request)
    # 工具抛异常时：不让 Agent 崩溃
    except Exception as e:
        # 转成一条错误状态的工具消息，对上这次调用的 ID，让模型读到原因
        return ToolMessage(f"错误：{e}", tool_call_id=request.tool_call["id"], status="error")

# 创建 Agent：传入模型、两个工具，并挂上错误中间件
agent = create_agent(model, tools=[charging_search, charging_book], middleware=[errors_to_model])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `tool_call_id` 取的是 `request.tool_call["id"]`，`tool_call` 是字典，要用方括号，不能写成 `.id`。
- `status="error"` 是 `ToolMessage` 的关键字参数，不要写进 content 里。
- `handler(request)` 必须 `return`，漏了 `return` 工具结果就丢了。

**昨日复习（只做第③步）**：第二轮 Day 4 的「带运行时注入和权限检查的工具，以及按身份调用 Agent」（`CarContext` 和 `set_ac`）。只看你自己写的注释重写，重点核对：参数名必须叫 `runtime`；`create_agent` 用 `context_schema=` 传类，`invoke` 用 `context=` 传实例。

## 动手练习（30 分钟）

在 `tool_design_demo.py` 的基础上：

1. 把 `STATIONS` 扩到 8 条，给 `charging_search` 加 `page: int = 1` 参数，每页 3 条。不是最后一页时，在结果末尾加一句「共 8 条，当前第 1 页，传 page=2 查看更多」。
2. 修改 `errors_to_model`：只捕获 `ValueError`（模型能改的），其他异常继续抛出。再给 `charging_book` 加一个会 `raise RuntimeError("充电平台超时")` 的分支，确认它会中断运行。

想一想：平台超时这类错误，应该让模型重试、换个站，还是直接告诉用户？由谁决定更合适？

## 读后自测

1. 工具上线后，模型总在重复调用、参数也老填错。你怎么知道该改哪里，而不是凭感觉改描述？——见「核心概念」第 1 节
2. 参数校验失败（比如预约时间早于现在），应该报协议错误还是工具执行错误？为什么？——见「核心概念」第 3 节
3. 同一个搜索工具，有时只要一句概览，有时要拿 ID 去做下一步。返回值怎么设计？——见「核心概念」第 2 节

## AI 历史知识点：Computer Use 计算机使用（2024，Anthropic）

**是什么**：让 Claude 像人一样操作电脑：看屏幕截图，移动鼠标，点击按钮，输入文字。开发者通过 API 接入，Claude 把指令转换成一连串电脑操作。

**何时由谁提出**：Anthropic 于 2024 年 10 月 22 日发布，与升级版 Claude 3.5 Sonnet、Claude 3.5 Haiku 一起推出，当时是公开测试版。

**为什么重要**：在 OSWorld 基准的「仅截图」类别中，Claude 3.5 Sonnet 得分 14.9%，次优系统是 7.8%；允许更多步数时达到 22.0%（人类约 70%～75%）。官方也坦白它「有时笨拙且容易出错」，滚动、拖拽、缩放都还有困难。训练中的关键一环是让模型准确数像素，才能把光标移到目标上。

**和今天的趋势有什么联系**：官方的说法是不再「为模型造工具」，而是「让模型适应已有的工具」。这和今天的主题正好互补：能写成专门工具的地方，按今天的方法把工具设计好，又快又准；没有 API 的旧软件，再交给 computer use。它带来的主要新风险是提示词注入，这也是今天要求工具结果「只放数据」的原因之一。

**原始来源**：
- [Introducing computer use, a new Claude 3.5 Sonnet, and Claude 3.5 Haiku - Anthropic（2024-10-22）](https://www.anthropic.com/news/3-5-models-and-computer-use)
- [Developing a computer use model - Anthropic（2024-10-22）](https://www.anthropic.com/news/developing-computer-use)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| 评测指标（Anthropic） | 顶层准确率、单次调用和整个任务的耗时、调用总数、token 消耗、工具报错数；验证器可用精确匹配或 Claude 评判，不要因格式差异误判 |
| 评测任务 | 基于真实数据，强任务可能需要几十次调用；弱任务例子：「下周和 jane 约个会」 |
| 工具合并例子 | `list_users`+`list_events`+`create_event` → `schedule_event`；`read_logs` → `search_logs`；三个客户查询 → `get_customer_context` |
| 命名空间 | 按服务或资源加前缀（`asana_search`、`jira_search`）；前缀还是后缀对评测结果有不小影响，因模型而异 |
| 描述长度（Claude 文档） | 每个工具至少 3～4 句：做什么、何时用和何时不用、参数含义、限制 |
| `input_examples` | 每个示例必须符合 `input_schema`，否则 400；简单示例约 20～50 token，复杂嵌套约 100～200；不支持服务端工具和 computer use |
| `strict: true` | 保证输入符合 schema；Opus 5.5、Sonnet 5.5 等模型上强制工具调用（`tool_choice: any`）会返回 400，文档建议 `auto` 配合 strict |
| Claude `tool_result` | `tool_use_id` 必填、`content`、`is_error`；`tool_result` 块必须排在 content 数组最前面，否则 400；输入无效时 Claude 会带修正重试 2～3 次 |
| MCP 工具定义字段（2025-11-25） | `name`（建议 1～128 字符，限字母数字 `_ - .`）、`title`、`description`、`icons`、`inputSchema`（不能为 null，无参数推荐 `{"type":"object","additionalProperties":false}`）、`outputSchema`、`annotations`（来自不可信服务端时必须视为不可信）、`execution.taskSupport` |
| MCP 结果 | `content`（text/image/audio/resource_link/resource）、`structuredContent`（有 `outputSchema` 时必须符合；为兼容还应附一份序列化 JSON 文本）、`isError` |
| MCP 错误分层 | 协议错误：未知工具、请求结构不合法、服务端错误；执行错误（`isError: true`）：API 失败、输入校验错误、业务逻辑错误（SEP-1303，Final） |
| MCP 2026-07-28 与工具相关 | 去掉会话和 `initialize` 握手；跨调用状态用服务端生成的句柄作为普通参数；`tools/list` 应按确定顺序返回，方便缓存；列表结果需带 `ttlMs`、`cacheScope`；所有结果带 `resultType`；`inputSchema`/`outputSchema` 允许任意 JSON Schema 2020-12 关键字；tasks 移到官方扩展 |
| LangChain 工具 | 类型注解必填，决定输入 schema；`args_schema` 可用 Pydantic，`Literal` 生成枚举；`config`、`runtime` 是保留参数名；`@wrap_tool_call` 中间件包裹工具调用 |

### 来源说明

- 演示代码用脚本化假模型（运行环境没有 API Key），只验证 schema 生成、错误回传和中间件行为，不代表真实模型的选择。
- 今日 AI 动态：已核对 Anthropic、Claude 博客、OpenAI API 更新日志、Google Workspace Updates、Meta Newsroom、DeepSeek、智谱、Qoder、Qwen Code 等官方页面，最新内容都早于 11 月 1 日。

### 延伸学习

- Claude Academy《Building with the Claude API》的「Tool use with Claude」一节（Tool schemas、Sending tool results、Using multiple tools），以及「Model Context Protocol」一节的「Defining tools with MCP」。

### 参考资料

- [Writing effective tools for AI agents - Anthropic（2025-09-11）](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Define tools - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Handle tool calls - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
- [Tools - MCP 规范 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [SEP-1303: Input Validation Errors as Tool Execution Errors](https://modelcontextprotocol.io/seps/1303-input-validation-errors-as-tool-execution-errors)
- [Key Changes - MCP 规范 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [Tools - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/tools)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
