# Day 18 · 框架对比（2026-10-16）

## 读前三问

1. LangChain 官方把 Agent 工具链分成哪三层？LangGraph、LangChain、Claude Agent SDK 各在哪一层？
2. 「框架」和「harness」最大的区别是什么？为什么 Claude Agent SDK 一跑起来就自带 Read、Bash 这些工具？
3. 同样叫「把任务交给另一个 Agent」，OpenAI Agents SDK 的 handoff 和 Claude 的 subagent 有什么不同？

## 从一个场景开始

组里要给车机助手加两个能力：一是语音说「空调调到 22 度」马上执行；二是「帮我规划周末自驾去千岛湖，写一份行程报告」，要查资料、比价、写文件，可能跑十几分钟。

技术选型会上大家各推各的：有人说 LangChain 上手快，有人说 LangGraph 能断点续跑，有人说 CrewAI 的「角色团队」很直观，还有人说 Claude Agent SDK 自带读写文件、搜索、子 Agent，开箱就能干长活。

你发现大家其实不在比同一种东西：有的是**积木**，有的是**底座**，有的是**装好家具的房子**。不先分清层次，就只能比 GitHub star 数。今天的任务就是给这些框架分层，再按场景选。

## 一句话总结

Agent 框架分三层：运行时管「可靠地跑」（持久化、中断恢复），框架管「快速搭」（模型和工具的统一接口），harness 管「开箱干长活」（自带工具、规划、子 Agent、上下文压缩）。按任务的确定性和时长选层，再按模型绑定和生态选具体产品。

## 先跑起来

同一个工具 `set_ac`（调空调），分别用「框架」LangChain 和「harness」Claude Agent SDK 接上。第一个文件用剧本模型代替 Claude，不需要 API Key；第二个文件调用真实的 Claude。

```python
# car_ac_langchain.py —— 框架路线：LangChain create_agent（langchain 1.4.6）
from langchain.agents import create_agent
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI

# 没有 API Key：用「剧本模型」代替 Claude 做决策；循环和工具执行是 LangChain 的真实代码
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self
model = Scripted(messages=iter([
    AI("", tool_calls=[{"name": "set_ac", "args": {"temp": 22}, "id": "c1"}]),
    AI("空调已调到22度。")]))

@tool
def set_ac(temp: int) -> str:
    """设置车内空调温度（16~30 度）"""
    return f"OK: 空调 {temp}℃"

agent = create_agent(model, tools=[set_ac], system_prompt="你是车机助手")
res = agent.invoke({"messages": [{"role": "user", "content": "有点热，空调调到22度"}]})
for m in res["messages"]:
    print(f"{m.type:>5}: {m.text or [c['name'] for c in m.tool_calls]}")
print("自带工具：无，只有你给的", [t.name for t in [set_ac]])
```

```python
# car_ac_claude_sdk.py —— Harness 路线：Claude Agent SDK（claude-agent-sdk 0.2.159）
import anyio
from claude_agent_sdk import (ClaudeAgentOptions, query, tool, create_sdk_mcp_server,
                              SystemMessage, AssistantMessage, ResultMessage, ToolUseBlock)

@tool("set_ac", "设置车内空调温度（16~30 度）", {"temp": int})
async def set_ac(args):
    return {"content": [{"type": "text", "text": f"OK: 空调 {args['temp']}℃"}]}

car = create_sdk_mcp_server(name="car", tools=[set_ac])   # 自定义工具以进程内 MCP server 形式挂上
opts = ClaudeAgentOptions(
    system_prompt="你是车机助手，回答一句话",
    mcp_servers={"car": car},
    allowed_tools=["mcp__car__set_ac"],                   # 免确认；名字格式 mcp__<server>__<tool>
    tools=["Read", "Grep"],                               # 自带工具按需裁剪；不写则全部带上
    max_turns=3,
)

async def main():
    async for m in query(prompt="有点热，空调调到22度", options=opts):
        if isinstance(m, SystemMessage) and m.subtype == "init":
            tools = m.data["tools"]
            print("可用工具：", tools)
        elif isinstance(m, AssistantMessage):
            for b in m.content:
                if isinstance(b, ToolUseBlock): print("调用工具：", b.name, b.input)
        elif isinstance(m, ResultMessage):
            print("最终回答：", m.result)
            print(f"轮数 {m.num_turns}，session_id 有值：{bool(m.session_id)}")

anyio.run(main)
```

