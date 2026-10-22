# Day 24 · 护栏与人在回路（2026-10-22）

## 读前三问

1. 护栏可以放在 Agent 的哪几个位置？确定性护栏和模型型护栏各适合拦什么？
2. 人工审批弹得越多越安全吗？怎么决定「哪些动作要问人」？
3. 什么叫可逆性设计？一个车机 Agent 的动作，你会怎么按可逆性分级？

## 从一个场景开始

昨天我们用人工审批拦住了「被短信骗去发位置」。上线试用一周，车主反馈了两件事。

第一件：「后排孩子睡了，空调调到 32 度」，助手照做了。没有任何注入，就是用户说的话本身有点过头：32 度对孩子不合适。直接拒绝不友好，照做又有风险，最好是**让车主改一下数字再执行**。

第二件：测试同学说「帮我关掉 ESP」，模型居然认真地去找能关 ESP 的工具。这种请求根本不该进入模型，应该**在门口就挡掉**。

还有一个产品经理提的：每次调空调都弹窗确认，太烦了，大家都是闭眼点「同意」。

这三件事指向同一个主题：**护栏**（guardrail，在 Agent 各个关口做检查的程序）和**人在回路**（human-in-the-loop，让人在关键步骤参与决定）。今天讲怎么把它们放对位置、放对数量。

## 一句话总结

护栏是在输入、工具调用、输出三个关口做的检查，能用规则就别用模型；人工审批只放在不可逆、外发数据、越界参数的动作上，并允许人改参数；动作本身尽量设计成可撤销的。

## 先跑起来

把三件事放进一个文件：输入护栏挡掉「关 ESP」；空调温度只有超出 18–28 度才弹审批（条件审批），车主把 32 改成 26 再放行；`set_ac` 执行前记下旧值，留一条撤销记录。无 API Key，模型决策用剧本代替；中间件、中断和恢复都是真实库代码。

```python
# car_hitl.py —— 车机 Agent 三道护栏：输入拦截 + 条件审批(可改参数) + 撤销记录（langchain 1.4.7）
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware, before_agent
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI
from langchain_core.tools import tool
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command

car, undo_log = {"ac": 22}, []  # 车辆状态 + 撤销记录（可逆性设计）

@tool
def set_ac(temp: int) -> str:
    """设置空调温度"""
    undo_log.append(("ac", car["ac"]))  # 执行前先记下旧值，出错能一键恢复
    car["ac"] = temp
    return f"空调已设为 {temp} 度"

@before_agent(can_jump_to=["end"])  # 第一道：确定性输入护栏，命中就直接结束
def block_safety(state, runtime):
    if "ESP" in state["messages"][-1].content:
        return {"messages": [AI("ESP 属于行车安全系统，语音助手无权关闭。")], "jump_to": "end"}

def tc(name, **args): return AI("", tool_calls=[{"name": name, "args": args, "id": name}])
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self

model = Scripted(messages=iter([tc("set_ac", temp=32), AI("已为您调好空调。")]))
hitl = HumanInTheLoopMiddleware(interrupt_on={"set_ac": {
    "allowed_decisions": ["approve", "edit", "reject"],
    "when": lambda req: not 18 <= req.tool_call["args"]["temp"] <= 28}})  # 第二道：只有越界才问人
agent = create_agent(model, tools=[set_ac], middleware=[block_safety, hitl], checkpointer=InMemorySaver())

r = agent.invoke({"messages": [{"role": "user", "content": "帮我关掉 ESP"}]}, {"configurable": {"thread_id": "a"}})
print("① 输入护栏：", r["messages"][-1].content, "| 模型被调用了吗？", "否" if len(r["messages"]) == 2 else "是")

cfg = {"configurable": {"thread_id": "b"}}
r = agent.invoke({"messages": [{"role": "user", "content": "后排孩子睡了，空调调到 32 度"}]}, cfg)
print("② 待审批：", r["__interrupt__"][0].value["action_requests"][0]["args"])
# 车主在屏幕上把 32 改成 26 再放行
r = agent.invoke(Command(resume={"decisions": [{"type": "edit",
    "edited_action": {"name": "set_ac", "args": {"temp": 26}}}]}), cfg)
print("③ 工具结果：", r["messages"][-2].content)
print("④ 车辆状态：", car, "撤销记录：", undo_log)
```

