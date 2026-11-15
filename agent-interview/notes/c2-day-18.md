# Day 18 · 框架对比（第二轮：护栏挂在哪、权限按什么顺序算、工具怎么一次写多处用）（2026-11-15）

## 今日 AI 动态（2026-11-15）

过去 24 小时关注的厂商没有新的官方发布。

---

> 第二轮说明：第一轮 Day 18 讲过三层（运行时 / 框架 / harness）、handoff 与 subagent 的区别、选型五个维度。今天不再比「谁更好」，而是比**扩展点**：同一条业务规则在两个框架里分别挂在哪；harness 的权限检查按什么顺序走；工具怎样只写一次、两边都能用；要等车主确认很久时，两边各怎么暂停和恢复。

## 读前三问

1. 同一条「温度只能 16~28 度」的规则，在 LangChain 里挂在哪？在 Claude Agent SDK 里挂在哪？为什么不能只写进 prompt？
2. Claude Agent SDK 里 `allowed_tools` 只写了 `set_ac`，为什么不等于「只能用 `set_ac`」？必须每次都执行的检查该放在哪一步？
3. 车控工具已经用 LangChain 接过一次，现在还要接进 Claude Agent SDK，怎样才能不重写一遍？

## 从一个场景开始

上一轮的选型会开完了：语音控车走 LangChain + LangGraph，长任务「行程报告」走 Claude Agent SDK。两套跑起来以后，安全评审提了一条硬规定：**空调温度只能 16~28 度，超出直接拒绝**。

第一版是在两边的 system prompt 里各写一句「温度不要超过 28 度」。测试同学说「冷死了开到 35 度」，有一次模型真把 35 发下去了。第二版各写一段代码拦截，结果出了三个问题：

- LangChain 那边的拦截写成了同步函数，Agent 用 `ainvoke` 跑，直接报 `NotImplementedError`。
- Claude 那边把检查写在 `can_use_tool` 回调里，结果一次都没触发，因为 `set_ac` 已经在 `allowed_tools` 里，被提前批准了。
- `set_ac` 在两个项目里各实现了一遍，改一处忘一处。

这三个问题背后其实是同一个问题：**每个框架留给你插手的位置不同，检查的先后顺序也不同。**

## 一句话总结

业务规则不能靠 prompt，要挂在框架的扩展点上：LangChain 用中间件包住工具调用，Claude Agent SDK 用工具执行前的 hook，hook 在所有权限检查之前运行。工具用 MCP 只写一次，两边共用。

## 先跑起来

一个 MCP server 提供 `set_ac`，同一条规则分别挂进 LangChain（剧本模型，不需要 Key）和 Claude Agent SDK（调用真实 Claude）。

```python
# car_server.py —— 车控工具只写一次，以 MCP server 形式暴露（fastmcp 4.2.0）
from fastmcp import FastMCP

mcp = FastMCP("car")

@mcp.tool
def set_ac(temp: int) -> str:
    """设置车内空调温度（摄氏度）"""
    return f"OK: 空调 {temp}℃"

if __name__ == "__main__":
    mcp.run(show_banner=False, log_level="ERROR")  # 默认 stdio：由调用方作为子进程拉起
```