**运行**：`pip install langchain claude-agent-sdk`，然后 `python car_ac_langchain.py` 和 `python car_ac_claude_sdk.py`（后者需要 Claude 的 API 凭据）。

**实测输出**（2026-10-16，第二个文件连续运行 2 次，结果一致）：

```
human: 有点热，空调调到22度
   ai: ['set_ac']
 tool: OK: 空调 22℃
   ai: 空调已调到22度。
自带工具：无，只有你给的 ['set_ac']
---
可用工具： ['Grep', 'Read', 'mcp__car__set_ac']
调用工具： mcp__car__set_ac {'temp': 22}
最终回答： 已将空调调至22度。
轮数 2，session_id 有值：True
```

**该看什么**：
- 工具逻辑一模一样，变的只是**包装**：LangChain 从类型注解和 docstring 推出 schema；Claude Agent SDK 要你显式写 `{"temp": int}`，工具名也变成 `mcp__car__set_ac`。
- LangChain 那边模型只认识你给的一个工具；Claude Agent SDK 这边即使只给了一个工具，也多出了自带的 `Read`、`Grep`。实测去掉 `tools=[...]` 那行，可用工具一下变成 40 多个（含 Bash、Edit、Write、WebSearch）——这就是「harness 自带家具」。
- Claude 这边返回的是一串**事件流**（初始化、助手消息、最终结果），结束时带 `session_id`，以后可以用它续上这次会话。

## 核心概念

### 1. 三层：运行时、框架、harness

- **是什么**：LangChain 官方的分法。**运行时**（LangGraph）负责持久化、中断恢复、流式输出；**框架**（LangChain、OpenAI Agents SDK、CrewAI）提供模型、工具、Agent 循环的统一接口；**harness**（Deep Agents、Claude Agent SDK）在此之上预装规划、子 Agent、文件系统、上下文管理。
- **没有它会怎样**：把 LangGraph 和 CrewAI 放在一起比「谁更好」，就像比 Room 和一个记账 App 谁更好——根本不在一层。
- **对应你熟悉的**：运行时 ≈ Android 系统服务（WorkManager 保证任务活下来）；框架 ≈ Jetpack（统一的积木）；harness ≈ 一个能直接改的成品 App 模板。

三层不是三选一，常常叠在一起用：LangChain 的 Agent 跑在 LangGraph 上，Deep Agents 又建在 LangChain 之上。

### 2. 循环和工具归谁管

- **是什么**：用框架时，你挑模型、写工具、组装循环，Agent 只有你给的能力。用 harness 时，循环、自带工具、权限、上下文压缩、会话恢复都已就位，你主要做**配置和裁剪**。
- **没有它会怎样**：自己写长任务 Agent，要重新造读写文件、搜索、子 Agent、上下文快满时自动压缩这一整套，很容易做漏。
- **对应你熟悉的**：自己用 `MediaCodec` 拼播放器 vs 直接用 ExoPlayer：后者开箱即用，但你得学会怎么关掉不要的功能。

Claude Agent SDK 原名 Claude Code SDK。改名是因为驱动 Claude Code 的那套 harness 不只能写代码，也能做研究、分析表格等。它的核心思路是「给 Claude 一台电脑」：终端、文件、搜索，按「收集上下文 → 行动 → 验证」循环干活。

代价是能力默认偏多。车机这种场景要像上面代码那样用 `tools` 裁掉不需要的，再用 `allowed_tools` / 权限模式控制哪些能免确认执行。

### 3. 多 Agent 的协作方式不一样

- **是什么**：同样是「交给别的 Agent」，各家语义不同：
  - **Handoff（OpenAI Agents SDK）**：控制权转移，专家接管后续对话，原 Agent 不再出场。
  - **Subagent / Agent 当工具（Claude Agent SDK、LangChain）**：主 Agent 调子 Agent，拿回结果，自己保持控制（Day 16 的主管模式）。
  - **角色团队（CrewAI）**：每个 Agent 有 role / goal / backstory，按顺序或由经理 Agent 分派任务；外层用 Flow 管状态和流程。
  - **群聊（AutoGen）**：多个 Agent 轮流发言。后继者 Microsoft Agent Framework 已改成带类型的图工作流。
