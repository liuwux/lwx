# Day 22 · 可观测性（2026-10-20）

## 读前三问

1. trace 和 span（LangSmith 里叫 run）分别是什么？一次 Agent 对话会长成什么样的树？
2. 线上 Agent「嘴上说做了、实际没做」，你怎么定位？
3. 给 Agent 打日志时，哪些该默认记录，哪些要谨慎？

## 从一个场景开始

昨天你用评测集拦住了一次回归。可评测集只覆盖你想得到的用例。

版本上线一周后，客服转来一条投诉：「我说主驾太冷，语音回复『已为您调到 26 度』，可空调根本没动。」你在车上试了几十遍，每次都正常。

Android 上遇到这种问题，你会翻 logcat、看崩溃堆栈。但 Agent 没崩溃：模型回复得很顺，程序也没抛异常。你手里只有一句最终回复，不知道中间模型调了哪些工具、传了什么参数、工具回了什么。

缺的是一份**完整的执行记录**：这次对话里每一步做了什么、花了多久、结果是什么。这就是今天的主题：可观测性（observability）。

## 一句话总结

可观测性就是把 Agent 的每次对话记录成一棵调用树，能看到每一步的输入、输出、耗时和花费，再加上版本和用户标签，让线上问题能被搜出来、看清楚、复现，最后补进评测集。

## 先跑起来

把投诉场景复现出来：模型给 `set_ac` 传了错误的区域名「主驾」，工具返回了错误，模型却仍然说「已调好」。我们用 LangChain 自带的 `RunCollectorCallbackHandler` 在本地收集执行记录，它收集的 run 和上传到 LangSmith 的是同一种数据结构。无 API Key，模型决策用剧本代替；Agent 循环、工具执行和 trace 收集都是真实库代码。

```python
# car_trace.py —— 给车机 Agent 打 trace，定位「说调好了，其实没调」（langchain 1.4.6）
import time
from langchain.agents import create_agent
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI
from langchain_core.tools import tool
from langchain_core.tracers.run_collector import RunCollectorCallbackHandler

@tool
def get_cabin_temp() -> str:
    """读取车内温度"""
    time.sleep(1.2)  # 模拟 CAN 总线读数很慢
    return "12度"

@tool
def set_ac(zone: str, temp: int) -> str:
    """设置某个区域的空调温度"""
    if zone not in ("driver", "passenger"):
        return f"error: 未知区域 {zone}"  # 工具报错，但只是一段文字，程序不会崩
    return "ok"

def tc(name, **args): return AI("", tool_calls=[{"name": name, "args": args, "id": name}])
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self

# 无 API Key：用剧本代替 Claude 的决策。模型传错了区域名，却仍然回复「已调好」
model = Scripted(messages=iter([tc("get_cabin_temp"), tc("set_ac", zone="主驾", temp=26), AI("已为您调到26度")]))
agent = create_agent(model, tools=[get_cabin_temp, set_ac])

collector = RunCollectorCallbackHandler()  # 本地收集 run 树；换成 LangSmith 时只需设环境变量
result = agent.invoke({"messages": [{"role": "user", "content": "主驾太冷了"}]},
                      config={"callbacks": [collector], "run_name": "car_voice_turn",
                              "tags": ["v2.3.0"], "metadata": {"vin": "LSV***001", "thread_id": "t-42"}})
print("用户听到：", result["messages"][-1].content)

def show(run, depth=0):  # 按父子关系打印 run 树：类型、名字、耗时、工具输出
    ms = (run.end_time - run.start_time).total_seconds() * 1000
    tool_out = run.outputs.get("output") if run.run_type == "tool" and run.outputs else None
    note = f"  → {getattr(tool_out, 'content', tool_out)}" if tool_out else ""
    print(f"{'  ' * depth}[{run.run_type}] {run.name}  {ms:.0f}ms{note}")
    for child in sorted(run.child_runs, key=lambda r: r.start_time):
        show(child, depth + 1)

root = collector.traced_runs[0]
print("trace_id:", root.trace_id, "| tags:", root.tags, "| metadata:", root.extra.get("metadata"))
show(root)
```

**运行**：`pip install langchain && python car_trace.py`

**实测输出**（2026-10-20，Python 3.13，langchain 1.4.6）：

```
用户听到： 已为您调到26度
trace_id: 01a156cb-97c2-701c-8d8d-1e9c3a08a560 | tags: ['v2.3.0'] | metadata: {'ls_integration': 'langchain_create_agent', 'vin': 'LSV***001', 'thread_id': 't-42'}
[chain] car_voice_turn  1211ms
  [chain] model  1ms
    [llm] Scripted  1ms
  [chain] tools  1203ms
    [tool] get_cabin_temp  1201ms  → 12度
  [chain] model  1ms
    [llm] Scripted  0ms
  [chain] tools  2ms
    [tool] set_ac  0ms  → error: 未知区域 主驾
  [chain] model  1ms
    [llm] Scripted  0ms
```