```python
# two_harness.py —— 同一个 MCP 工具 + 同一条护栏（温度 16~28），分别接进 LangChain 和 Claude Agent SDK
import anyio, sys, warnings; warnings.filterwarnings("ignore")
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain.messages import AIMessage, ToolMessage
from langchain.mcp import MCPAdapter
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher, ResultMessage, AssistantMessage, ToolUseBlock

SERVER = {"command": sys.executable, "args": ["car_server.py"]}
def too_hot_or_cold(args): return not 16 <= args.get("temp", 0) <= 28   # 两边共用的业务规则

@wrap_tool_call                                   # LangChain：护栏 = 中间件，包住每次工具执行
async def guard(request, handler):                # 用 ainvoke 跑，中间件也必须是 async
    if too_hot_or_cold(request.tool_call["args"]):
        return ToolMessage("拒绝：温度须在 16~28 之间", tool_call_id=request.tool_call["id"])
    return await handler(request)

async def pre_tool(inp, tool_use_id, ctx):        # Claude SDK：护栏 = PreToolUse hook，在权限检查之前
    if too_hot_or_cold(inp["tool_input"]):
        return {"hookSpecificOutput": {"hookEventName": "PreToolUse",
                "permissionDecision": "deny", "permissionDecisionReason": "温度须在 16~28 之间"}}
    return {}
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self
async def run_langchain():                         # 剧本模型：先要 35 度，被拒后改 28 度
    async with MCPAdapter({"mcpServers": {"car": SERVER}}) as mcp:
        model = Scripted(messages=iter([
            AIMessage("", tool_calls=[{"name": "set_ac", "args": {"temp": 35}, "id": "c1"}]),
            AIMessage("", tool_calls=[{"name": "set_ac", "args": {"temp": 28}, "id": "c2"}]),
            AIMessage("最高只能 28 度，已调到 28 度。")]))
        agent = create_agent(model, tools=await mcp.list_tools(), middleware=[guard])
        res = await agent.ainvoke({"messages": [{"role": "user", "content": "冷死了，空调开到35度"}]})
        for m in res["messages"][1:]:
            print("  LC", m.type, m.text or m.tool_calls[0]["args"])
async def run_claude():
    opts = ClaudeAgentOptions(
        mcp_servers={"car": {"type": "stdio", **SERVER}}, tools=[],          # 不带任何自带工具
        allowed_tools=["mcp__car__set_ac"], permission_mode="dontAsk", max_turns=4,
        hooks={"PreToolUse": [HookMatcher(matcher="mcp__car__set_ac", hooks=[pre_tool])]})
    async for m in query(prompt="冷死了，空调开到35度。只调用工具，回答一句话。", options=opts):
        if isinstance(m, AssistantMessage):
            [print("  CL 调用", b.input) for b in m.content if isinstance(b, ToolUseBlock)]
        if isinstance(m, ResultMessage): print("  CL 结果:", m.result, "| 轮数", m.num_turns)

anyio.run(run_langchain); anyio.run(run_claude)
```

**运行**：`pip install "langchain[mcp]" claude-agent-sdk`，两个文件放同一目录，执行 `python two_harness.py`（Claude 部分需要 API 凭据）。

**实测输出**（2026-11-15，连续运行 2 次，LangChain 部分完全一致，Claude 部分措辞略有不同、行为一致）：

```
  LC ai {'temp': 35}
  LC tool 拒绝：温度须在 16~28 之间
  LC ai {'temp': 28}
  LC tool OK: 空调 28℃
  LC ai 最高只能 28 度，已调到 28 度。
  CL 调用 {'temp': 35}
  CL 结果: 空调没调成，35度超出了允许范围，系统只接受16~28度。要我改成最高的28度吗? | 轮数 2
```

**该看什么**：
- 两边模型都**真的发出了** 35 度的调用，拦截发生在执行之前。规则是代码，模型听不听话都一样。
- 同一个工具，LangChain 里叫 `set_ac`，Claude 里叫 `mcp__car__set_ac`。matcher 和 `allowed_tools` 都要用全名。
- 拒绝之后怎么做，由模型决定：剧本模型自己改成 28 度，真实的 Claude 选择先问车主。想要固定的「自动降级」，就得在护栏里直接改参数，不能指望模型。

## 核心概念

### 1. 扩展点：中间件和 hook

- **是什么**：框架给你留的「插手位置」。LangChain 的中间件可以挂在 Agent 开始前、每次调模型前后、每次工具调用外面；Claude Agent SDK 的 hook 挂在 harness 的事件上，比如工具执行前（`PreToolUse`）、执行后、子 Agent 结束、上下文压缩前。
- **没有它会怎样**：规则只能写进 prompt。LangChain 官方博客的原话是「你没法靠 prompt 做到合规」。
- **对应你熟悉的**：OkHttp 拦截器链。LangChain 多个中间件的执行顺序也是这样：`before_*` 按注册顺序执行，`after_*` 倒序执行，`wrap_*` 一层套一层，第一个中间件在最外层。

