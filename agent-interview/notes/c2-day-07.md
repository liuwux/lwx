# Day 7 · 规划（第二轮：计划给谁看、计划即状态、先审后做）（2026-11-04）

## 今日 AI 动态（2026-11-04）

### 趋势观察

本期两条官方动态都和「AI 生成的内容和使用权限要能被管住」有关：Anthropic 把文本水印补到了 8 月之前发布的老模型上；Google 把 K-12 学生账号的 Gemini 和搜索访问改成默认关闭，由学校管理员决定是否开启。推测：合规和管控正在从「事后声明」变成「默认开启的产品机制」，做企业 Agent 时，水印、审计、默认权限这类能力会越来越多地由平台直接提供。

### 今日要点

- Claude 平台：Mythos 5、Opus 4.7、Sonnet 4.6 生成的文本从 11 月 3 日起也带上 Anthropic 文本水印。
- Google：Guided Learning 进入 Google Classroom 和搜索（搜索部分先在美国部分学校试点），K-12 学生账号的 Gemini 与搜索访问将改为默认关闭、由管理员开启。

### 分公司动态

#### 1. Anthropic / Claude

**[Claude Platform release notes：文本水印扩展到更早发布的模型](https://platform.claude.com/docs/en/release-notes/overview)**（2026-11-03）

发布记录写明：Claude Mythos 5、Opus 4.7、Sonnet 4.6 生成的文本现在带有 Anthropic 文本水印。Sonnet 4.6 上如果请求把 `temperature`、`top_p` 或 `top_k` 设成非默认值，生成的文本可能不带水印。8 月的官方说明里提到，水印做法是 Google DeepMind SynthID-Text 的一个变体：在几个词同样合适时，用密钥而不是普通随机数来选词，不加隐藏字符，也不增加 token；检测 API 目前只对符合条件的机构做私有预览。

**为什么重要**：如果你的 Agent 把 Claude 生成的文本发给用户或下游系统，现在要知道这些文本可能被检测为「很可能出自 Claude」，而改采样参数会影响部分模型上的水印。

#### 2. Google

**[Responsible AI for schools with Guided Learning](https://blog.google/products-and-platforms/products/education/ai-learning-era-updates-2026/)**（2026-11-03）

Guided Learning 是一个分步辅导功能：学生账号用搜索做题时，系统一步一步引导解题，而不是直接给出最终答案。Classroom 里的 Guided Learning 即日起全球可用，老师可以把它布置成一对一学习任务，并实时看到学生卡在哪里；搜索部分本月晚些时候在美国部分学校试点。同时，K-12 学生账号的 Gemini、Gemini Notebook、Classroom 中的 Gemini 和 Google 搜索的访问将改为默认完全关闭，需要学校管理员明确开启，Google 会在正式推出前公布细节。

**为什么重要**：这是大厂把「AI 能力默认关闭、由管理员按组织开启」做成默认设置的例子，企业 Agent 的权限设计也常要求这种「默认最小权限」。

### 动态来源

- [Claude Platform release notes - Claude Platform Docs（2026-11-03 条目）](https://platform.claude.com/docs/en/release-notes/overview)
- [How Claude's text watermarking works - Anthropic（2026-08-14，背景）](https://www.anthropic.com/news/claude-text-watermark)
- [Responsible AI for schools with Guided Learning - Google（2026-11-03）](https://blog.google/products-and-platforms/products/education/ai-learning-era-updates-2026/)

---

> 第二轮说明：第一轮 Day 7 讲过 Plan-and-Execute、ReWOO、LLMCompiler 的对比，重规划的触发条件，todo 工具和长任务进度文件。今天不重复，改讲三件进阶的事：新模型还需不需要写出来的计划；计划放进 Agent 状态之后怎么改；有副作用的任务怎么「先审计划、再放权限」。

## 读前三问

1. Claude Agent SDK 在新模型上默认不再提供 todo 工具，为什么？那你还该不该给 Agent 加 todo 工具？
2. 计划写在 Agent 状态里，执行中发现某一步不需要了，怎么改？为什么 `write_todos` 不允许并行调用？
3. 车机 Agent 要帮司机改导航、订酒店，你怎么设计「先出计划、司机确认后再执行」？

## 从一个场景开始

司机说：「周末开车去黄山，帮我安排充电和住宿。」

第一版 Agent 一口气列了三步：查续航、找沿途充电站、订酒店，然后按顺序执行。查完续航发现 520km 够跑 460km，根本不用充电，但它还是照着计划把充电站查了一遍，白白多等了几秒。

第二版改成每一步都由模型自由决定，不写计划。速度快了，但车机屏幕上什么都看不到，司机不知道它在干什么，也不知道酒店是不是已经订了。

产品经理提了两个要求：计划要能显示在屏幕上；订酒店这种有副作用的操作，要先让司机看到计划、点了确认才能执行。

今天就讲这三件事：计划是写给谁看的，计划放在哪里、怎么改，以及怎么先审后做。

## 一句话总结

新模型自己就能把多步任务做完，显式计划主要是给人看、给程序用的：把计划存进 Agent 状态，根据观察结果整张替换；有副作用的任务先在只读模式下出计划，人确认之后再放开执行权限。

## 先跑起来

用 LangChain 内置的 `TodoListMiddleware`：模型通过 `write_todos` 工具把计划写进 Agent 状态，查完续航后删掉「找充电站」这一步。用脚本化的假模型，不需要 API Key。

```python
# plan_as_state.py —— 计划存进状态、根据观察结果改计划（脚本化假模型，无需 API Key）
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware
from langchain.messages import AIMessage
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel

@tool
def get_range(dest: str) -> str:
    """查询当前续航和到目的地的距离。"""
    return f"续航 520km，到{dest} 460km，不需要中途充电。"

@tool
def book_hotel(city: str) -> str:
    """预订酒店（有副作用，不可重复执行）。"""
    return f"已预订{city}酒店，订单号 H001。"

class ScriptedModel(GenericFakeChatModel):
    def bind_tools(self, tools, **kw):
        return self
def call(name, args, i, text=""):  # 一条 AIMessage = 一句说明 + 一个工具调用
    return AIMessage(text, tool_calls=[{"name": name, "args": args, "id": f"c{i}"}])
def todos(*items):  # items: (内容, 状态)
    return {"todos": [{"content": c, "status": s} for c, s in items]}
script = [
    call("write_todos", todos(("查续航", "in_progress"), ("找沿途充电站", "pending"), ("订酒店", "pending")), 0, "先列计划。"),
    call("get_range", {"dest": "黄山"}, 1),
    call("write_todos", todos(("查续航", "completed"), ("订酒店", "in_progress")), 2, "续航够，删掉充电站这一步。"),
    call("book_hotel", {"city": "黄山"}, 3),
    call("write_todos", todos(("查续航", "completed"), ("订酒店", "completed")), 4),
    AIMessage("续航够直达黄山，不用中途充电；酒店已订好，订单号 H001。"),
]
agent = create_agent(ScriptedModel(messages=iter(script)), tools=[get_range, book_hotel],
                     middleware=[TodoListMiddleware()])
out = agent.invoke({"messages": [{"role": "user", "content": "周末开车去黄山，帮我安排充电和住宿"}]})
for m in out["messages"][1:]:
    if m.type == "ai" and m.tool_calls:
        tc = m.tool_calls[0]
        plan = [f"{t['content']}:{t['status']}" for t in tc["args"].get("todos", [])]
        print(f"AI  {m.content} -> {tc['name']} {plan or tc['args']}")
    elif m.type == "tool" and m.name != "write_todos":
        print(f"   Obs: {m.content}")
    elif m.type == "ai":
        print(f"Answer: {m.content}")
print("state['todos'] =", [(t["content"], t["status"]) for t in out["todos"]])
```

**运行**：`pip install -U langchain langgraph && python plan_as_state.py`

**实测输出**（2026-11-04，Python 3.13，langchain 1.4.8，langgraph 1.2.19）：

```
AI  先列计划。 -> write_todos ['查续航:in_progress', '找沿途充电站:pending', '订酒店:pending']
AI   -> get_range {'dest': '黄山'}
   Obs: 续航 520km，到黄山 460km，不需要中途充电。
AI  续航够，删掉充电站这一步。 -> write_todos ['查续航:completed', '订酒店:in_progress']
AI   -> book_hotel {'city': '黄山'}
   Obs: 已预订黄山酒店，订单号 H001。
AI   -> write_todos ['查续航:completed', '订酒店:completed']
Answer: 续航够直达黄山，不用中途充电；酒店已订好，订单号 H001。
state['todos'] = [('查续航', 'completed'), ('订酒店', 'completed')]
```

**该看什么**：
- 第二次 `write_todos` 传的是整张新列表，「找沿途充电站」直接从列表里消失了。重规划就是重写一次列表，不需要单独的 Replan 节点。
- 最后一行从 `out["todos"]` 读计划，而不是从消息文本里解析。车机 UI 可以直接渲染这个字段。
- 这是第一轮讲的 ReWOO 做不到的：ReWOO 的计划一次写死，看不到「续航够」这个观察结果。本例的三步是预先写好的，真实模型会自己决定什么时候改计划。

## 核心概念

### 1. 计划是给谁用的：新模型不靠它也能做完

- **是什么**：Claude Agent SDK 文档写明，`TodoWrite` 和 `TaskCreate` 等任务跟踪工具默认只在 Opus 4.7、Sonnet 4.6、Haiku 4.5 及更早的模型上提供；更新的模型不写 todo 列表也能完成多步任务。LangChain 的 Deep Agents 也把 todo 改成了需要主动开启。
- **没有它会怎样**：不写计划，模型照样能做完；但人和程序都看不到进度。所以官方的建议是：只有当你的应用要读取这些调用（记录日志或渲染进度）时才开启。
- **对应你熟悉的**：`WorkManager` 任务本身不需要通知栏也能跑完，通知栏进度是给用户看的。

面试时的判断标准：要给用户展示进度、要审计、任务跨会话需要恢复，就保留显式计划；只是想让模型「更会做事」，在新模型上收益不大，而且每次写计划都要花 token 和时间。

### 2. 计划即状态：整张替换，不做增量修改

- **是什么**：`TodoListMiddleware` 给 Agent 加一个 `write_todos` 工具，并在状态里加一个 `todos` 字段。每项只有内容和状态（`pending`、`in_progress`、`completed`）。每次调用都用新列表替换整个旧列表。
- **没有它会怎样**：计划只存在于对话文本里，UI 要从文本里解析；上下文被压缩之后，计划也可能跟着丢失。
- **对应你熟悉的**：ViewModel 里的 `StateFlow<List<Todo>>`，每次 emit 一个新列表，UI 只负责渲染。

因为是整张替换，同一轮里并行调用两次 `write_todos`，就不知道该以哪次为准。所以中间件在模型返回后会检查：一轮里出现多个 `write_todos` 调用，就全部回一条错误，要求只调用一次。

### 3. 先审后做：只读模式出计划，确认后再放权限

- **是什么**：Claude Agent SDK 的 `plan` 权限模式下，Claude 可以读代码、做调研，但文件修改一律不会被自动批准，都会交给你的 `canUseTool` 回调决定。审完计划后，可以用 `set_permission_mode()` 在会话中途切到 `acceptEdits` 等模式，放开执行。
- **没有它会怎样**：Agent 一边想一边改，等你发现方向错了，改动已经发生了。Claude Academy 的说法是：在代码还不存在时审计划，比审一份已经写完的 diff 便宜得多。
- **对应你熟悉的**：运行时权限。App 先展示「为什么需要定位」，用户同意之后才拿到权限。

一个容易被问到的细节：主会话处于 plan 模式时，子 Agent 即使配置了 `acceptEdits` 也会继承 plan 模式。原因是防止「把活交给子 Agent」绕过计划审核。

### 4. 重规划时别重复有副作用的步骤

- **是什么**：重写计划时，已完成、有副作用的步骤（订酒店、发消息）要保留为 `completed`，不能被重新排成 `pending`。
- **没有它会怎样**：模型重写列表时，可能把「订酒店」又写成待办，然后再订一次。
- **对应你熟悉的**：支付请求带幂等键，重试时服务端识别出是同一笔订单。

模型写的 `completed` 只是它自己的说法，可靠的判断来自工具结果。程序可以在 `write_todos` 前后做核对：标成完成的副作用步骤，必须能在历史里找到对应的成功工具结果。这一点留给动手练习。

## 高频面试题

**Q1：新模型不需要 todo 工具也能做完多步任务，那你的 Agent 还要不要显式计划？**
- 看计划给谁用：模型自己用，新模型上收益不大，还会多花 token。
- 用户要看进度、要审计、要跨会话恢复，就保留，并让程序读取结构化的计划。
- Claude Agent SDK 的做法：新模型默认不提供，应用需要时主动开启。
- 用评测对比开启前后的成功率、步数和延迟，再做决定。

**Q2（追问）：`write_todos` 为什么一次替换整张列表？为什么不允许并行调用？**
- 整张替换让状态简单：最后一次调用就是当前计划，删除和重排都不需要额外的操作。
- 并行调用两次，无法判断哪张列表为准。
- 中间件在模型返回后检查，多次调用全部回错误，让模型重新只调一次。
- 代价是列表很长时每次都要重写整张，所以计划应该保持短小。

**Q3（场景题）：车机 Agent 要改导航、订酒店，怎么做「先出计划、司机确认再执行」？**
- 规划阶段只开放只读工具（查续航、查酒店），不开放下单和改导航。
- 计划以结构化数据输出，在屏幕上渲染成卡片，司机可以确认或修改。
- 确认后再放开写工具；行车中改用语音确认，并设超时自动取消。
- 子 Agent 必须继承同样的限制，不能借委派绕过审批。

**Q4（追问）：重规划后，Agent 又订了一次酒店，怎么防？**
- 重写计划时把已完成的副作用步骤保留为 `completed`，在 prompt 里写明不能改回。
- 程序侧核对：有成功结果的副作用调用，再次出现时直接拦截（参考第二轮 Day 6 的重复调用拦截）。
- 工具本身做幂等：同一行程、同一酒店的重复预订返回原订单号。
- 计划状态和工具结果不一致时，以工具结果为准。

## 常见误区

- **「计划写得越详细，Agent 越可靠」**：新模型不写计划也能做完多步任务；计划过长，每次整张重写都更贵，也更容易写错状态。计划的主要价值在于让人和程序看得见。
- **「plan 模式就是不让 Agent 调用任何工具」**：plan 模式下 Agent 仍然可以读文件、做调研，被拦下的是修改操作；而且拦下之后不是直接拒绝，而是交给你的审批回调决定。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：构造带计划的脚本，创建挂 TodoListMiddleware 的 Agent，并从状态读出计划**（取自 `plan_as_state.py`）

```python
# 生成一条 AI 消息：一句说明文字 + 一个工具调用
def call(name, args, i, text=""):
    return AIMessage(text, tool_calls=[{"name": name, "args": args, "id": f"c{i}"}])
# 把若干 (内容, 状态) 元组转成 write_todos 需要的参数
def todos(*items):
    return {"todos": [{"content": c, "status": s} for c, s in items]}
# 预先写好模型每一轮的输出
script = [
    # 第一轮：列出三步计划，第一步标为进行中
    call("write_todos", todos(("查续航", "in_progress"), ("找沿途充电站", "pending"), ("订酒店", "pending")), 0, "先列计划。"),
    # 第二轮：调用查续航工具
    call("get_range", {"dest": "黄山"}, 1),
    # 第三轮：根据观察结果整张重写计划，去掉充电站这一步
    call("write_todos", todos(("查续航", "completed"), ("订酒店", "in_progress")), 2, "续航够，删掉充电站这一步。"),
]
# 创建 Agent：假模型 + 两个业务工具 + 计划中间件
agent = create_agent(ScriptedModel(messages=iter(script)), tools=[get_range, book_hotel],
                     middleware=[TodoListMiddleware()])
# 运行 Agent，传入司机的请求
out = agent.invoke({"messages": [{"role": "user", "content": "周末开车去黄山，帮我安排充电和住宿"}]})
# 从状态里读出最终计划，只取内容和状态
print([(t["content"], t["status"]) for t in out["todos"]])
```

（练习片段只保留了脚本的前三轮，完整版见「先跑起来」。）

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `todos()` 返回的是 `{"todos": [...]}`，外面这层字典就是 `write_todos` 的参数，不要只返回列表。
- 每个 todo 的键是 `content` 和 `status`，不是 `title`、`state`。
- 计划从 `out["todos"]` 读，不是从 `out["messages"]` 里找。

**昨日复习（只做第③步）**：第二轮 Day 6 的「拦截重复调用的中间件，以及挂上它的 Agent」（`block_repeats`）。只看你自己写的注释重写，重点核对：用 `old["id"] != call["id"]` 排除本次调用；`getattr(m, "tool_calls", None) or []`；拦截时 `ToolMessage` 带 `tool_call_id` 和 `status="error"`。

## 动手练习（30 分钟）

在 `plan_as_state.py` 的基础上：

1. 在脚本里 `book_hotel` 之后加一步：`write_todos` 把「订酒店」改回 `pending`，再调用一次 `book_hotel`。运行，确认酒店被订了两次。
2. 写一个 `@wrap_tool_call` 中间件：如果 `book_hotel` 在历史里已经有成功结果，就不执行，返回「已预订，订单号 H001，请勿重复预订」。
3. 在同一轮 AIMessage 里放两个 `write_todos` 调用，观察中间件返回的错误信息。

想一想：计划状态和工具结果冲突时，UI 应该显示哪一个？

## 读后自测

1. Claude Agent SDK 在新模型上默认不再提供 todo 工具，为什么？那你还该不该给 Agent 加 todo 工具？——见「核心概念」第 1 节
2. 计划写在 Agent 状态里，执行中发现某一步不需要了，怎么改？为什么 `write_todos` 不允许并行调用？——见「核心概念」第 2 节
3. 车机 Agent 要帮司机改导航、订酒店，你怎么设计「先出计划、司机确认后再执行」？——见「核心概念」第 3 节

## AI 历史知识点：MuZero（2019，DeepMind）

**是什么**：一种强化学习算法，把树搜索和一个学出来的模型结合起来做规划。它不需要知道游戏规则，只学规划真正用得到的三样东西：局面好不好（价值）、下一步该怎么走（策略）、刚才那步得了多少分（奖励）。

**何时由谁提出**：DeepMind 的 Julian Schrittwieser、Ioannis Antonoglou、Thomas Hubert、Karen Simonyan、Laurent Sifre 等人，2019 年 11 月 19 日发布于 arXiv，2020 年 12 月正式发表于《Nature》。

**为什么重要**：它的前身 AlphaZero 必须知道规则才能往前推演；MuZero 在不知道规则的情况下，在围棋、国际象棋、将棋上达到 AlphaZero 的超人水平，并在 57 款 Atari 游戏上刷新了最好成绩。DeepMind 的博客还提到：在围棋上，把每步思考时间从 0.1 秒加到 50 秒，棋力提升超过 1000 Elo。

**和今天的趋势有什么联系**：MuZero 说明「规划」不一定要写出一份完整计划，而是在内部推演下一步的价值。今天讲的新模型「不写 todo 也能做完多步任务」是同一个方向：规划能力越来越多地内化到模型里，显式计划更多是给人看的。「多想一会儿就更强」，也和今天推理模型的测试时计算思路一致。

**原始来源**：
- [Mastering Atari, Go, Chess and Shogi by Planning with a Learned Model - arXiv:1911.08265（2019-11-19）](https://arxiv.org/abs/1911.08265)
- [MuZero: Mastering Go, chess, shogi and Atari without rules - Google DeepMind（2020-12-23）](https://deepmind.google/blog/muzero-mastering-go-chess-shogi-and-atari-without-rules/)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `TodoListMiddleware` | `from langchain.agents.middleware import TodoListMiddleware`；可选参数 `system_prompt`、`tool_description`；工具名 `write_todos`；状态字段 `todos`（`OmitFromInput`，不能从输入传入） |
| Todo 结构 | `content` + `status`，状态取值 `pending` / `in_progress` / `completed`；内置提示允许无关任务同时为 `in_progress` |
| `write_todos` 行为 | 返回 `Command(update={"todos": ..., "messages": [ToolMessage]})`；整张替换；`after_model` 检测到一轮内多次调用就全部回 `status="error"` 的 ToolMessage |
| 内置提示要点（本机源码） | 简单的少步骤请求不要用；完成一步立刻标记完成；最终答案写在最后一次 `write_todos` 之后的消息里 |
| Deep Agents | `create_deep_agent()` 默认不装 `TodoListMiddleware`，需通过 `middleware=` 传入（GitHub issue #7437） |
| Claude Agent SDK 任务工具 | `TodoWrite`、`TaskCreate`、`TaskGet`、`TaskUpdate`、`TaskList`；默认只在 Opus 4.7、Sonnet 4.6、Haiku 4.5 及更早模型提供（Claude Code v2.1.268 起）；`CLAUDE_CODE_ENABLE_TASKS=0` 时改用 `TodoWrite` |
| 开启任务工具 | `allowed_tools` 中写任一任务工具名；或 `tools` 选项列出；或 `env={"CLAUDE_CODE_ENABLE_TODO_TOOLS": "1"}` |
| 任务生命周期 | `pending` → `in_progress` → `completed`；`TaskUpdate` 设 `status: "deleted"` 删除；`TaskCreate` 的 ID 在 `tool_use_result` 里，不在输入里 |
| 权限模式 | `default`、`dontAsk`、`acceptEdits`、`bypassPermissions`、`plan`、`auto`；Python 用 `permission_mode=`，会话中用 `set_permission_mode()` 切换 |
| plan 模式细节 | 文件修改不会被自动批准，即使 allow 规则匹配或 `PreToolUse` hook 返回 allow；可用 `AskUserQuestion` 澄清需求；子 Agent 继承 plan 模式 |
| 权限判定顺序 | hooks → deny 规则 → ask 规则 → 权限模式 → allow 规则 → `canUseTool` 回调 |

### 来源说明

- 演示代码用脚本化假模型（运行环境没有 API Key），只验证 `TodoListMiddleware` 的状态读写和整张替换行为，不代表真实模型何时会改计划。
- `TodoListMiddleware` 的状态字段、并行检查和内置提示来自本机安装的 langchain 1.4.8 源码；官方文档页没有写明状态字段名。
- MuZero 来自原始论文和 DeepMind 博客（非 LangChain/Claude 来源）。
- 今日 AI 动态：OpenAI 官网本次无法打开，未能核实，没有收录。

### 延伸学习

- Claude Academy《Claude Code 101》的 The explore → plan → code → commit workflow 一课：为什么先在 Plan Mode 里探索和规划，再写代码。
- Claude Academy《The AI-native SDLC playbook》的 Claude Code plan mode as the default starting point 一课：把审过的计划提交成 `plan.md`，之后用它核对 diff。

### 参考资料

- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [ARCHITECTURE.md describes todo management as a default capability, but it is opt-in - langchain-ai/deepagents #7437](https://github.com/langchain-ai/deepagents/issues/7437)
- [Track todos - Claude Code Docs（Agent SDK）](https://code.claude.com/docs/en/agent-sdk/todo-tracking)
- [Configure permissions - Claude Code Docs（Agent SDK）](https://code.claude.com/docs/en/agent-sdk/permissions)
- [The explore → plan → code → commit workflow - Claude Code 101 - Claude Academy](https://academy.claude.com/courses/claude-code-101/the-explore-plan-code-commit-workflow)
- [Claude Code plan mode as the default starting point - The AI-native SDLC playbook - Claude Academy](https://academy.claude.com/courses/ai-native-sdlc-playbook/plan-mode)