**该看什么**：
- 第一行用户听到的是「已调好」，但树里 `set_ac` 的输出是 `error`。**只看最终回复永远发现不了这个 bug**，只有 trace 能看到。
- 耗时一眼就能看出瓶颈：1211ms 里有 1201ms 花在 `get_cabin_temp`，不是模型慢。
- 树的结构就是 Agent 循环：model → tools → model → tools → model。`tags` 和 `metadata` 挂在根节点上，以后可以按版本号或车辆筛选。

## 核心概念

### 1. trace 与 span：一次对话是一棵树

- **是什么**：span 是「一步工作」的记录，比如一次模型调用或一次工具调用，带输入、输出、开始和结束时间。LangSmith 把它叫 run，Langfuse 叫 observation。trace 是同一次请求下所有 span 组成的树，靠同一个 trace ID 串起来。多轮对话再用 thread（会话）把多个 trace 串起来。
- **没有它会怎样**：只能看到最终回复，中间哪一步错了、哪一步慢了都不知道。
- **对应你熟悉的**：Android Studio Profiler 的 System Trace / Perfetto，一层层嵌套的时间片；trace ID 像跨进程的调用链 ID。

| 层级 | 车机例子 |
|---|---|
| thread | 车主这次上车后的整段对话 |
| trace | 「主驾太冷了」这一轮 |
| span / run | 一次模型调用、一次 `set_ac` 调用 |

### 2. 记什么：结构默认记，内容谨慎记

- **是什么**：结构数据指耗时、模型名、工具名、token 数、是否报错；内容数据指提示词原文、工具参数和返回值。
- **没有它会怎样**：只记结构，排障时看不到参数；什么都记，车主语音里的地址、电话就进了日志系统。
- **对应你熟悉的**：Android 上不能把用户隐私打进 logcat 或崩溃上报，道理一样。

官方做法值得在面试里引用：Claude Agent SDK 和 Anthropic Python SDK 的 OpenTelemetry 导出**默认只记结构，不记内容**，提示词和工具输出要单独打开开关。Anthropic 在自己的多智能体研究系统里也是「监控 Agent 的决策模式和交互结构」，不去看具体对话内容。折中做法是：结构全量记录，内容脱敏或只在采样、调试时记录。

### 3. 标签与元数据：让 trace 能被搜到

- **是什么**：tags 是字符串标签（如版本号），metadata 是键值对（如车辆 ID、会话 ID、环境）。LangSmith 里子节点会继承父节点的 tags 和 metadata。
- **没有它会怎样**：一天几十万条 trace，你找不到投诉的那一条，也没法回答「v2.3.0 上线后报错率是不是升了」。
- **对应你熟悉的**：Firebase Crashlytics 上报时附带的 app 版本、设备型号、自定义 key。

最少要带四类：**版本**（提示词或代码版本）、**会话 ID**、**用户或设备标识**（脱敏）、**环境**（测试或生产）。有了这些，「按版本对比报错率」「按会话回放整段对话」都只是一次筛选。

### 4. 线上排障闭环：从告警到评测用例

- **是什么**：看监控指标发现异常，搜出对应 trace，读树定位到具体那一步，再在本地复现、修复，最后把这条 trace 加进评测集。
- **没有它会怎样**：同一个 bug 修完下次又回来；评测集永远只有你自己想到的用例。
- **对应你熟悉的**：线上崩溃 → 看堆栈 → 修 → 写回归测试。

Agent 的特点是**不会崩溃，只会做错**。所以要专门监控「沉默的失败」：工具返回错误但回复说成功、同一个工具被反复调用、单次对话 token 数异常飙升。LangSmith 支持把 trace 直接加进数据集，这就接上了 Day 21 的评测。

## 高频面试题

**1. Agent 的可观测性和普通服务的日志监控有什么不同？**
- Agent 不确定：同样的输入可能走不同路径，必须记录每次的完整轨迹。
- 失败往往不抛异常，而是「做错了还说做对了」，要看工具结果和最终回复是否一致。
- 要额外记录 token 用量和成本。
- 内容里常有用户隐私，记录要分级。

**2.（追问）为什么 trace 要做成树，平铺一串日志不行吗？**
- 树能表达父子关系：这次工具调用属于哪一轮、哪个子 Agent。
- 能按节点算耗时，一眼找出瓶颈。
- 多智能体时，子 Agent 的调用挂在父 Agent 的工具节点下，整条委派链在一个 trace 里。
- 跨服务时靠传递 trace 上下文把不同进程的 span 接到同一棵树。