两者最大的区别在**谁拥有循环**。LangChain 的循环在你的进程里，中间件就是普通函数调用。Claude Agent SDK 背后跑着一个 Claude Code 进程，hook 是它回调到你进程里的函数，所以有超时：`PreToolUse` 的回调超时，这次工具调用就会被拦下。多个 `PreToolUse` hook 会并行执行，结果取最严的一个，只要有一个返回 deny 就拦截。

### 2. harness 的权限检查顺序

- **是什么**：Claude 每次要用工具，SDK 按固定顺序检查：① hook → ② deny 规则 → ③ ask 规则 → ④ 权限模式 → ⑤ allow 规则（`allowed_tools`）→ ⑥ `can_use_tool` 回调。前面任何一步做出决定，后面的步骤就不再执行。
- **没有它会怎样**：场景里的坑就是这么来的。`set_ac` 在第 ⑤ 步已被批准，第 ⑥ 步的回调根本走不到。
- **对应你熟悉的**：Android 权限。`tools` / `disallowed_tools` 相当于 Manifest 里不声明这个权限，工具对模型直接不可见；`allowed_tools` 相当于预授权，免去弹窗；`can_use_tool` 相当于运行时弹窗，只有前面都没决定时才会弹出。

三条结论：
- `allowed_tools` 只负责免确认，不是白名单。没列出的工具依然存在，要隐藏工具得用 `tools` 或 `disallowed_tools`。
- `bypassPermissions` 不受 `allowed_tools` 约束，会放行所有工具。hook 和 deny 规则仍然有效。
- 无论什么情况都必须执行的检查，放 `PreToolUse` hook。hook 返回 `allow` 也跳不过后面的 deny 和 ask 规则。

### 3. 工具和框架解耦：MCP 写一次

- **是什么**：把车控工具做成 MCP server，LangChain 通过 `langchain.mcp` 的 `MCPAdapter` 加载，Claude Agent SDK 通过 `mcp_servers` 加载，同一份代码给两个框架用。
- **没有它会怎样**：每个框架各写一遍，规则和参数校验迟早不一致。将来换框架时，工具也得跟着重写。
- **对应你熟悉的**：AIDL 定义的 Service。接口定义一次，任何客户端 bind 上来就能用。

换框架时，真正的迁移成本不在工具上，而在三处：护栏挂载点（中间件 ↔ hook）、工具命名（加不加 `mcp__服务名__` 前缀）、状态和暂停的机制（见概念 4）。所以业务规则最好写成像 `too_hot_or_cold` 这样的纯函数，两边各包一层薄适配。

### 4. 要等很久时：暂停和恢复归谁管

- **是什么**：付费预订充电位要等车主在手机上确认，可能几分钟，也可能几小时。LangChain 的 Agent 编译出来就是一个 LangGraph 图，所以用 `interrupt` 加 checkpointer 暂停，进程可以退出，恢复时传 `Command(resume=...)`。Claude Agent SDK 有两种做法：在 `can_use_tool` 回调里一直等，回调不返回，执行就一直挂着；或者让 `PreToolUse` hook 返回 `defer`，本轮结束（`stop_reason` 为 `tool_deferred`），之后用同一个会话恢复。
- **没有它会怎样**：在回调里死等，车机熄火、进程被杀，这次确认就丢了。
- **对应你熟悉的**：前台 Service 硬撑，对比把任务交给 WorkManager，状态落盘后随时可以恢复。

混合架构就是这样来的：LangChain 中间件不是一个独立的运行时，`create_agent` 的产物可以直接当作外层 `StateGraph` 的一个节点。外层用 LangGraph 管确认和持久化，内层节点里调用 Claude Agent SDK 写行程报告。一次长等待只交给一层管，不要两层都存状态。

## 高频面试题