- **没有它会怎样**：照搬代码迁移时，以为语义一样，结果「谁来回答用户」变了。Claude 官方迁移指南专门提醒：要「B 永久接管」，就别用 subagent，改用自己写的 Python 分发器。

### 4. 选型看五个维度

| 维度 | 要问的问题 |
|---|---|
| 层次 | 要的是可靠运行、快速搭建，还是开箱干长活？ |
| 确定性 | 流程能画成图吗？能就用图（LangGraph），不能再交给模型自主循环 |
| 模型绑定 | 只用 Claude？还是要多家模型切换？ |
| 持久化与人在回路 | 要不要暂停等确认、断点续跑？ |
| 维护状态 | 项目还在演进吗？AutoGen 已进入维护模式，新用户被引导到 Microsoft Agent Framework |

Anthropic 的一贯建议（Day 17 引过）：先直接用 API，几行代码就能写出 Agent 循环；用框架就得搞清楚它底层做了什么。

## 高频面试题

**1. 让你给新项目选 Agent 框架，你怎么选？**
- 先分层：可靠运行选 LangGraph；快速标准化选 LangChain / OpenAI Agents SDK；长任务开箱选 Claude Agent SDK / Deep Agents。
- 能写成固定流程的部分用图或代码控制，只把开放的部分交给模型循环。
- 再看模型绑定、持久化需求、团队熟悉度、项目维护状态。
- 简单场景直接用模型 API，不上框架。

**2.（追问）Claude Agent SDK 和直接用 Messages API 写工具循环，有什么区别？**
- Messages API：每轮请求、执行工具、拼回结果都要你写，最灵活。
- Agent SDK：带上 Claude Code 同款 harness，循环、自带工具、子 Agent、上下文压缩、权限、会话恢复都已实现。
- 自定义工具以进程内 MCP server 挂入，工具名为 `mcp__服务名__工具名`。
- 代价：自带能力多、权限要主动收紧；绑定 Claude。

**3.（追问）把 OpenAI Agents SDK 的 handoff 迁到 Claude Agent SDK 的 subagent，行为会变吗？**
- 会。handoff 是专家接管，原 Agent 不再运行。
- subagent 是委派，主 Agent 拿到结果后继续掌控并负责最终回答。
- 纯分诊场景差别小；要「永久接管」就用代码写分发器。
- guardrail 在 Claude 这边没有专门对象，要用普通函数或 hook 实现。

**4.（场景题）车机要做「语音调空调」和「生成周末自驾行程报告」两个能力，怎么选型？**
- 调空调：低延迟、流程确定，用规则 / 意图分类加直接工具调用，甚至不需要 Agent 循环。
- 行程报告：多步、长时、要查资料写文件，适合云端 harness（Claude Agent SDK 或 Deep Agents），工具裁剪到搜索加文件读写。
- 涉及付费预订：用 LangGraph 做确认节点和持久化（Day 17）。
- 两者共享同一套车控工具，用 MCP 暴露，避免每个框架各写一遍。

## 常见误区

1. **「选一个框架就够了，它们是互斥的」**：三层常常叠用。LangChain 的 Agent 跑在 LangGraph 上；Claude Agent SDK 也可以当成 LangGraph 里的一个节点来调。
2. **「harness 开箱即用，所以权限也安全」**：恰恰相反，自带工具越多，越要主动裁剪（`tools`）和收紧免确认范围（`allowed_tools`、权限模式）。车机上绝不能默认带着 Bash。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：用 Claude Agent SDK 定义工具、挂上、配置、跑循环**（取自 `car_ac_claude_sdk.py`，打印部分略有精简）