**3.（场景题）车主投诉「说调好了，空调没动」，你怎么排查？**
- 用会话 ID 或车辆标识加时间段搜出对应 trace。
- 看 `set_ac` 的参数和返回：发现区域名传错，工具返回 error。
- 定因：模型忽略了工具错误；修工具描述（列出合法区域）并让错误更显眼。
- 把这条 trace 加入评测集，加一条「工具报错时不许说成功」的检查，再加报错率告警。

**4. 线上 trace 要不要全量记录提示词和回复？**
- 结构数据全量记录，成本低、无隐私风险。
- 内容默认不记，或脱敏后记录，敏感字段打码。
- 调试或灰度时按比例采样、限期保存。
- 官方 SDK 也是默认只记结构，内容需显式打开。

## 常见误区

1. **「有日志就够了」**：平铺的文本日志串不起父子关系，也没有每一步的耗时和 token。Agent 需要的是结构化的调用树。
2. **「没报错就是没问题」**：Agent 最常见的故障是沉默的失败。要监控工具报错率、重复调用次数和 token 异常，而不是只看异常数。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：挂上收集器跑 Agent，再递归打印 run 树**（取自 `car_trace.py`）

```python
# 创建本地收集器，它会记下这次调用产生的所有 run
collector = RunCollectorCallbackHandler()
# 调用 Agent：用户说一句话
result = agent.invoke({"messages": [{"role": "user", "content": "主驾太冷了"}]},
                      # 通过 config 挂上收集器，并给根节点起名、打版本标签、附带车辆和会话信息
                      config={"callbacks": [collector], "run_name": "car_voice_turn",
                              "tags": ["v2.3.0"], "metadata": {"vin": "LSV***001", "thread_id": "t-42"}})
# 打印用户最终听到的那句话
print("用户听到：", result["messages"][-1].content)

# 递归函数：打印一个节点，再按时间顺序打印它的所有子节点
def show(run, depth=0):
    # 算出这一步的耗时（毫秒）
    ms = (run.end_time - run.start_time).total_seconds() * 1000
    # 如果是工具节点，取出它的输出
    tool_out = run.outputs.get("output") if run.run_type == "tool" and run.outputs else None
    # 工具输出可能是消息对象，取它的 content
    note = f"  → {getattr(tool_out, 'content', tool_out)}" if tool_out else ""
    # 按层级缩进，打印类型、名字、耗时和工具输出
    print(f"{'  ' * depth}[{run.run_type}] {run.name}  {ms:.0f}ms{note}")
    # 子节点按开始时间排序后逐个递归
    for child in sorted(run.child_runs, key=lambda r: r.start_time):
        show(child, depth + 1)

# 第一条收集到的 run 就是整棵树的根
root = collector.traced_runs[0]
# 从根开始打印
show(root)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 收集器放在 `config={"callbacks": [collector]}` 里，是**列表**；名字字段是 `run_name`，不是 `name`。
- 子节点属性是 `child_runs`，根节点从 `collector.traced_runs[0]` 取。
- 耗时要先相减得到 timedelta，再 `.total_seconds() * 1000`。

**昨日复习（只做第③步）**：Day 21 的「评测主循环」。只看你自己写的注释，重写双层 for 循环、`car.clear()`、结果分和轨迹分那几行，重点核对：参数名 `trajectory_match_mode`，评分器用 `outputs=` / `reference_outputs=`，结果取 `["score"]`。

## 动手练习（30 分钟）

在 `car_trace.py` 上改：

1. 写一个函数 `find_silent_failures(root)`：遍历 run 树，找出所有输出以 `error` 开头的工具节点；如果找到了，而最终回复里含「已」字，就打印「⚠ 沉默失败」。
2. 在 `show` 里加一个规则：单个节点耗时超过 1000ms 时，在行尾标「🐢 慢」。
3. 把剧本改成模型连续 3 次调用 `get_cabin_temp`，写一个计数检查：同一工具在一个 trace 里被调用超过 2 次就告警。

## 读后自测

1. trace 和 span（run）分别是什么？一次 Agent 对话会长成什么样的树？→ 见核心概念 1、「先跑起来」的输出
2. 线上 Agent「嘴上说做了、实际没做」，你怎么定位？→ 见核心概念 4、面试题 3
3. 打日志时哪些该默认记录，哪些要谨慎？→ 见核心概念 2、面试题 4

---

## 参考区（用时查，不必通读）

### 细节速查

**LangSmith 概念**
- run（≈ OTel span）、trace（同一 trace ID 下的 run 集合，单个 trace 上限 25,000 个 run）、thread（多轮会话，用 metadata 里的 `thread_id` 等键分组）、project（一个应用的 trace 容器）、feedback（挂在 run 上的打分）、tags、metadata。
- SaaS 版 trace 保留 180 天；想长期保留就加入 dataset。
- 开启：`LANGSMITH_TRACING=true`、`LANGSMITH_API_KEY`、`LANGSMITH_PROJECT`（默认 `default`）；欧区设 `LANGSMITH_ENDPOINT`；`LANGCHAIN_CALLBACKS_BACKGROUND=false` 改为同步上报。
- 退出前确保上报：`from langchain_core.tracers.langchain import wait_for_all_tracers`。
- 局部开关：`with langsmith.tracing_context(enabled=True, project_name=...)`。
- 非 LangChain 代码：`@traceable(run_type="llm"|"tool"|..., name=...)`；`with langsmith.trace(name, run_type, inputs=...) as rt: rt.end(outputs=...)`；`langsmith.wrappers.wrap_openai`。
- 配置：`invoke(..., {"tags": [...], "metadata": {...}, "run_name": "..."})` 或 `.with_config(...)`；tags/metadata 会被子节点继承，`run_name` 只作用于当前节点。

**Claude Agent SDK（OpenTelemetry）**
- 三种信号：metrics（token、成本、会话等计数）、log events（每次提示、API 请求、API 错误、工具结果）、traces（beta）。
- 开启：`CLAUDE_CODE_ENABLE_TELEMETRY=1`；traces 还需 `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`；`OTEL_TRACES_EXPORTER` / `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER=otlp`；`OTEL_EXPORTER_OTLP_ENDPOINT`、`OTEL_EXPORTER_OTLP_PROTOCOL`、`OTEL_EXPORTER_OTLP_HEADERS`。
- 通过 SDK 运行时不要用 `console` exporter（stdout 是 SDK 的消息通道）。
- span 名：`claude_code.interaction`（一轮）→ `claude_code.llm_request`、`claude_code.tool`（含 `tool.blocked_on_user`、`tool.execution`）、`claude_code.hook`；子 Agent 的 span 挂在父 Agent 的 `claude_code.tool` 下；span 带 `session.id`。
- 打标：`OTEL_SERVICE_NAME`、`OTEL_RESOURCE_ATTRIBUTES`（如 `service.version`、`deployment.environment`、`enduser.id`，值需百分号编码）。
- 内容开关（默认关）：`OTEL_LOG_USER_PROMPTS`、`OTEL_LOG_TOOL_DETAILS`、`OTEL_LOG_TOOL_CONTENT`、`OTEL_LOG_RAW_API_BODIES`。
- 导出失败默认静默；`CLAUDE_CODE_OTEL_DIAG_STDERR=1` 可输出诊断。SDK 会自动传播 W3C `TRACEPARENT`，Agent 的 span 能挂到你应用的 span 下。

**Anthropic Python SDK（OpenTelemetry，beta）**
- 注册 `TracerProvider` 后每次 `messages.create` 产生 span `anthropic.messages.create`。
- 默认 `traces.content_mode = metadata_only`；改为 `content` 才记录提示和回复（单 span 默认上限 0.75 MB）。
- `conversation_id` 设置 `gen_ai.conversation.id`；`ANTHROPIC_OPEN_TELEMETRY=false` 关闭。

**Langfuse 数据模型**（非 LangChain/Claude 来源）：trace、observation（span / generation / event，可嵌套）、session（多轮分组）、`user_id`；基于 OpenTelemetry，可同时发往多个后端。

### 来源说明

- 示例代码的模型决策是剧本，未调用真实 Claude；未实际上传到 LangSmith（无 API Key），用 `RunCollectorCallbackHandler` 在本地收集同结构的 run。
- Langfuse 为非 LangChain/Claude 来源，仅用于数据模型对照。

### 延伸学习

- LangChain 文档 · LangSmith Tracing quickstart 与 Observability concepts
- Claude 文档 · Agent SDK 的 Observability with OpenTelemetry；Claude Code 的 Monitoring 参考

### 参考资料

- [Observability concepts - Docs by LangChain](https://docs.langchain.com/langsmith/observability-concepts)
- [Trace LangChain applications - Docs by LangChain](https://docs.langchain.com/langsmith/trace-with-langchain)
- [Custom instrumentation - Docs by LangChain](https://docs.langchain.com/langsmith/annotate-code)
- [Observability with OpenTelemetry - Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/observability)
- [Use OpenTelemetry with the Claude SDK - Claude Platform Docs](https://platform.claude.com/docs/en/manage-claude/opentelemetry/sdk)
- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Observability Data Model - Langfuse](https://langfuse.com/docs/observability/data-model)（非 LangChain/Claude 来源）