**1. 「温度不超过 28 度」这条规则，你在 Agent 里怎么实现？**
- 不只靠 prompt。模型可能不听，实测两边模型都发出了 35 度的调用。
- LangChain：`wrap_tool_call` 中间件，不合规就直接返回 `ToolMessage`，不调用 `handler`。
- Claude Agent SDK：`PreToolUse` hook 返回 `deny` 并写明原因，原因会交给模型。
- 规则本身写成纯函数，两边共用，并配单元测试。

**2.（追问）`allowed_tools` 只写了 `set_ac`，Claude 还能调用别的工具吗？`can_use_tool` 为什么没触发？**
- 能。`allowed_tools` 只是预批准；要隐藏工具，用 `tools` 收窄或用 `disallowed_tools` 移除。
- 检查顺序是 hook → deny → ask → 模式 → allow → 回调，被 allow 规则批准的调用不会走到回调。
- `bypassPermissions` 下所有工具都会放行，只有 hook 和 deny 规则还能拦。
- 必查的规则放 `PreToolUse`；无头运行用 `dontAsk`，未批准的直接拒绝。

**3.（追问）护栏写好了，Agent 用 `ainvoke` 一跑就报 `NotImplementedError`，为什么？**
- 用 `@wrap_tool_call` 包的是同步函数，只有同步实现，异步调用路径找不到对应版本。
- 改成 `async def` 加 `await handler(request)`，或者继承 `AgentMiddleware` 同时实现同步和异步两个版本。

**4.（场景题）已有 LangGraph 车机流程，要接入 Claude Agent SDK 写行程报告，车控工具不能重复实现，预订充电位要车主确认，确认可能要等几小时。怎么设计？**
- 车控工具做成一个 MCP server，两边都加载；业务规则写成纯函数，两边各包一层。
- 外层 LangGraph 负责「确认 → 预订」：`interrupt` 加持久化的 checkpointer，进程可以退出。
- 行程报告作为一个节点，内部调用 Claude Agent SDK，`tools` 只保留需要的几个，再配 `dontAsk`。
- 预订节点保持幂等（第二轮 Day 17），等待状态只存在外层这一份。

## 常见误区

1. **「`allowed_tools` 就是白名单」**：它只决定免不免确认。真正收窄可用工具要用 `tools` / `disallowed_tools`，尤其别在 `bypassPermissions` 下指望靠它限制工具。
2. **「中间件写同步还是异步都一样」**：同步写的 `@wrap_tool_call` 在 `ainvoke` / `astream` 下会直接报错。用异步就要写 `async def`，或者同时实现同步和异步两个版本。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：同一条护栏的两种挂法**（取自 `two_harness.py`，略有精简）