**运行**：`pip install langchain && python car_hitl.py`

**实测输出**（2026-10-22，Python 3.13，langchain 1.4.7）：

```
① 输入护栏： ESP 属于行车安全系统，语音助手无权关闭。 | 模型被调用了吗？ 否
② 待审批： {'temp': 32}
③ 工具结果： Note: a human reviewer replaced this tool call before it ran. The call recorded in your message is the one you produced, not the one that executed. This was intentional and authorized. Do not re-issue your original call. Executed instead: set_ac with arguments {"temp": 26}.

Tool response:
空调已设为 26 度
④ 车辆状态： {'ac': 26} 撤销记录： [('ac', 22)]
```

**该看什么**：
- ①：消息只有 2 条（用户 + 拒绝回复），模型一次都没被调用。输入护栏省掉了一次模型调用，也不给模型犯错的机会。
- ③：车主改了参数后，框架会在工具结果前**自动加一段说明**，告诉模型「人改过了，别再重发原来的调用」。没有这段话，模型可能以为自己没成功，又发一次 32 度。
- ④：真正执行的是 26 度；撤销记录里留着旧值 22，想回滚只要把它写回去。

## 核心概念

### 1. 护栏的三个关口

- **是什么**：护栏可以放在三个位置：**输入**（请求进模型之前）、**工具调用**（动作执行之前）、**输出**（回复给用户之前）。每个位置可以用规则（关键词、正则、数值范围）检查，也可以用另一个模型检查。
- **没有它会怎样**：所有判断都压在主模型身上，它既要干活又要自查，两头都做不好。Anthropic 在《Building effective agents》里也说，让一个模型实例处理请求、另一个并行做筛查，通常比同一次调用兼顾两件事效果更好。
- **对应你熟悉的**：OkHttp 的拦截器链：请求拦截器、响应拦截器各管一段，按顺序执行。

选型原则：**能用规则就用规则**。规则快、便宜、结果确定，适合「ESP」「温度范围」这类边界清楚的检查；模型型护栏能看懂语义，但更慢、更贵，还会误判，留给「这段回复是否不当」这类规则写不出来的判断。LangChain 里中间件按列表顺序执行，可以叠成多层。

### 2. 审批点：问谁、问什么、问几次

- **是什么**：在工具执行前停下来，把「要做什么、参数是什么」展示给人，由人决定。人的决定有四种：**批准**、**改参数后批准**、**拒绝并说明原因**、**直接替工具给答案**（只用于「问用户」类工具）。
- **没有它会怎样**：要么全自动出事，要么全手动没人用。
- **对应你熟悉的**：Android 运行时权限弹窗，以及「仅本次允许 / 始终允许 / 拒绝」三个选项。

关键是**问得少、问得准**。Anthropic 统计 Claude Code 用户会批准 93% 的权限弹窗，结果是人不再认真看自己批准了什么。所以：
- 按条件触发：今天的代码只在温度越界时才问（`when`）。
- 让人能**改**而不只是「同意 / 拒绝」：改参数比整个拒掉更友好。
- 把审批往上提一层：Claude Code 的 Plan Mode 让人先审整份计划，而不是每一步都审。

### 3. 中断与恢复：工程上的坑

- **是什么**：审批靠「中断」实现：图运行到某处暂停，状态存进 checkpointer（状态存档器），等人回复后用同一个 `thread_id` 恢复。
- **没有它会怎样**：等人回复期间进程一直挂着；车主十分钟后才点，服务早超时了。
- **对应你熟悉的**：`onSaveInstanceState` 存状态，进程被杀后再恢复界面。

