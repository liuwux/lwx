# Day 4 · Function Calling / Tool Use（第二轮：工具规模化、中间结果与隐藏参数）（2026-11-01）

## 今日 AI 动态（2026-11-01）

### 趋势观察

周末官方发布很少，关注的厂商里只有 OpenAI 的 Python SDK 有一次和 Agent 相关的更新：给托管 Agent 加上可配置的上下文压缩。推测：「上下文满了怎么办」正在从开发者自己写的 harness 逻辑，变成平台托管 Agent 的一个配置项，这和昨天讲的长任务 harness 是同一个问题。

### 今日要点

- OpenAI Python SDK 3.42.0 发布，为 beta 版的 agents、sessions 和定时 Agent 运行加入上下文压缩设置。

### 分公司动态

#### 1. OpenAI

**[feat(api): add agent context compaction settings（openai-python PR #4209，随 3.42.0 发布）](https://github.com/openai/openai-python/pull/4209)**（2026-10-31）

PR 于 10 月 29 日合入，随 3.42.0 版本在 10 月 31 日发布（PyPI 上传时间为 2026-10-31 00:27 UTC）。它让 beta 版的 agents、sessions 和定时 Agent 运行可以配置上下文压缩，用 `async_` 参数选择后台压缩还是阻塞压缩，也能读到 Agent、会话和定时任务当前生效的压缩设置。

**为什么重要**：压缩时机（后台做还是等它做完再继续）直接影响长对话 Agent 的延迟和连贯性，现在它成了托管 Agent 的显式配置项，选型时值得对比。

### 动态来源

- [openai/openai-python PR #4209: add agent context compaction settings](https://github.com/openai/openai-python/pull/4209)

---

> 第二轮说明：第一轮 Day 4 讲过完整调用流程、JSON Schema、`tool_choice`、`strict`、并行调用和错误回传。今天不再重复，改讲工具数量和数据量变大以后的三个问题：工具定义太多怎么办，中间结果太大怎么办，哪些参数不该让模型填。

## 读前三问

1. 车机助手接了 200 个工具，每次请求都把全部定义发给模型会有什么问题？Claude 的 tool search 是怎么解决的？
2. 「查最近 30 天行程，找出最费电的 3 次」要调几十次工具，怎么避免几十份结果全部塞进上下文？
3. 车辆 VIN、用户身份这类参数为什么不该出现在工具 schema 里？LangChain 里怎么把它们传给工具？

## 从一个场景开始

你的车机助手从 8 个工具长到了 200 个：导航、空调、座椅、媒体、车辆诊断，还接了 4 个第三方 MCP server。上线后出现三件事：

- 用户还没说话，单是工具定义就占了几万 token，首字延迟变长，账单也涨了。
- 模型开始选错工具，`media_play` 和 `podcast_play` 经常分不清。
- 有人在测试中对助手说「帮我查一下 VIN 为 XXX 的车在哪」，模型老老实实把别人的 VIN 填进了 `get_location(vin)`。

第一轮学的调用流程本身没错，问题出在规模和边界上：工具一多、数据一大、参数一敏感，原来的写法就扛不住了。

## 一句话总结

工具多了按需加载，数据大了让代码在沙箱里先处理、只把结论交给模型，身份和权限类参数由运行时注入，不让模型填。工具结果里只放数据，不放指令。

## 先跑起来

用 LangChain 的 `ToolRuntime` 让工具从运行时上下文拿 VIN 和乘员身份，模型只负责填温度。用脚本化的假模型代替真实模型，不需要 API Key。

```python
# hidden_args_demo.py —— 模型只填业务参数，车辆身份由运行时注入（脚本化假模型，无需 API Key）
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.messages import AIMessage
from langchain.tools import tool, ToolRuntime
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.utils.function_calling import convert_to_openai_tool

@dataclass
class CarContext:
    vin: str          # 车辆识别码：来自登录态，不让模型填
    role: str         # driver / passenger

@tool
def set_ac(temp: int, runtime: ToolRuntime[CarContext]) -> str:
    """设置空调温度（16-30 摄氏度）。用户说冷、热或报出具体温度时使用。"""
    if runtime.context.role != "driver":
        return "拒绝：只有驾驶员可以调空调"
    return f"[{runtime.context.vin}] 空调已设为 {temp}℃"

@tool
def get_battery(runtime: ToolRuntime[CarContext]) -> str:
    """查询当前电量和续航。用户问电够不够、能开多远时使用。"""
    return f"[{runtime.context.vin}] 电量 38%，续航 160 km"

print("模型看到的 set_ac 参数:", convert_to_openai_tool(set_ac)["function"]["parameters"])

class ScriptedModel(GenericFakeChatModel):
    def bind_tools(self, tools, **kw):  # 假模型不需要真的绑定工具
        return self

def script():  # 第 1 轮：并行调两个工具；第 2 轮：给出最终回答
    return iter([AIMessage("", tool_calls=[
                    {"name": "set_ac", "args": {"temp": 22}, "id": "c1"},
                    {"name": "get_battery", "args": {}, "id": "c2"}]),
                 AIMessage("已调到 22℃，电量 38%，够开到下一个充电站。")])

for role in ("driver", "passenger"):
    agent = create_agent(ScriptedModel(messages=script()), tools=[set_ac, get_battery],
                         context_schema=CarContext)
    out = agent.invoke({"messages": [{"role": "user", "content": "有点热，顺便看看电量"}]},
                       context=CarContext(vin="VIN-DEMO-001", role=role))
    print(f"\n--- role={role}")
    for m in out["messages"]:
        print(f"{m.type:>5}: {m.content or m.tool_calls}")
```

**运行**：`pip install -U langchain langgraph && python hidden_args_demo.py`

**实测输出**（2026-11-01，Python 3.13，langchain 1.4.7）：

```
模型看到的 set_ac 参数: {'properties': {'temp': {'type': 'integer'}}, 'required': ['temp'], 'type': 'object'}

--- role=driver
human: 有点热，顺便看看电量
   ai: [{'name': 'set_ac', 'args': {'temp': 22}, 'id': 'c1', 'type': 'tool_call'}, {'name': 'get_battery', 'args': {}, 'id': 'c2', 'type': 'tool_call'}]
 tool: [VIN-DEMO-001] 空调已设为 22℃
 tool: [VIN-DEMO-001] 电量 38%，续航 160 km
   ai: 已调到 22℃，电量 38%，够开到下一个充电站。

--- role=passenger
human: 有点热，顺便看看电量
   ai: [{'name': 'set_ac', 'args': {'temp': 22}, 'id': 'c1', 'type': 'tool_call'}, {'name': 'get_battery', 'args': {}, 'id': 'c2', 'type': 'tool_call'}]
 tool: 拒绝：只有驾驶员可以调空调
 tool: [VIN-DEMO-001] 电量 38%，续航 160 km
   ai: 已调到 22℃，电量 38%，够开到下一个充电站。
```

**该看什么**：
- 第一行：模型拿到的 schema 里只有 `temp`，`runtime` 参数整个被藏起来了。模型想填 VIN 也没有地方填。
- 乘客那一组：权限检查在工具里做，所以 `set_ac` 返回了拒绝。但最后一句回答仍然说「已调到 22℃」，因为假模型是写死的，不读工具结果。真实模型会读到拒绝，再改口；这也说明权限必须由工具自己把关，不能指望模型。
- 两个工具调用在同一条 AI 消息里，`create_agent` 执行完后各自回一条 `ToolMessage`，对应第一轮讲的并行调用。

## 核心概念

### 1. 工具定义本身就占上下文：按需加载

- **是什么**：每个工具的名称、描述和 schema 每次请求都要发给模型，按输入 token 计费。Claude 的 tool search 让你给大部分工具标上 `defer_loading: true`：模型一开始只看到常用的几个工具和一个「搜索工具」，需要时先搜，API 再把搜到的工具定义展开给它。
- **没有它会怎样**：官方举的例子是 5 个 MCP server、58 个工具，光定义就约 55K token；工具超过 30～50 个，选择准确率也会明显下降。
- **对应你熟悉的**：Android 的按需加载模块（Dynamic Feature Module）：不是所有功能都打进首次安装包，用到才下载。

三点要记住：`tools` 数组里仍然要发全部定义，`defer_loading` 只决定哪些进模型的上下文；至少保留一个不延迟的工具，官方建议把最常用的 3～5 个留在外面；延迟加载的工具不进系统提示前缀，搜到后以内联方式追加，所以 prompt cache 不会失效。也可以自己用向量检索实现搜索工具，返回 `tool_reference` 块即可。

### 2. 中间结果不必进上下文：让模型写代码调工具

- **是什么**：Programmatic Tool Calling（PTC）。给工具加上 `allowed_callers: ["code_execution_20260120"]`，模型就不再一次一次地调工具，而是在代码执行沙箱里写一段 Python，循环调用、过滤、汇总，只把最终输出交回上下文。
- **没有它会怎样**：查 30 次行程就是 30 个来回，每份原始结果都进上下文，又慢又贵，模型还要在一大堆数据里找答案。官方的报销合规例子里，约 200KB 原始数据最后只剩约 1KB 进上下文。
- **对应你熟悉的**：在 Room 里写一条 SQL 让数据库算好 `ORDER BY ... LIMIT 3` 再返回，而不是把整张表读进内存。

它不是万能的。适合扇出（同样的查询做几十次）、结果很大但只要摘要的场景；不适合每一步都要模型看过上一步结果再决定的串行流程。官方在 τ²-bench 这类每轮只调一两次工具的任务上实测：效果不变，成本反而高了约 8%。另外 `allowed_callers` 不是安全边界，客户端仍要能处理模型直接发起的调用。

### 3. 模型不该填的参数：运行时注入

- **是什么**：把参数分成两类。业务参数（温度、目的地）由模型根据对话填；身份和权限参数（用户 ID、VIN、乘员角色）由服务端从登录态、请求上下文里取，直接注入工具，不出现在 schema 里。LangChain 用 `ToolRuntime` 实现：`runtime.context` 放这次调用不可变的数据，`runtime.state` 放对话状态，`runtime.store` 放跨会话的长期数据。
- **没有它会怎样**：参数出现在 schema 里，模型就可能根据用户输入、甚至工具结果里被注入的文字去填，场景里查别人车辆位置就是这样发生的。
- **对应你熟悉的**：Hilt 注入的 `UserSession`。ViewModel 不会让 UI 层把用户 ID 当参数传进来，而是从注入的会话对象里取。

`config` 和 `runtime` 是保留参数名，自己的工具参数不要用这两个名字。

### 4. 工具结果里只放数据

- **是什么**：`tool_result` 是模型读取外部数据的通道，Claude 会对其中像指令的内容保持警惕。官方排障文档里有一条：如果你把自己的指令写在 `tool_result` 里，模型可能把它当作提示词注入，拒绝执行或者要求用户确认。指令应该放到 `tool_result` 之后的用户消息里。
- **没有它会怎样**：为了「顺便提醒模型」，把「请用一句话回答」塞进工具结果，结果模型反而起疑；反过来，第三方 MCP 返回的网页内容如果夹带指令，也是走这个通道进来的。
- **对应你熟悉的**：WebView 的 `addJavascriptInterface`：网页来的数据只能当数据用，不能让它随便调用原生方法。

同一类格式边界还有两条：并行调用的多个结果必须放在同一条用户消息里，分开发会让模型以后少并行；对话中途往 `tools` 数组前面插新工具会让缓存失效，要动态加工具就用 `defer_loading` 配合 tool search。

## 高频面试题

**Q1（场景题）：车机助手接了 200 个工具，延迟高、经常选错工具，你怎么改？**
- 先看数据：工具定义占多少 token，选错集中在哪几组相似工具。
- 合并职责重叠的工具，统一前缀命名（`media_`、`nav_`），描述里写清什么时候用。
- 其余工具 `defer_loading`，常用的 3～5 个留在外面，配合 tool search 按需加载。
- 用评测集回归「工具选择正确率」，对比改动前后的 token 和延迟。

**Q2（追问）：用了 tool search，prompt cache 还有效吗？中途要加一个新工具怎么办？**
- 有效：延迟加载的工具不进系统提示前缀，搜到的定义以内联方式追加，前缀不变。
- 带 `defer_loading` 的工具不能同时加 `cache_control`，缓存断点放在不延迟的工具上。
- 中途加工具别插到 `tools` 前面，标成 `defer_loading` 让模型搜到它，缓存就不受影响。

**Q3（场景题）：「最近 30 天哪 3 次行程最费电」，普通工具调用和 PTC 怎么选？**
- 这是扇出加聚合：30 次同样的查询，只要前 3 名，适合 PTC。
- 原始行程数据留在沙箱里，只有排序后的结论进上下文，省 token 也省来回。
- 如果每一步都要模型看过上一步再决定，或者只调一两次，用普通调用，PTC 反而更贵。
- 上线前用真实流量对比开启前后的输入 token。

**Q4（追问）：为什么 VIN 不让模型填？在 prompt 里写「只能用当前车辆的 VIN」不行吗？**
- prompt 约束是软的，用户输入或被注入的工具结果都可能让模型填进别的值。
- 身份类参数从服务端登录态注入，schema 里没有这个字段，模型想填也填不了。
- 权限检查在工具内部做（如只有驾驶员能调空调），按「模型可能被骗」来设计。
- 同理，`allowed_callers` 也只是引导，不是安全边界。

## 常见误区

- **「`defer_loading` 的工具可以不放进 `tools` 数组」**：必须放。API 要用完整定义来做搜索和展开，搜到一个没定义的工具名会直接 400。
- **「PTC 总是更省钱」**：只在扇出、大结果的场景省。串行、调用少的任务里，沙箱和脚本的开销会超过节省，官方实测成本反而上升。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带运行时注入和权限检查的工具，以及按身份调用 Agent**（取自 `hidden_args_demo.py`）

```python
# 定义运行时上下文：车辆识别码和乘员角色，都不让模型填
@dataclass
class CarContext:
    # 车辆识别码，来自登录态
    vin: str
    # 乘员角色：driver 或 passenger
    role: str

# 声明一个工具：模型只填温度，运行时对象另外注入
@tool
def set_ac(temp: int, runtime: ToolRuntime[CarContext]) -> str:
    """设置空调温度（16-30 摄氏度）。用户说冷、热或报出具体温度时使用。"""
    # 权限检查放在工具里：不是驾驶员就拒绝
    if runtime.context.role != "driver":
        return "拒绝：只有驾驶员可以调空调"
    # 从注入的上下文取 VIN，拼出执行结果
    return f"[{runtime.context.vin}] 空调已设为 {temp}℃"

# 打印模型实际看到的参数 schema，确认 runtime 被藏起来了
print("模型看到的 set_ac 参数:", convert_to_openai_tool(set_ac)["function"]["parameters"])

# 分别以驾驶员和乘客身份各跑一次
for role in ("driver", "passenger"):
    # 创建 Agent：传入模型、工具列表和上下文的类型
    agent = create_agent(ScriptedModel(messages=script()), tools=[set_ac, get_battery],
                         context_schema=CarContext)
    # 调用时把这次的上下文对象单独传进去，不放进消息里
    out = agent.invoke({"messages": [{"role": "user", "content": "有点热，顺便看看电量"}]},
                       context=CarContext(vin="VIN-DEMO-001", role=role))
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 参数名必须叫 `runtime`，类型写 `ToolRuntime[CarContext]`；换个名字它就会变成普通参数，出现在模型看到的 schema 里。
- docstring 不能省：`@tool` 用它作为工具描述，没有描述会报错，描述写得差模型就选不准。
- `context=` 是 `invoke` 的关键字参数，和第一个参数（消息字典）并列；`create_agent` 里对应的是 `context_schema=`，传的是类，不是实例。

**昨日复习（只做第③步）**：第二轮 Day 3 的「处理各种停止原因的 Agent 循环」（`run` 函数）。只看你自己写的注释重写，重点核对：截断分支在 `messages.append` 之前；`tool_use_id` 取 `b.id`；最后的 `return "stopped:max_turns"` 在 `for` 循环外面。

## 动手练习（30 分钟）

在 `hidden_args_demo.py` 的基础上做两件事：

1. 给 `set_ac` 加范围校验：温度不在 16～30 之间就 `raise ValueError`。再按 LangChain 文档写一个 `@wrap_tool_call` 中间件，把异常转成 `ToolMessage`（内容写清合法范围），加到 `create_agent(middleware=[...])`。把脚本里的温度改成 35，确认输出里出现的是错误提示，而不是程序崩溃。
2. 写一个纯 Python 函数 `search_tools(query, catalog)`：`catalog` 是 `{工具名: 描述}` 的字典，按关键词匹配，返回 Claude 自定义 tool search 需要的格式：`[{"type": "tool_reference", "tool_name": ...}]`。用 6 个车机工具（空调、座椅、导航、媒体、电量、胎压）测试「有点冷」和「还能开多远」两个查询。

想一想：第 1 题为什么把错误回给模型，而不是直接返回给用户？第 2 题如果描述里只写「控制空调」，「有点冷」还能搜到吗？这说明工具描述该怎么写？

## 读后自测

1. 车机助手接了 200 个工具，每次请求都把全部定义发给模型会有什么问题？Claude 的 tool search 是怎么解决的？——见「核心概念」第 1 节
2. 「查最近 30 天行程，找出最费电的 3 次」要调几十次工具，怎么避免几十份结果全部塞进上下文？——见「核心概念」第 2 节
3. 车辆 VIN、用户身份这类参数为什么不该出现在工具 schema 里？LangChain 里怎么把它们传给工具？——见「核心概念」第 3 节

## AI 历史知识点：Function Calling API（2023，OpenAI）

**是什么**：开发者在 Chat Completions 接口里用 JSON Schema 描述函数（当时的参数叫 `functions` 和 `function_call`），模型不再只输出自然语言，而是可以输出一个包含函数参数的 JSON 对象，由应用去执行。官方的说法是「一种更可靠地把 GPT 的能力和外部工具、API 连接起来的新方式」。

**何时由谁提出**：OpenAI 于 2023 年 6 月 13 日在《Function calling and other API updates》中发布，随 `gpt-4-0613` 和 `gpt-3.5-turbo-0613` 两个模型上线，同一篇公告还发布了 16K 上下文版本和降价。

**为什么重要**：在此之前，让模型调工具主要靠提示词约定格式再用正则解析（如 ReAct 的文本格式），或者像 Toolformer 那样在训练中教模型插入 API 调用。Function Calling 把「工具调用」变成了 API 的一等公民：模型专门训练过、输出结构化、开发者不用再解析文本。官方当时列的用途，回答问题时调外部工具、把自然语言转成 API 调用或 SQL、从文本中抽取结构化数据，今天仍然是 Agent 的主要用法。

**和今天的趋势有什么联系**：公告里已经提醒，不可信的工具输出可能诱导模型做出意外操作，建议对真实世界的操作加用户确认，这正是今天第 3、4 节讲的问题。三年后，问题从「能不能调工具」变成了「工具太多怎么加载、结果太大怎么处理」，于是有了 tool search 和 PTC。

**原始来源**：
- [Function calling and other API updates - OpenAI（2023-06-13）](https://openai.com/index/function-calling-and-other-api-updates/)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| tool search 类型 | `tool_search_tool_regex_20251119`（Python 正则，不区分大小写，≤200 字符）、`tool_search_tool_bm25_20251119`（自然语言，≤500 字符）；都搜名称、描述、参数名和参数描述 |
| tool search 流程 | 模型发 `server_tool_use`（ID 前缀 `srvtoolu_`）→ API 返回 `tool_search_tool_result`，内含 `tool_reference` → API 自动展开定义 → 模型发普通 `tool_use`；不要给 `srvtoolu_` 回 `tool_result` |
| tool search 限制 | 每次请求最多 10,000 个延迟工具；默认每次搜索返回 5 个，`limit` 可设 1～10,000；全部延迟会 400；`defer_loading` 与 `cache_control` 不能同时用；搜不到返回空数组而非报错 |
| tool search 适用 | 10 个以上工具、定义超过约 10K token、多个 MCP server；少于 10 个且每次都用时不必用 |
| MCP 与 tool search | 不在单个定义上设，改在 `mcp_toolset` 的 `default_config` 或 `configs` 里设 `defer_loading` |
| 自定义 tool search | 自己的工具返回普通 `tool_result`，`content` 里放 `{"type": "tool_reference", "tool_name": ...}`；被引用的工具必须在 `tools` 里有定义 |
| tool search 效果（2025-11 官方博客） | 58 个工具场景上下文从约 77K 降到约 8.7K（减少 85%）；MCP 评测 Opus 4 从 49% 到 74%，Opus 4.5 从 79.5% 到 88.1% |
| PTC 配置 | 需要 `{"type": "code_execution_20260120", "name": "code_execution"}`；`allowed_callers`：`["direct"]`（默认）、`["code_execution_20260120"]`，官方建议二选一 |
| PTC 流程 | `tool_use` 带 `caller: {"type": "code_execution_20260120", "tool_id": "srvtoolu_..."}`；响应带 `container`，续发时必须带上容器 ID；回复只能含 `tool_result`，内容只能是字符串或 `text` 块 |
| PTC 限制 | 不支持 `strict: true`、`disable_parallel_tool_use: true`；`tool_choice` 指定的工具必须包含 `"direct"`；MCP connector 工具不能被代码调用；待处理调用约 4 分钟超时；不符合 ZDR |
| PTC 效果 | 智能搜索基准平均效果 +11%、输入 token −24%；75 个工具的项目管理基准输入 token 约 −38%；10～49 个工具的生产流量通常省 20%～40%；τ²-bench 成本约 +8% |
| 细粒度工具流式 | 工具上设 `eager_input_streaming: true` 并开启流式；参数以 `input_json_delta` 片段到达，不经服务端校验，可能是残缺 JSON；解析失败就不执行，回 `is_error: true`，内容为 `{"INVALID_JSON": 原始字符串}`（用 JSON 库序列化） |
| `input_examples` | 给复杂参数的工具举例；官方内部测试复杂参数准确率从 72% 到 90% |
| `ToolRuntime` 属性 | `context`（单次调用不可变数据）、`state`（短期对话状态）、`store`（长期存储）、`stream_writer`（进度推送）、`tool_call_id` |
| LangChain 其他 | 工具返回 `Command(update={...})` 可改状态，需附带 `ToolMessage(tool_call_id=runtime.tool_call_id)`；`return_direct=True` 时并行批次里全部为 true 才直接返回；错误处理用 `@wrap_tool_call` 中间件 |
| 常见报错 | `tool_use ids were found without tool_result blocks immediately after`：少了结果或结果不在最前；`All tools have defer_loading: true`；`Tool reference '...' not found in available tools` |

### 来源说明

- 演示代码用脚本化假模型（本次运行环境没有 API Key），只验证 `ToolRuntime` 注入和工具执行，不代表真实模型的回答。tool search 和 PTC 只依据官方文档讲解，未实际调用。
- 历史知识点来自 OpenAI 官方博客（非 LangChain/Claude 来源）。

### 延伸学习

- Claude Academy《Building with the Claude API》的「Tool use with Claude」一节，其中「Using multiple tools」「Fine grained tool calling」两课与今天相关。
- Anthropic 工程博客《Introducing advanced tool use for Claude》（2025-11-24）。

### 参考资料

- [Tool search tool - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
- [Programmatic tool calling - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
- [Fine-grained tool streaming - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming)
- [Troubleshooting tool use - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use)
- [Introducing advanced tool use for Claude - Anthropic](https://www.anthropic.com/engineering/advanced-tool-use)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
- [Tools - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/tools)
