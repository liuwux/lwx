# Day 6 · ReAct（第二轮：思考块回传、重复动作与观察质量）（2026-11-03）

## 今日 AI 动态（2026-11-03）

### 趋势观察

本期唯一的新发布来自 Claude：Messages API 推出 message threads，长会话的 Agent 每轮只发新增的消息。它不减少 token，只减少每轮重新上传的数据量，目标是首 token 延迟。推测：Agent 循环跑到几十万 token 之后，瓶颈正在从「模型想得慢」转到「每轮搬运历史慢」，平台开始在传输层替 harness 做优化。

### 今日要点

- Claude Messages API 的 message threads 进入公开测试：首轮发完整对话，之后每轮只发新消息和上一个响应的 ID，500k～1M token 的请求首 token 延迟中位数下降约 35%。

### 分公司动态

#### 1. Anthropic / Claude

**[Make long-running agents faster with message threads](https://claude.com/resources/articles/message-threads)**（2026-11-02）

请求里加一个 `thread` 参数：第一次请求带完整对话并开启线程，之后每次只带新消息和上一个响应的 ID。线程在两轮之间加密保存在内存里、不落盘，最后一次请求后 5 分钟过期，过期后客户端重新发送完整对话即可。Claude 每轮仍然读取完整对话，所以 token 用量和费用不变，变小的只是请求体。官方称大多数 Claude Code 用户已默认开启，500k～1M token 的请求首 token 延迟中位数下降约 35%；Claude Platform、AWS、Microsoft Foundry 和 Google Cloud 渠道均可用。

**为什么重要**：今天讲的 ReAct 循环每轮都要把全部历史和工具结果重新发一遍，这个功能直接降低了长循环、带截图的 computer use Agent 每一轮的等待时间。

### 动态来源

- [Make long-running agents faster with message threads - Claude（2026-11-02）](https://claude.com/resources/articles/message-threads)

---

> 第二轮说明：第一轮 Day 6 讲过 Thought-Action-Observation、文本 ReAct 和原生工具调用的区别、stop 序列和失败模式表。第二轮 Day 3 讲过步数上限中间件。今天不重复，改讲进阶的三件事：Thought 变成思考块之后怎么回传；模型原地打转时怎么拦；观察结果太差时怎么办。

## 读前三问

1. 用 Claude 的自适应思考跑工具循环，回传历史时把 thinking 块删掉，会发生什么？
2. 日志里 Agent 用同样的参数连续搜了 4 次。除了设步数上限，还能怎么办？
3. ReAct 和纯思维链（CoT）的失败原因有什么不同？这对设计车载问答 Agent 有什么启发？

## 从一个场景开始

车机助手接到一句「外滩附近哪里能快充？」。

第一步，它调用 `charging_search(area="外滩")`，默认半径 1km，返回「没有空闲快充」。第二步，它又用一模一样的参数查了一次。第三步、第四步还是一样。直到碰到步数上限，司机只听到「抱歉，我没能完成」。

团队排查时又发现两个问题。有人为了省 token，把历史里的 thinking 块删掉再回传，接口报了 400。还有人把空结果写成「无结果」三个字，模型完全不知道下一步该放大半径还是换地点。

这三件事都发生在 ReAct 循环内部：Thought 的载体、Action 的重复、Observation 的质量。

## 一句话总结

ReAct 现在的形态是「思考块 + 工具调用」循环：思考块要原样回传；原地打转要靠检测重复动作并给出可执行的提示来纠正；观察结果的质量决定 Agent 能不能自己跳出死路，实在不行要切换策略。

## 先跑起来

用中间件拦截「同名同参数」的重复调用：第二次不再真正执行，而是告诉模型「结果不会变，请换参数」。用脚本化的假模型，不需要 API Key。

```python
# react_loop_guard.py —— ReAct 循环里的「重复动作」拦截（脚本化假模型，无需 API Key）
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain.messages import AIMessage, ToolMessage
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel

@tool
def charging_search(area: str, radius_km: int = 1) -> str:
    """搜索某区域半径内空闲的快充站。"""
    if radius_km < 3:
        return f"{area} {radius_km}km 内没有空闲快充。可以把 radius_km 调到 3 以上再试。"
    return f"{area} {radius_km}km 内：人民广场站 空闲2（2.4km）"

@wrap_tool_call
def block_repeats(request, handler):  # 同名同参数的调用，第二次不再真正执行
    call = request.tool_call
    for m in request.state["messages"]:
        for old in getattr(m, "tool_calls", None) or []:
            if old["id"] != call["id"] and (old["name"], old["args"]) == (call["name"], call["args"]):
                return ToolMessage("重复调用：同样的参数已经查过，结果不会变。请换参数或直接回答。",
                                   tool_call_id=call["id"], status="error")
    return handler(request)

class ScriptedModel(GenericFakeChatModel):
    def bind_tools(self, tools, **kw):
        return self
def step(thought, args, i):  # 一条 AIMessage = Thought（文本）+ Action（tool_call）
    return AIMessage(thought, tool_calls=[{"name": "charging_search", "args": args, "id": f"c{i}"}])
script = [step("先查外滩附近 1km。", {"area": "外滩"}, 0),
          step("再查一次外滩。", {"area": "外滩"}, 1),  # 模型卡住：原样重复上一步
          step("被拦了，按提示把半径放大到 3km。", {"area": "外滩", "radius_km": 3}, 2),
          AIMessage("外滩 3km 内人民广场站有 2 个空闲快充，约 2.4km。")]
agent = create_agent(ScriptedModel(messages=iter(script)), tools=[charging_search],
                     middleware=[block_repeats])
out = agent.invoke({"messages": [{"role": "user", "content": "外滩附近哪里能快充？"}]})
for m in out["messages"][1:]:
    if m.type == "ai":
        print(f"Thought: {m.content}\n Action: {m.tool_calls[0]['args']}" if m.tool_calls else f" Answer: {m.content}")
    else:
        print(f"   Obs[{m.status}]: {m.content}")
```

**运行**：`pip install -U langchain langgraph && python react_loop_guard.py`

**实测输出**（2026-11-03，Python 3.13，langchain 1.4.8，langgraph 1.2.19）：

```
Thought: 先查外滩附近 1km。
 Action: {'area': '外滩'}
   Obs[success]: 外滩 1km 内没有空闲快充。可以把 radius_km 调到 3 以上再试。
Thought: 再查一次外滩。
 Action: {'area': '外滩'}
   Obs[error]: 重复调用：同样的参数已经查过，结果不会变。请换参数或直接回答。
Thought: 被拦了，按提示把半径放大到 3km。
 Action: {'area': '外滩', 'radius_km': 3}
   Obs[success]: 外滩 3km 内：人民广场站 空闲2（2.4km）
 Answer: 外滩 3km 内人民广场站有 2 个空闲快充，约 2.4km。
```

**该看什么**：
- 第二次 Action 和第一次完全相同，`Obs[error]` 说明工具没有真正执行，中间件直接回了一条提示。
- 第一次的空结果没有只写「无结果」，而是带了下一步建议（把半径调到 3 以上）。真实模型读到这样的观察才知道往哪改；本例的三步是预先写好的。
- 中间件通过 `request.state["messages"]` 读到了之前所有的工具调用，不需要自己额外存状态。

## 核心概念

### 1. Thought 现在是思考块，必须原样回传

- **是什么**：原生实现里，ReAct 的 Thought 就是 Claude 的 `thinking` 块。用自适应思考（`thinking: {"type": "adaptive"}`）时，交错思考是自动的：每拿到一个工具结果，模型都可以先想一段再行动，不需要 beta 头。
- **没有它会怎样**：如果回传时删掉或改写思考块，接口会报 400。官方文档说，单独按 `type == "thinking"` 过滤，还会顺手删掉 `redacted_thinking` 块，同样会破坏协议。
- **对应你熟悉的**：服务端下发的带签名 token，客户端只能原样带回，不能解析后重新拼。思考块里的 `signature` 字段就是这个作用。

两条容易忽略的规则：第一，一整个工具循环算一个 assistant 轮次，这一轮内不能切换思考配置；切换了不会报错，而是悄悄关掉思考，要看响应里有没有 `thinking` 块才能发现。第二，在 Opus 5.5 等模型上，思考内容默认不显示，`thinking` 字段是空字符串、只有 `signature`，也照样要原样回传。

### 2. think 工具已经不是首选

- **是什么**：think 工具是一个什么都不做的工具，只有一个 `thought` 参数，让模型在长工具链中途「停下来想一想」。2025 年它在 τ-Bench 航空场景上带来了明显提升。
- **现在的变化**：官方博客顶部加了更新说明：扩展思考的能力已经提升，**大多数情况下建议用扩展思考，不再单独加 think 工具**。
- **对应你熟悉的**：系统已经提供了原生能力，就不再自己用反射去实现。

面试时可以这样回答：交错思考已经覆盖了「拿到结果后再想」这件事。think 工具原本适合的场景是规则多、步骤有先后、出错代价高的任务，比如退改签。并行调用或单次调用的任务本来就不需要它。

### 3. 重复动作：最典型的 ReAct 推理错误

- **是什么**：ReAct 论文在 HotpotQA 上做了错误分析，ReAct 失败案例里 47% 是推理错误，其中一种是 ReAct 特有的：模型反复生成之前的 Thought 和 Action，跳不出循环。
- **没有它会怎样**：只设步数上限，等于眼看着它重复到底再强制停下，司机等了半天只得到「没能完成」。
- **对应你熟悉的**：按钮防抖。同样的点击在短时间内只处理一次，第二次直接给反馈。

上限和检测的分工不同：上限是兜底，保证一定会停；检测是纠偏，在第二次重复时就把模型拉回来。检测返回的提示必须能直接照着做，比如「换参数或直接回答」，不能只说「失败」。

### 4. 观察质量决定能不能自己走出死路

- **是什么**：同一份分析里，ReAct 失败案例有 23% 是搜索结果错误：结果为空或没有用。论文说这类结果会打乱模型的推理，让它很难重新组织思路。
- **没有它会怎样**：工具只回「无结果」，模型只能猜下一步，最常见的猜法就是原样再试一次，于是变成第 3 节的循环。
- **对应你熟悉的**：空列表页不只显示一片空白，还要给「清除筛选」「扩大范围」的按钮。

论文还给了一个回退策略：ReAct 在 HotpotQA 上 7 步、FEVER 上 5 步内没给出答案，就改用 CoT-SC（多次采样思维链再投票）；反过来，CoT-SC 投票结果不够集中时，再改用 ReAct。它说明的道理是：ReAct 事实可靠但会卡住，CoT 灵活但容易幻觉，卡住时应该换策略，而不是硬撑到上限。车机上的对应做法是：查不到就明确告诉司机，并给出一个替代方案。

## 高频面试题

**Q1（场景题）：线上 Agent 用同样的参数连续搜了 4 次，最后超时。你怎么修？**
- 先看 trace：观察结果是不是空的、有没有给下一步建议。
- 修工具：空结果要说明原因和改法（放大半径、换关键词）。
- 加检测：同名同参数在第二次就拦下，回一条可执行的提示。
- 保留步数和调用上限作为兜底，触发后返回明确的「未完成」。

**Q2（追问）：用 Claude 自适应思考跑工具循环，为了省 token 把 thinking 块删掉再回传，会怎样？**
- 工具循环内必须原样回传思考块，删掉或改写会报 400。
- 按 `type == "thinking"` 过滤还会误删 `redacted_thinking` 块。
- 想省上下文，就别在循环中途动它；跨轮次的旧思考块交给 API 自动处理。
- 注意：Opus 4.5 及以后等模型会保留旧思考块，它们计入上下文和输入 token。

**Q3：think 工具和扩展思考怎么选？**
- 官方现在的建议：大多数情况用扩展思考。
- 交错思考已经能在每个工具结果之后思考。
- think 工具只在规则多、顺序决策、出错代价高的场景考虑，且要用评测验证。
- 单次调用或并行调用的任务两者都不需要额外加。

**Q4（追问）：ReAct 和 CoT 的失败分布有什么不同？对车载问答有什么启发？**
- CoT 失败案例里 56% 是幻觉，ReAct 是 0%；ReAct 主要败在推理错误（47%）和搜索结果差（23%）。
- 启发一：涉及实时数据（充电桩、路况）必须走工具，不能让模型凭记忆答。
- 启发二：工具结果的质量和提示，比 prompt 里的大道理更能减少卡死。
- 启发三：设好切换策略的点，卡住时换路径或如实告诉司机。

## 常见误区

- **「比较参数字典就能识别重复」**：本例是字面比较，`{"area": "外滩"}` 和 `{"area": "外滩", "radius_km": 1}` 实际上是同一次查询，却会被当成不同调用。要先补齐默认值、统一格式再比较。
- **「上一轮的思考块不占上下文」**：要看模型。较早的模型只保留最后一轮，API 会自动去掉更早的思考块；Opus 4.5 及以后的 Opus、Sonnet 4.6 及以后的 Sonnet 和 5.x 系列会保留，它们计入上下文窗口，按输入 token 计费。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：拦截重复调用的中间件，以及挂上它的 Agent**（取自 `react_loop_guard.py`）

```python
# 声明一个包裹每次工具调用的中间件
@wrap_tool_call
def block_repeats(request, handler):
    # 取出这次要执行的工具调用（字典：name、args、id）
    call = request.tool_call
    # 遍历 Agent 状态里已有的全部消息
    for m in request.state["messages"]:
        # 只有 AI 消息带 tool_calls，其他消息当成空列表
        for old in getattr(m, "tool_calls", None) or []:
            # 排除这次调用本身，再比较工具名和参数是否完全相同
            if old["id"] != call["id"] and (old["name"], old["args"]) == (call["name"], call["args"]):
                # 是重复调用：不执行工具，直接回一条错误提示，告诉模型怎么改
                return ToolMessage("重复调用：同样的参数已经查过，结果不会变。请换参数或直接回答。",
                                   tool_call_id=call["id"], status="error")
    # 不是重复调用：正常执行工具
    return handler(request)

# 创建 Agent：传入模型和工具，挂上去重中间件
agent = create_agent(ScriptedModel(messages=iter(script)), tools=[charging_search],
                     middleware=[block_repeats])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 当前这次调用已经在 `state["messages"]` 里了，不排除 `old["id"] != call["id"]`，每次调用都会被当成重复。
- `getattr(m, "tool_calls", None) or []`：人类消息和工具消息没有 `tool_calls`，漏了 `or []` 会在 `None` 上遍历报错。
- 拦截时返回的 `ToolMessage` 必须带 `tool_call_id=call["id"]`，否则模型的这次调用没有配对的结果。

**昨日复习（只做第③步）**：第二轮 Day 5 的「会做业务校验的预约工具，以及把异常转成错误结果的中间件」（`charging_book` 和 `errors_to_model`）。只看你自己写的注释重写，重点核对：`tool_call_id` 用 `request.tool_call["id"]`；`status="error"` 是关键字参数；`handler(request)` 前面要有 `return`。

## 动手练习（30 分钟）

在 `react_loop_guard.py` 的基础上：

1. 把脚本第二步改成 `{"area": "外滩", "radius_km": 1}`，确认现在拦不住。再修改 `block_repeats`：比较之前用 `charging_search.args_schema` 补齐默认值（提示：`args_schema(**args).model_dump()`），让它重新拦住。
2. 再挂一个 `ToolCallLimitMiddleware(tool_name="charging_search", run_limit=2)`，看第三次搜索时的输出有什么不同。

想一想：被拦下两次之后，应该让模型继续想办法，还是直接结束并告诉司机？

## 读后自测

1. 用 Claude 的自适应思考跑工具循环，回传历史时把 thinking 块删掉，会发生什么？——见「核心概念」第 1 节
2. 日志里 Agent 用同样的参数连续搜了 4 次。除了设步数上限，还能怎么办？——见「核心概念」第 3 节
3. ReAct 和纯思维链（CoT）的失败原因有什么不同？这对设计车载问答 Agent 有什么启发？——见「核心概念」第 4 节

## AI 历史知识点：过程监督 Let's Verify Step by Step（2023，OpenAI）

**是什么**：训练奖励模型时，不只看最终答案对不对（结果监督），而是给推理的每一步打分（过程监督）。论文同时发布了 PRM800K 数据集：针对 1.2 万道题的 7.5 万个解答，共 80 万条人工标注的单步对错标签。

**何时由谁提出**：OpenAI 的 Hunter Lightman、Vineet Kosaraju、Yura Burda、Jan Leike、John Schulman、Ilya Sutskever、Karl Cobbe 等人，2023 年 5 月 31 日发布于 arXiv。

**为什么重要**：在 MATH 测试集的代表性子集上，从 1860 个候选解里挑答案，过程奖励模型挑对了 78.2%，结果奖励模型是 72.4%，多数投票是 69.6%。论文还发现主动学习能明显提高过程监督的标注效率。它说明「每一步都对」比「答案碰巧对」更可靠，也更容易发现推理错在哪一步。

**和今天的趋势有什么联系**：ReAct 的轨迹就是一串可检查的步骤。今天讲的重复动作检测，是在用规则做最简单的「单步检查」；评测 Agent 时看整条轨迹而不只看最终回答，也是同一个思路。后来的推理模型训练也大量借鉴了对中间步骤的监督。

**原始来源**：
- [Let's Verify Step by Step - arXiv:2305.20050（2023-05-31）](https://arxiv.org/abs/2305.20050)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| 交错思考 | 自适应思考下自动开启、无需 beta 头；手动扩展思考（`type: "enabled"`）需要 beta 头，且思考预算计算方式不同；Haiku 4.5 不支持；只对 Messages API 调用的工具生效 |
| 思考块回传 | 工具循环内必须原样回传（含 `signature`、`redacted_thinking`）；跨轮次建议全部回传，由 API 自动过滤；工具循环之外可以省略旧轮次的思考 |
| 旧思考块是否计费 | 保留型：Opus 4.5 及以后 Opus、Sonnet 4.6 及以后 Sonnet、Haiku 5.5、5.x 系列，计入窗口和输入 token；只留最后一轮型：更早的 Opus/Sonnet、Haiku 4.5 及以前 |
| 轮内切换思考配置 | 不报错，静默关闭思考；可能去掉不合法的思考块；同时让提示缓存失效 |
| `tool_choice` 与思考 | 手动扩展思考：强制工具调用报错，用 `auto` 或 `none`；自适应思考：支持强制，但 Opus 5.5、Sonnet 5.5、Fable 5.1、Mythos 5.1 返回 400；被接受时会跳过思考 |
| 思考显示 | `display: "summarized"` 才能看到思考文本；Opus 5.5 等默认 omitted，`thinking` 为空字符串，只有 `signature` |
| think 工具（2025-03-20） | 参数只有 `thought`；τ-Bench 航空 pass^1 约 0.57 对基线 0.37（正文数据，正文与表格略有出入）；零售 0.812 对 0.783；SWE-Bench 平均提升 1.6%；现已建议多数情况用扩展思考 |
| ReAct 错误分析（HotpotQA，各抽 50 条） | ReAct：推理错误 47%、搜索结果错误 23%、幻觉 0%、标签歧义 29%；CoT：幻觉 56%、推理错误 16%、标签歧义 28%；成功案例中 CoT 有 14% 是假阳性（推理或事实有幻觉），ReAct 6% |
| ReAct 回退策略 | ReAct→CoT-SC：HotpotQA 7 步、FEVER 5 步内无答案；CoT-SC→ReAct：n 次采样中多数答案出现少于 n/2 次 |
| LangChain `ToolCallRequest` | 字段：`tool_call`、`tool`、`state`、`runtime`（本机 langchain 1.4.8 实测）；`@wrap_tool_call` 可以不调用 `handler`，直接返回 `ToolMessage` |
| LangChain 相关内置中间件 | `ToolCallLimitMiddleware`、`ModelCallLimitMiddleware`、`ToolRetryMiddleware`、`ToolErrorMiddleware`、`ContextEditingMiddleware`、`SummarizationMiddleware` |
| `jump_to` | 节点式钩子（`@before_model`、`@after_model`）可返回 `jump_to`，目标为 `end`、`tools`、`model`；需用 `can_jump_to` 声明 |

### 来源说明

- ReAct 错误分析和回退策略来自原始论文（非 LangChain/Claude 来源）。
- 演示代码用脚本化假模型（运行环境没有 API Key），只验证中间件拦截行为，不代表真实模型的选择。
- 今日 AI 动态：OpenAI 官网和帮助中心页面本次均无法打开，未能核实，没有收录。

### 延伸学习

- Claude Academy《Building with the Claude API》「Features of Claude」部分的 Extended thinking 一课：讲思考块签名、`redacted_thinking` 以及如何测试应用能否正确处理被加密的思考块（课程示例用的是旧的 `budget_tokens` 写法）。
- Claude Academy《Claude Platform 101》的 What is thinking? 一课：讲自适应思考、effort 档位，示例是一个带天气工具、规划两站自驾路线的 Agent。

### 参考资料

- [Thinking - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/thinking)
- [Thinking in tool and multi-turn workflows - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/thinking-tool-workflows)
- [The "think" tool for complex tool use - Anthropic（2025-03-20）](https://www.anthropic.com/engineering/claude-think-tool)
- [Custom middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [Extended thinking - Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/extended-thinking)
- [What is thinking? - Claude Platform 101 - Claude Academy](https://academy.claude.com/courses/claude-platform-101/what-is-thinking)
- [ReAct: Synergizing Reasoning and Acting in Language Models - arXiv:2210.03629（非 LangChain/Claude 来源）](https://arxiv.org/html/2210.03629v3)