```python
# 业务规则写成纯函数：温度不在 16~28 之间就算违规
def too_hot_or_cold(args):
    # 取不到温度时按 0 处理，也会被判违规
    return not 16 <= args.get("temp", 0) <= 28

# LangChain：用装饰器把函数变成「包住工具调用」的中间件
@wrap_tool_call
# 异步版本，参数是本次调用请求和「真正执行工具」的函数
async def guard(request, handler):
    # 从请求里取出模型给的参数做检查
    if too_hot_or_cold(request.tool_call["args"]):
        # 违规：不执行工具，直接伪造一条工具结果给模型
        return ToolMessage("拒绝：温度须在 16~28 之间", tool_call_id=request.tool_call["id"])
    # 合规：交给下一层，真正执行工具
    return await handler(request)

# Claude SDK：工具执行前的 hook，三个参数是事件数据、调用 ID、上下文
async def pre_tool(inp, tool_use_id, ctx):
    # 参数在 tool_input 里
    if too_hot_or_cold(inp["tool_input"]):
        # 违规：在 hookSpecificOutput 里给出 deny 和原因
        return {"hookSpecificOutput": {"hookEventName": "PreToolUse",
                "permissionDecision": "deny", "permissionDecisionReason": "温度须在 16~28 之间"}}
    # 返回空字典：不干预，交给后面的权限检查
    return {}

# 注册 hook：事件名做 key，matcher 用工具全名
hooks = {"PreToolUse": [HookMatcher(matcher="mcp__car__set_ac", hooks=[pre_tool])]}
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `ToolMessage` 必须带 `tool_call_id`，取自 `request.tool_call["id"]`，漏了模型就对不上是哪次调用的结果。
- `permissionDecision` 要放在 `hookSpecificOutput` 里面，不能放在最外层；`hookEventName` 也别漏。
- `matcher` 写工具全名 `mcp__car__set_ac`，名字里是两个下划线；只写 `set_ac` 匹配不上，hook 不会触发。

**昨日复习（只做第③步）**：第二轮 Day 17 的「并行扇出 + 重试 + 幂等预约 + 分叉」（`trip_graph.py`）。只看你自己写的注释重写，重点核对：`Annotated[list[tuple], operator.add]`、`Send("quote", {"station": st})`、`h.next == ("book",)` 里的逗号、回放和分叉时输入都传 `None`。

## 动手练习（30 分钟）

在 `two_harness.py` 的基础上改：

1. 把 `guard` 改回同步的 `def`，去掉 `await`，记下报错的第一行，再改回来。
2. 把 Claude 部分的 hook 改成「自动降级」：温度超过 28 时返回 `permissionDecision: "allow"`，并把 `updatedInput` 设为 `{"temp": 28}`。对比模型的回答：它知道温度被改过吗？用一句话写下这样做的风险。
3. 删掉 `hooks=...` 这一行，把 `allowed_tools` 清空，保留 `dontAsk`，重跑。观察 `set_ac` 还能不能执行，用权限检查顺序解释原因。

## 读后自测

1. 同一条「温度只能 16~28 度」的规则，在 LangChain 里挂在哪？在 Claude Agent SDK 里挂在哪？为什么不能只写进 prompt？→ 见第 7 节「核心概念 1」
2. `allowed_tools` 只写了 `set_ac`，为什么不等于「只能用 `set_ac`」？必须每次都执行的检查该放在哪一步？→ 见第 7 节「核心概念 2」
3. 车控工具已经用 LangChain 接过一次，现在还要接进 Claude Agent SDK，怎样才能不重写一遍？→ 见第 7 节「核心概念 3」

## AI 历史知识点：AutoGen 多智能体对话框架（2023，微软研究院等）

**是什么**：一个开源的 LLM 应用框架。核心抽象是「可对话的 Agent」：每个 Agent 可以由 LLM、人类输入、工具或三者的组合驱动，Agent 之间靠互发消息协作完成任务。论文把这种写法叫「对话编程」：开发者用自然语言和代码定义 Agent 之间怎么对话，任务流程就是对话流程。

**何时由谁提出**：论文《AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation》，v1 于 2023 年 8 月 16 日提交到 arXiv，作者包括 Qingyun Wu、Gagan Bansal、Chi Wang 等 14 人，主要来自微软研究院和合作高校。

**为什么重要**：它是最早一批把「多 Agent 协作」做成通用框架的项目之一，「助手 Agent + 代表用户执行代码的代理 Agent」这种两人对话模式影响很大。论文在数学、编程、问答、运筹优化、在线决策等多个领域做了实验。之后 CrewAI 的角色团队、各家的 group chat 设计，都能看到它的影子。

**和今天的联系**：2025 年 10 月，AutoGen 维护者宣布 AutoGen 与 Semantic Kernel 合并为 Microsoft Agent Framework。AutoGen 之后只修关键 bug 和安全问题，不再加大功能。从「自由群聊」走到「带类型的图工作流」，和今天的结论方向一致：可控性要靠框架提供的扩展点和确定性结构，不能只靠 Agent 之间自由对话。这也说明选框架时要看项目的维护状态。

**原始来源**：[arXiv 2308.08155](https://arxiv.org/abs/2308.08155v2) · [AutoGen Update（GitHub Discussion #7066）](https://github.com/microsoft/autogen/discussions/7066)

---

## 参考区（用时查，不必通读）

### 细节速查

**扩展点对照**

| 时机 | LangChain 中间件 | Claude Agent SDK hook（Python 可用） |
|---|---|---|
| 整个运行开始 / 结束 | `before_agent` / `after_agent` | `UserPromptSubmit` / `Stop` |
| 每次调模型 | `before_model`、`after_model`、`wrap_model_call` | 无直接对应（模型调用在 harness 内部） |
| 每次调工具 | `wrap_tool_call` | `PreToolUse`、`PostToolUse`、`PostToolUseFailure` |
| 子 Agent | Agent 当工具时在外层处理 | `SubagentStart`、`SubagentStop` |
| 上下文压缩 | `SummarizationMiddleware`（内置） | `PreCompact` |
| 需要审批时 | `HumanInTheLoopMiddleware`（按工具 `.name` 匹配） | `PermissionRequest`、`Notification` |

- `PreToolUse` 的决定有四种：`allow`、`deny`、`ask`、`defer`；多个 hook 结果不同时，优先级是 deny > defer > ask > allow。`updatedInput` 和 `defer` 同时用时会被忽略。
- hook 默认超时：大多数事件 600 秒；`UserPromptSubmit` 30 秒。`PreToolUse` 超时等同于拦截。

**权限模式**

| 模式 | 行为 |
|---|---|
| `default` | 没有模式层面的自动批准；需要审批的调用交给 `can_use_tool` |
| `dontAsk` | 本来要弹确认的一律拒绝，不调用回调；适合无头运行 |
| `acceptEdits` | 自动批准工作目录内的文件编辑和 `mkdir`、`rm`、`mv` 等 |
| `bypassPermissions` | 几乎全部放行，`allowed_tools` 约束不了它；hook 和 deny 规则仍然有效 |
| `plan` | 只探索、做计划，文件编辑总要走回调 |
| `auto` | 由分类模型逐个审核命令和网络请求 |

**其他**
- `disallowed_tools=["Bash"]`：从请求里移除这个工具定义；`["Bash(rm *)"]`：Bash 保留，只拒绝匹配的命令。
- `can_use_tool` 返回 `PermissionResultAllow(updated_input=...)` 或 `PermissionResultDeny(message=...)`；Python 用单字符串的 `query` 时，官方示例需要流式输入加一个占位 `PreToolUse` hook，才能让回调生效。
- `langchain.mcp` 要求 `langchain[mcp]>=1.4.0`，目前是 beta，导入时会有 `LangChainBetaWarning`；它取代了独立包 `langchain-mcp-adapters`（`MultiServerMCPClient` → `MCPAdapter`）。
- 实测环境：langchain 1.5.0、langchain-core 1.8.0、langgraph 1.2.22、fastmcp 4.2.0、claude-agent-sdk 0.2.172，Python 3.13。

### 来源说明

- 「AI 历史知识点」用的是 arXiv 论文页和 microsoft/autogen 的 GitHub Discussion，属于非 LangChain/Claude 来源。
- 本期动态检索时 OpenAI 官网无法打开，Qoder 和 Hugging Face 页面也没能打开，这几家未能核实。

### 延伸学习

- Claude Academy《AI-native SDLC playbook》的 Hooks as approval gates 一课：讲用 hook 做审批关卡，团队 hook 放进 `.claude/settings.json`，不可关闭的放在托管设置。
- LangChain 博客 How Middleware Lets You Customize Your Agent Harness：把中间件分成合规、动态控制、上下文管理、生产可靠性、工具集五类。

### 参考资料

- [Configure permissions - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/permissions)
- [Intercept and control agent behavior with hooks - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/hooks)
- [Handle approvals and user input - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/user-input)
- [Hooks as approval gates - Claude Academy](https://academy.claude.com/courses/ai-native-sdlc-playbook/hooks-as-approval-gates)
- [Middleware overview - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/overview)
- [Custom middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [Migrate from langchain-mcp-adapters - Docs by LangChain](https://docs.langchain.com/oss/python/migrate/langchain-mcp-adapters)
- [How Middleware Lets You Customize Your Agent Harness - LangChain Blog](https://www.langchain.com/blog/how-middleware-lets-you-customize-your-agent-harness)