面试容易追问的一点：LangGraph 恢复时，**中断所在的节点会从头再跑一遍**。所以中断之前不能有非幂等的副作用（比如先发短信、再问人），否则恢复后会再发一次。官方规则还包括：不要用宽泛的 `try/except` 包住 `interrupt()`（它靠抛异常实现暂停）；同一节点里多个中断的顺序要固定（按序号匹配回复）。

### 4. 可逆性设计

- **是什么**：设计动作时就考虑「做错了能不能撤回」。按可逆性分级：**可逆**（调空调、改导航）自动执行并留撤销记录；**难撤回**（发消息、付款、删数据）必须审批或先预览；**不可接受**（关 ESP）直接禁止。
- **没有它会怎样**：审批是「事前」防线，人也会看走眼。动作可撤销，就多了一道「事后」防线。
- **对应你熟悉的**：数据库事务回滚、Gmail 的「撤销发送」（其实是延迟几秒再真正发出）。

常用手法：执行前记录旧值（今天的 `undo_log`）、为每个写操作准备一个补偿操作、对外发动作先给预览或延迟发送。Claude Agent SDK 的文件检查点（file checkpointing）也是这个思路：用 Write / Edit 改文件前先备份，可以回滚。但它不跟踪 Bash 命令改的文件，也撤不回远程操作，这正说明可逆性要在工具设计时就考虑，不能指望框架兜底。

## 高频面试题

**1. 护栏一般放在哪些位置？怎么选规则还是模型？**
- 输入、工具调用前、输出三个关口，按顺序叠加。
- 边界清楚的（黑名单、数值范围、PII 格式）用规则：快、便宜、结果确定。
- 语义判断（回复是否不当、是否跑题）用小模型，接受延迟和误判。
- 便宜的放前面，命中就提前结束，省掉后面的调用。

**2.（追问）审批弹太多用户闭眼点同意，你怎么优化？**
- 按风险触发：只对不可逆、外发数据、参数越界的调用审批（条件审批）。
- 提供「改参数」和「本类操作以后不再问」，减少重复弹窗。
- 审批上移到计划层：先批整份计划，执行中只在偏离计划时再问。
- 用分类器替代部分人工审批（如 Claude Code 的 auto mode），并统计误放率。

**3.（场景题）车主说「把车窗全打开，再给老婆发消息说我快到了」，你怎么设计护栏和审批？**
- 输入关口：行驶中车速超过阈值时，开窗请求用规则限制开度或拒绝。
- 开窗可逆：直接执行，留撤销记录，回复里告诉车主「说『关窗』即可恢复」。
- 发消息难撤回：展示收件人和正文，允许车主改正文后再发；收件人必须在通讯录里。
- 审批结果、被改的参数写进 trace，供 Day 22 的监控和 Day 21 的评测使用。

**4.（追问）LangGraph 中断恢复后，为什么有时同一条短信被发了两次？**
- 恢复时中断所在的节点从头重跑，中断前的代码会再执行一次。
- 如果「发短信」写在 `interrupt()` 之前，就会重复发送。
- 解决：副作用放到中断之后或单独节点；或者给副作用加幂等键。
- 另外别用宽泛 `try/except` 包住 `interrupt()`，否则暂停会失效。

## 常见误区

1. **「人工审批是最后一道防线，加上就安全了」**：人也会疲劳、会看走眼，93% 的批准率说明审批很容易流于形式。审批之外还要有权限收窄和可撤销设计。
2. **「改参数时直接改掉工具调用就行，不用告诉模型」**：模型不知道参数被改，可能认为自己的调用失败了而重发，或者回复里说错数值。LangChain 默认会在工具结果前附上说明，自己实现时也要保留这一步。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：输入护栏 + 条件审批 + 改参数后恢复**（取自 `car_hitl.py`）