```python
# 声明工具：名字、给模型看的描述、参数 schema 要显式写出
@tool("set_ac", "设置车内空调温度（16~30 度）", {"temp": int})
# 工具函数是异步的，参数统一装在 args 字典里
async def set_ac(args):
    # 返回值必须是 content 列表，里面放文本块
    return {"content": [{"type": "text", "text": f"OK: 空调 {args['temp']}℃"}]}

# 把工具打包成进程内的 MCP server
car = create_sdk_mcp_server(name="car", tools=[set_ac])
# 配置 Agent：系统提示、挂哪些 server、哪些工具免确认、保留哪些自带工具
opts = ClaudeAgentOptions(
    system_prompt="你是车机助手，回答一句话",
    # 字典的 key 就是 server 名，决定工具全名的中间段
    mcp_servers={"car": car},
    # 免确认的工具，用全名
    allowed_tools=["mcp__car__set_ac"],
    # 自带工具只留两个
    tools=["Read", "Grep"],
    # 最多跑 3 轮，防止失控
    max_turns=3,
)

# 主函数：边跑边接收事件
async def main():
    # query 返回异步迭代器，一条条吐出事件
    async for m in query(prompt="有点热，空调调到22度", options=opts):
        # 遇到最终结果事件，打印回答
        if isinstance(m, ResultMessage):
            print("最终回答：", m.result)

# 用 anyio 启动异步主函数
anyio.run(main)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `@tool` 有三个参数（名字、描述、schema），和 LangChain 的 `@tool` 不同，别漏掉描述。
- 工具全名是 `mcp__car__set_ac`，两段之间都是**两个下划线**。
- `query` 要配 `async for`，不能写成普通 `for`；`anyio.run(main)` 传函数本身，不是 `main()`。

**昨日复习（只做第③步）**：Day 17 的「定义状态、写确认节点、连图、编译」。只看你自己写过的注释，重写代码，重点核对：`thread_id` 放在 `configurable` 里、恢复用 `Command(resume=...)`、`add_conditional_edges` 传函数本身。

## 动手练习（30 分钟）

在两个文件的基础上改：

1. 给两个文件都加一个工具 `get_cabin_temp()`（返回「车内 28℃」），用户说「车里多少度？太热就调到 22」。对比两边的工具写法差异，用一句话写下来。
2. 在 `car_ac_claude_sdk.py` 里删掉 `tools=[...]` 那行，打印可用工具数量；再改成 `tools=[]`，看还剩什么。
3. 从 `allowed_tools` 里去掉 `mcp__car__set_ac`，重跑，观察模型能否调用它、最终回答怎么说，用一句话解释「可用」和「免确认」的区别。

## 读后自测

1. LangChain 官方把 Agent 工具链分成哪三层？LangGraph、LangChain、Claude Agent SDK 各在哪一层？→ 见第 6 节「核心概念 1」
2. 「框架」和「harness」最大的区别是什么？为什么 Claude Agent SDK 自带 Read、Bash 这些工具？→ 见第 6 节「核心概念 2」
3. OpenAI Agents SDK 的 handoff 和 Claude 的 subagent 有什么不同？→ 见第 6 节「核心概念 3」

---

## 参考区（用时查，不必通读）

### 细节速查

**框架速览**

| 产品 | 层次 | 核心抽象 | 多 Agent 方式 | 模型 | 备注 |
|---|---|---|---|---|---|
| LangGraph | 运行时 | StateGraph、checkpoint、interrupt | 子图、Send | 任意 | 对标 Temporal、Inngest |
| LangChain | 框架 | `create_agent`、middleware | Agent 当工具 | 任意 | 跑在 LangGraph 上 |
| Deep Agents | harness | `create_deep_agent` | 子 Agent 委派 | 任意 | 自带文件读写，规划需加 `TodoListMiddleware` |
| Claude Agent SDK | harness | `query` / `ClaudeSDKClient`、`ClaudeAgentOptions` | `AgentDefinition` + Agent 工具 | Claude | 原 Claude Code SDK |
| OpenAI Agents SDK | 框架 | `Agent`、`Runner`、`@function_tool` | handoff、`as_tool()` | 默认 OpenAI，可接其他 | 自带 tracing 面板 |
| CrewAI | 框架 | Agent（role/goal/backstory）、Task、Crew、Flow | sequential / hierarchical | 任意 | 生产建议 Flow 套 Crew |
| AutoGen | 框架 | Core、AgentChat、Extensions | 群聊 | 任意 | 维护模式，不再加新功能 |
| Microsoft Agent Framework | 框架 | `Agent`、`WorkflowBuilder` | 图工作流、`as_tool()` | 多家 | AutoGen 与 Semantic Kernel 团队的后继项目 |

**OpenAI Agents SDK → Claude Agent SDK 概念映射**（来自 Claude 官方 cookbook）
- `Agent(name, instructions, tools)` → `ClaudeAgentOptions` + `system_prompt`
- `@function_tool` → `@tool` + `create_sdk_mcp_server`
- `@input_guardrail` / `@output_guardrail` → 循环前后的普通函数，或 `UserPromptSubmit` hook（拦截时 `result` 为空，拿不到拒绝理由）
- `Runner.run` → `ClaudeSDKClient` 上下文管理器（`client.query()` + `receive_response()`）
- 会话 → 复用同一个 client；跨进程用 `resume=session_id`（存在本地磁盘，可跨重启、不能跨机器）
- tracing → OpenTelemetry；`ResultMessage` 带 `total_cost_usd`、`usage`
- handoff → `agents={...}` 里的 `AgentDefinition`

**Claude Agent SDK 常用选项**：`tools`（自带工具白名单）、`allowed_tools`（免确认）、`disallowed_tools`、`permission_mode`（如 `acceptEdits`、`bypassPermissions`）、`mcp_servers`、`agents`、`hooks`、`resume`、`max_turns`、`max_budget_usd`、`can_use_tool`、`setting_sources`。

**AutoGen → Microsoft Agent Framework 映射**：`AssistantAgent` → `Agent`；`FunctionTool` → `@tool`；`AgentTool` → `agent.as_tool()`；`RoundRobinGroupChat` → `SequentialBuilder`；`MagenticOneGroupChat` → `MagenticBuilder`；`GraphFlow` → `WorkflowBuilder`。迁移指南列出 Anthropic 客户端、分布式执行等仍在规划中。

**实测环境**：langchain 1.4.6、langchain-core 1.6.11、langgraph 1.2.15、claude-agent-sdk 0.2.159，Python 3.13。实测不加 `tools` 时，本环境初始化事件列出 43 个工具，并且模型先调 `ToolSearch` 才找到 `set_ac`（工具多时会延迟加载），所以示例里做了裁剪。

### 来源说明

- 非 LangChain/Claude 来源：AutoGen（microsoft/autogen 仓库 README）、Microsoft Agent Framework 迁移指南、CrewAI 官方文档、OpenAI Agents SDK 官方文档。
- Claude Agent SDK 官方文档页（code.claude.com）和 Building Effective Agents 本次未能打开，相关内容依据 Claude 官方 cookbook、claude.com 博客和本地实测。

### 延伸学习

- Claude 官方 cookbook 的 Claude Agent SDK 系列（从 one-liner research agent 开始）
- [Runtimes, frameworks, and harnesses](https://docs.langchain.com/oss/python/concepts/products)（LangChain 官方选型页，一页读完）
- Day 19「Code Agent 与 Computer Use」会接着讲 harness 里「给 Claude 一台电脑」的那部分。

### 参考资料

- [Runtimes, frameworks, and harnesses - Docs by LangChain](https://docs.langchain.com/oss/python/concepts/products)
- [Agents - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/agents)
- [Deep Agents Quickstart - Docs by LangChain](https://docs.langchain.com/oss/python/deepagents/quickstart)
- [Building agents with the Claude Agent SDK - Claude Blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
- [Migrating from the OpenAI Agents SDK - Claude Cookbook](https://platform.claude.com/cookbook/claude-agent-sdk-04-migrating-from-openai-agents-sdk)
- [The one-liner research agent - Claude Cookbook](https://platform.claude.com/cookbook/claude-agent-sdk-00-the-one-liner-research-agent)
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)（非 LangChain/Claude 来源）
- [Agent orchestration - OpenAI Agents SDK](https://openai.github.io/openai-agents-python/multi_agent/)（非 LangChain/Claude 来源）
- [CrewAI Introduction](https://docs.crewai.com/en/introduction)、[CrewAI Crews](https://docs.crewai.com/en/concepts/crews)（非 LangChain/Claude 来源）
- [microsoft/autogen](https://github.com/microsoft/autogen)（非 LangChain/Claude 来源）
- [AutoGen to Microsoft Agent Framework Migration Guide](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/)（非 LangChain/Claude 来源）