```python
# 声明一个「智能体开始前」运行的钩子，允许它直接跳到结束
@before_agent(can_jump_to=["end"])
# 钩子函数接收当前状态和运行时
def block_safety(state, runtime):
    # 看最新一条用户消息里有没有提到 ESP
    if "ESP" in state["messages"][-1].content:
        # 命中：塞一条拒绝回复，并跳到结束，模型不会被调用
        return {"messages": [AI("ESP 属于行车安全系统，语音助手无权关闭。")], "jump_to": "end"}

# 创建审批中间件，只针对 set_ac 这个工具配置
hitl = HumanInTheLoopMiddleware(interrupt_on={"set_ac": {
    # 允许审批人做的三种决定：批准、改参数、拒绝
    "allowed_decisions": ["approve", "edit", "reject"],
    # 条件审批：温度不在 18 到 28 度之间才弹出审批
    "when": lambda req: not 18 <= req.tool_call["args"]["temp"] <= 28}})
# 组装 Agent：输入护栏放在审批前面，并给一个 checkpointer 保存暂停状态
agent = create_agent(model, tools=[set_ac], middleware=[block_safety, hitl], checkpointer=InMemorySaver())

# 用 Command 恢复，决定类型是 edit
r = agent.invoke(Command(resume={"decisions": [{"type": "edit",
    # 改后的调用：工具名不变，参数把温度改成 26
    "edited_action": {"name": "set_ac", "args": {"temp": 26}}}]}), cfg)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 跳转写两处：装饰器参数 `can_jump_to=["end"]`（列表），返回值里 `"jump_to": "end"`（字符串）。
- `when` 拿到的是请求对象，参数在 `req.tool_call["args"]` 里，不是 `req["args"]`。
- edit 的结构是 `{"type": "edit", "edited_action": {"name": ..., "args": {...}}}`：`edited_action` 里**工具名也要写**。

**昨日复习（只做第③步）**：Day 23 的「给敏感工具挂人工审批 → 跑到中断 → 拒绝后恢复」。只看你自己写的注释，重写 `interrupt_on`、取 `r["__interrupt__"][0].value["action_requests"][0]` 和 `reject` 恢复那几行，重点核对：两次 `invoke` 用同一个 `cfg`。

## 动手练习（30 分钟）

在 `car_hitl.py` 上改：

1. 把剧本里的温度改成 24，确认这次**不弹审批**、直接执行（验证条件审批）。
2. 写一个 `undo_last()` 函数：从 `undo_log` 取出最后一条，把旧值写回 `car`。执行后打印车辆状态，应恢复成 22 度。
3. 加一个 `after_agent` 输出护栏：如果最终回复里出现的温度数字和 `car["ac"]` 不一致，就把回复改成「已为您设置为 X 度」（X 取实际值）。思考：为什么这一步用规则就够，不需要模型？

## 读后自测

1. 护栏可以放在哪几个位置？确定性和模型型护栏各拦什么？→ 见核心概念 1、面试题 1
2. 审批弹得越多越安全吗？怎么决定哪些动作要问人？→ 见核心概念 2、面试题 2
3. 什么叫可逆性设计？车机动作怎么分级？→ 见核心概念 4、面试题 3

---

## 参考区（用时查，不必通读）

### 细节速查

**LangChain `HumanInTheLoopMiddleware`（langchain 1.4.x）**
- `interrupt_on` 值：`True`（全部决定可用）、`False`（不拦）、或配置字典：`allowed_decisions`、`description`、`when`（需 `langchain>=1.3.3`）。
- 决定类型：`approve`、`edit`（`edited_action: {name, args}`）、`reject`（可带 `message`）、`respond`（人的回复作为工具结果，工具不执行；不要用它拒绝有副作用的工具，模型会当成功）。
- `edit_notice`：改参数后附在工具结果前的说明，设为 `None` 关闭；官方建议改动保守，大改可能让模型重新规划、重复执行。
- 模式：默认批量（一次中断列出本轮所有待审调用，`decisions` 列表顺序与 `action_requests` 一致）；`interrupt_mode="per_call"`（需 `langchain>=1.4.6`）每个调用单独中断，按 interrupt id 回复。
- `when` 在恢复时会再跑一次，同一调用必须返回同样结果；per-call 模式下审批中间件应放在重试类中间件之前。
- 流式：`stream_events(..., version="v3")`，看 `stream.interrupted` / `stream.interrupts`。

**LangChain 自定义护栏**
- `@before_agent` / `@after_agent`，配 `can_jump_to=["end"]`，返回 `"jump_to": "end"`。
- 官方分层示例顺序：确定性输入过滤 → PII 中间件 → 人工审批 → 模型型输出检查。

**LangGraph `interrupt()` 规则**
- 恢复：`Command(resume=...)` + 同一 `thread_id`；恢复值即 `interrupt()` 的返回值。
- 节点从头重跑；中断前副作用须幂等；不要宽泛 `try/except`；中断顺序固定；传 JSON 可序列化值；校验输入用条件边回环，别在节点里 `while True`。

**Claude Agent SDK**
- `can_use_tool(tool_name, input_data, context)` 返回 `PermissionResultAllow(updated_input=...)` 或 `PermissionResultDeny(message=...)`；可「批准并修改参数」（Claude 不会被告知改过）、「批准并记住」（`updated_permissions` 回传 `context.suggestions`）。
- 被允许规则或 `acceptEdits` / `bypassPermissions` 等模式自动放行的调用不会触发 `can_use_tool`；必须对每次调用生效的逻辑用 `PreToolUse` hook。等人时间很长时，hook 可返回 `defer`，进程退出后从会话恢复。
- `AskUserQuestion`：每次 1–4 个问题，每题 2–4 个选项；子智能体里暂不可用。
- 文件检查点：`enable_file_checkpointing=True` + `extra_args={"replay-user-messages": None}`，用用户消息 UUID 调 `rewind_files()`；只跟踪 Write / Edit / NotebookEdit，不含 Bash、子智能体（例外见文档）、目录操作、远程文件；不回滚对话本身。

**Claude Code auto mode（分类器替代人工审批）**
- 输入层：服务端注入探测扫描工具输出；输出层：Sonnet 4.6 转录分类器在执行前审查动作，只看用户消息和工具调用。
- 两阶段：快速初筛（偏向拦截）→ 被标记时再推理复核。只读工具和项目内文件编辑跳过分类器。
- 连续 3 次或累计 20 次被拒后升级给人；无界面模式直接终止。
- 数据：真实流量误拦率 0.4%；真实「过度主动」动作漏放率 17%（官方称为「诚实的数字」）。

### 来源说明

- 示例代码的模型决策是剧本，未调用真实 Claude。
- 本篇全部来自 LangChain 和 Anthropic 官方资料。

### 延伸学习

- Claude Academy · The AI-native SDLC playbook ·「Hooks as approval gates」（用 hook 做审批关卡）
- Claude Academy · Claude Code in action ·「Permission modes」
- LangChain 文档 · Human-in-the-loop、Guardrails

### 参考资料

- [Human-in-the-loop - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- [Guardrails - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/guardrails)
- [Interrupts - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [Handle approvals and user input - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/user-input)
- [Rewind file changes with checkpointing - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/file-checkpointing)
- [How we built Claude Code auto mode - Anthropic](https://www.anthropic.com/engineering/claude-code-auto-mode)
- [Trustworthy agents in practice - Anthropic](https://www.anthropic.com/research/trustworthy-agents)
- [Building effective AI agents - Anthropic](https://www.anthropic.com/research/building-effective-agents)
- [Hooks as approval gates - Claude Academy](https://academy.claude.com/courses/ai-native-sdlc-playbook/hooks-as-approval-gates)
