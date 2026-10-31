# Day 3 · 什么是 Agent（第二轮：停止条件、循环边界与长任务 harness）（2026-10-31）

## 今日 AI 动态（2026-10-31）

### 趋势观察

过去 24 小时官方发布不多，两条都和「旧东西退场、新形态试水」有关：Anthropic 把 Opus 4.8 的 fast mode 列入下线计划，引导用户迁到 Opus 5.5；Google 在轨道卫星上的 TPU 跑通了第一次 Gemma 推理。推测：模型更新换代越来越快，「换模型时要做哪些迁移」会成为线上 Agent 的日常运维工作，也就是昨天讲的提示词迁移。

### 今日要点

- Claude API 宣布弃用 Opus 4.8 的 fast mode，2026-11-30 移除，需要的话迁到 Opus 5.5。
- Google Project Suncatcher 原型卫星上的 TPU 首次在轨完成推理，跑的是开源模型 Gemma 4。

### 分公司动态

#### 1. Anthropic / Claude

**[Claude Platform 发布说明：弃用 Opus 4.8 的 fast mode](https://platform.claude.com/docs/en/release-notes/overview)**（2026-10-30）

fast mode 是用更高价格换更快输出速度的模式。官方宣布 `claude-opus-4-8` 的 fast mode 已弃用，2026-11-30 正式移除，之后用 `speed: "fast"` 请求 Opus 4.8 会直接报错。想继续用 fast mode，要按迁移指南换到 Claude Opus 5.5。

**为什么重要**：线上 Agent 如果靠 fast mode 压低语音交互延迟，一个月内必须完成模型迁移和回归评测，否则请求会直接失败。

#### 2. Google

**[We've received our first response from the Project Suncatcher prototype satellite](https://blog.google/innovation-and-ai/models-and-research/google-research/project-suncatcher-satellite-response/)**（2026-10-30）

Project Suncatcher 是 Google 研究「在太空用太阳能做 AI 计算」的长期项目，原型卫星 10 月 1 日发射。经过几周调试，团队在地面站过境时给星上运行的 Gemma 4 发了一个提示词（写一首关于太空的俳句），在卫星经过挪威斯瓦尔巴时收到了回复。下一步是对在轨 TPU 做热应力和辐射测试，目标是 2027 年发射两颗互联卫星。

**为什么重要**：这是数据中心级 TPU 第一次在轨完成推理，说明「算力放在哪里」的边界还在往外扩。

### 动态来源

- [Claude Platform release notes](https://platform.claude.com/docs/en/release-notes/overview)
- [Project Suncatcher prototype satellite response - Google Blog](https://blog.google/innovation-and-ai/models-and-research/google-research/project-suncatcher-satellite-response/)

---

> 第二轮说明：第一轮 Day 3 讲过 Agent 与 Workflow 的区别、Agent loop（`while stop_reason == "tool_use"`）、何时不该用 Agent 和三条设计原则。今天不再重复，改讲面试官追问最多的「循环的边界」：除了 `tool_use` 和 `end_turn`，还有哪些停止原因要处理；「完成」和「被迫停止」怎么区分；任务长到一个上下文窗口装不下时，harness 要怎么设计。

## 读前三问

1. Agent 循环里，除了 `tool_use` 和 `end_turn`，还有哪些 `stop_reason`？`max_tokens` 截断在工具调用中间时该怎么办？
2. 「步数上限」到了，Agent 应该返回什么？LangChain 里的两种调用上限有什么区别？
3. 一个要跑好几个小时、跨多个上下文窗口的 Agent，怎么保证下一个会话能接着干，而不是从头猜或者提前宣布完成？

## 从一个场景开始

你按第一轮 Day 3 的写法给车机做了个长途规划 Agent：`while stop_reason == "tool_use"` 一直循环，等到 `end_turn` 就把最后一段文字念给司机。上线一周，日志里出现三种怪事：

- 有一次助手什么都没说就结束了。查日志发现 `stop_reason` 是 `max_tokens`，模型写到一半的工具调用被截断，循环条件不满足就直接退出，最后一段文字是空的。
- 接入联网搜索（服务端工具）后，偶尔会出现 `pause_turn`，你的代码不认识这个值，把它当成了结束。
- 步数上限到了，代码返回了最后一轮的半截文字，司机听到的是「我再查一下充电站……」，然后就没有下文了。

循环本身只有十几行，真正的工作量在循环的边界上。

## 一句话总结

Agent 是模型加上一层 harness。循环本身很简单，难在边界：每种停止原因都要有对应处理，「完成」和「被迫停止」要分开报告，任务超出一个上下文窗口时，要靠外部的进度文件和测试清单交接。

## 先跑起来

用 Anthropic SDK 自带的 `Message` 类型构造四条脚本化的假响应，依次是：工具调用被截断、完整的工具调用、`pause_turn`、`end_turn`。然后用同一个循环分别在步数上限为 6 和 2 时各跑一次。不需要 API Key。

```python
# harness_demo.py —— 一个会处理各种 stop_reason 的 Agent 循环（用脚本化的假响应，无需 API Key）
from anthropic.types import Message

def msg(stop, *blocks):  # 用 SDK 自带的 Message 类型构造一条假响应
    return Message.model_validate({"id": "m", "type": "message", "role": "assistant", "model": "claude-opus-5-5",
        "stop_reason": stop, "content": list(blocks), "usage": {"input_tokens": 1, "output_tokens": 1}})
TOOL = {"type": "tool_use", "id": "t1", "name": "get_battery", "input": {}}
SCRIPT = [msg("max_tokens", {"type": "text", "text": "先查电量"}, TOOL),          # 工具调用写到一半被截断
          msg("tool_use", TOOL),                                                    # 重试后完整调用
          msg("pause_turn", {"type": "text", "text": "正在联网查充电站…"}),          # 服务端工具循环暂停
          msg("end_turn", {"type": "text", "text": "电量 38%，途经嘉兴服务区充电即可到杭州。"})]

class FakeClient:
    def __init__(self): self.script, self.calls = list(SCRIPT), []
    def create(self, **kw):
        self.calls.append(kw["max_tokens"]); return self.script.pop(0)

def run(client, task, max_turns):
    messages, max_tokens = [{"role": "user", "content": task}], 1024
    for turn in range(1, max_turns + 1):
        r = client.create(max_tokens=max_tokens, messages=messages)
        print(f"第{turn}轮 stop_reason={r.stop_reason}")
        if r.stop_reason == "end_turn":
            return "completed", r.content[-1].text
        if r.stop_reason == "max_tokens" and r.content[-1].type == "tool_use":
            max_tokens *= 4; continue                       # 半截的调用不能执行，也不追加，加大上限重来
        messages.append({"role": "assistant", "content": r.content})
        if r.stop_reason == "pause_turn":
            continue                                        # 原样追加 assistant 内容后重发即可
        if r.stop_reason == "tool_use":
            results = [{"type": "tool_result", "tool_use_id": b.id, "content": "38%"}
                       for b in r.content if b.type == "tool_use"]
            messages.append({"role": "user", "content": results})   # 只放 tool_result，别追加文字
            continue
        return "stopped:" + r.stop_reason, None             # refusal 等：明确报告，不假装成功
    return "stopped:max_turns", None                        # 被迫停止 ≠ 完成

for limit in (6, 2):
    c = FakeClient(); status, answer = run(c, "电量够不够开到杭州？", limit)
    print(f"max_turns={limit} → {status} | {answer} | 每轮 max_tokens={c.calls}\n")
```

**运行**：`pip install -U anthropic && python harness_demo.py`

**实测输出**（2026-10-31，Python 3.13，anthropic 2.3.0）：

```
第1轮 stop_reason=max_tokens
第2轮 stop_reason=tool_use
第3轮 stop_reason=pause_turn
第4轮 stop_reason=end_turn
max_turns=6 → completed | 电量 38%，途经嘉兴服务区充电即可到杭州。 | 每轮 max_tokens=[1024, 4096, 4096, 4096]

第1轮 stop_reason=max_tokens
第2轮 stop_reason=tool_use
max_turns=2 → stopped:max_turns | None | 每轮 max_tokens=[1024, 4096]
```

**该看什么**：
- 第 1 轮被截断后，`max_tokens` 从 1024 变成 4096，而且那条半截的响应没有追加进历史。第一轮的写法遇到这种情况会直接退出循环。
- 同一个脚本，步数上限为 2 时返回 `stopped:max_turns`，答案是 `None`，而不是把最后半句话当作答案返回。调用方据此决定是转人工还是提示「没查完」。
- 这里的响应是假的，只用来演示控制流；真实环境中 `pause_turn` 只会在用了服务端工具（如联网搜索）时出现。

## 核心概念

### 1. Agent = 模型 + harness

- **是什么**：LangChain 文档把模型之外的一切（提示词、工具、中间件）叫作 harness。Anthropic 的 Tool Runner、LangChain 的 `create_agent` 都是现成的 harness，替你写好了循环。
- **没有它会怎样**：自己写循环时，最容易漏的是边界：截断、暂停、拒答、上限、错误。场景里三个问题全出在这里。
- **对应你熟悉的**：`Looper.loop()` 本身很短，真正让 App 稳定的是 ANR 检测、异常处理和生命周期管理。

选现成的还是自己写？Tool Runner 会自动执行工具、回传结果，提供 `max_iterations` 上限，工具抛出异常时默认会把错误作为 `is_error: true` 的结果交给模型。需要精细控制（审批、自定义重试、特殊日志）时，可以在循环里接管历史，或者干脆自己写，但上面这些边界要自己补全。

### 2. stop_reason 不只两种

- **是什么**：Claude 的每个成功响应都带 `stop_reason`。除了 `tool_use` 和 `end_turn`，Agent 循环最常遇到的还有三种：`max_tokens`（输出上限到了）、`pause_turn`（服务端工具循环到了迭代上限，默认 10 次）、`refusal`（模型拒答）。
- **没有它会怎样**：循环只认识两种值，其他情况就会被当成「结束」，于是出现场景里的空回答。
- **对应你熟悉的**：`when` 表达式不写 `else` 分支，新增一个枚举值就会走错路。

三种情况的处理方式不同：

| 情况 | 正确做法 |
|---|---|
| `max_tokens` 且最后一块是 `tool_use` | 调用不完整，不能执行；加大 `max_tokens` 重试 |
| `pause_turn` | 把 assistant 内容原样追加，再发一次请求，模型会接着做 |
| `refusal` | 是 HTTP 200，不是异常；读 `stop_details`，按需换备用模型重试 |

还有一个容易踩的坑：回传工具结果时，在 `tool_result` 后面再加一段文字，模型会以为轮到用户说话了，返回一个几乎为空的 `end_turn`。

### 3. 完成 ≠ 被迫停止

- **是什么**：步数、调用次数、成本上限是必须有的护栏，但触发上限之后返回的必须是一个明确的「未完成」状态，不能把半截输出当作答案。
- **没有它会怎样**：司机听到「我再查一下……」就没下文了；上游系统还会把这次记成「成功」，评测数据也被污染。
- **对应你熟悉的**：协程被取消要抛 `CancellationException`，而不是悄悄返回一个默认值。

LangChain 把上限拆成两种中间件：`ModelCallLimitMiddleware` 限制模型调用次数，`ToolCallLimitMiddleware` 限制工具调用次数（可以只限某一个工具，比如搜索）。两者的默认行为不同：前者默认 `"end"`，到达上限就平稳结束；后者默认 `"continue"`，只拦下超额的工具调用并回一条错误消息，模型可以换别的办法继续。

### 4. 跨上下文窗口的长任务：靠外部状态交接

- **是什么**：任务长到一个上下文窗口装不下时，每个新会话一开始都不记得之前做过什么。Anthropic 的做法是：第一个会话用专门的提示词（initializer）准备环境，写出启动脚本、进度文件和一份功能清单；之后每个会话只做一个功能，做完提交 git、更新进度。
- **没有它会怎样**：官方观察到三种失败：一次做太多，做到一半上下文用完，留下没有记录的半成品；后来的会话看到已有进展就宣布完成；没有端到端测试就把功能标成已完成。
- **对应你熟悉的**：进程随时可能被系统杀掉，所以要用 `onSaveInstanceState` 把状态存到外面，重建时再恢复。

功能清单用 JSON，每项带 `passes` 字段，初始全部为未通过，后续会话只能改这个字段，不能删改测试。官方选 JSON 而不是 Markdown，是因为模型更不容易乱改 JSON。每个新会话开头固定几步：读 git log 和进度文件，选优先级最高的未完成项，先跑一遍基本的端到端测试，确认上一个会话没有留下问题。

## 高频面试题

**Q1：手写 Agent 循环，除了 `tool_use` 和 `end_turn` 还要处理什么？**
- `max_tokens`：最后一块是半截的 `tool_use` 时，不执行、不追加，加大上限重试。
- `pause_turn`：服务端工具暂停，把 assistant 内容原样追加再发一次。
- `refusal`：HTTP 200，读 `stop_details`，记录下来，必要时换备用模型。
- 未知值统一走「明确停止并上报」分支，不当作完成。

**Q2（追问）：步数上限到了，你的 Agent 返回什么？上游怎么用这个结果？**
- 返回结构化状态（如 `stopped:max_turns`），和 `completed` 分开，答案为空或标注不完整。
- 上游据此决定：转人工、提示用户「没查完」、或者带着进度续跑。
- 监控里单独统计被迫停止率，它往往说明工具返回不清晰或者任务描述模糊。
- LangChain 里用调用上限中间件，选 `"end"` 还是 `"error"` 取决于上游能不能处理异常。

**Q3（场景题）：要做一个能连续跑 3 小时、给车机 App 补全测试用例的 Agent，你怎么设计 harness？**
- 第一个会话做初始化：生成待补功能清单（JSON，带 `passes`），写启动脚本，建进度文件，首次提交。
- 后续每个会话只做一项，做完跑测试、提交 git、更新进度，保证随时可以中断。
- 每个会话开头先读进度和 git log，跑一遍冒烟测试，再选下一项。
- 「完成」由测试通过判定，不由模型自述；只允许改 `passes`，禁止删测试。

**Q4（追问）：用 Tool Runner / `create_agent` 这种现成循环，还是自己写？**
- 默认用现成的：循环、工具执行、错误回传、上限都已经写好，代码少、不易漏边界。
- 需要逐步审批、自定义重试、按工具定制日志时，在循环里接管历史，或者自己写。
- 自己写时，Q1 的边界和上限要全部补齐，并用脚本化的假响应做单元测试（如今天的演示）。

## 常见误区

- **「`stop_reason` 不是 `tool_use` 就说明结束了」**：`max_tokens`、`pause_turn`、`refusal` 都不是正常完成，统统当作结束，就会出现空回答或半截回答。
- **「长任务靠压缩上下文就够了」**：官方明确说压缩有帮助但不够。跨会话的长任务还需要进度文件、功能清单和 git 这类外部状态，让新会话能快速接手。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：处理各种停止原因的 Agent 循环**（取自 `harness_demo.py` 的 `run` 函数）

```python
# 定义循环函数：传入客户端、任务和最大轮数
def run(client, task, max_turns):
    # 历史里先放用户任务，输出上限从 1024 起步
    messages, max_tokens = [{"role": "user", "content": task}], 1024
    # 最多跑 max_turns 轮，轮次从 1 开始数
    for turn in range(1, max_turns + 1):
        # 带着当前上限和完整历史请求一次模型
        r = client.create(max_tokens=max_tokens, messages=messages)
        # 打印这一轮为什么停下
        print(f"第{turn}轮 stop_reason={r.stop_reason}")
        # 正常结束：返回完成状态和最后一段文字
        if r.stop_reason == "end_turn":
            return "completed", r.content[-1].text
        # 工具调用被截断：上限翻四倍，不追加历史，直接重试
        if r.stop_reason == "max_tokens" and r.content[-1].type == "tool_use":
            max_tokens *= 4; continue
        # 其余情况先把模型这一轮的内容追加进历史
        messages.append({"role": "assistant", "content": r.content})
        # 服务端工具暂停：追加后直接再请求一次
        if r.stop_reason == "pause_turn":
            continue
        # 模型要调工具：为每个调用生成一条结果
        if r.stop_reason == "tool_use":
            results = [{"type": "tool_result", "tool_use_id": b.id, "content": "38%"}
                       for b in r.content if b.type == "tool_use"]
            # 结果作为一条用户消息追加，里面只放工具结果
            messages.append({"role": "user", "content": results})
            continue
        # 其他停止原因：明确报告停止，不给答案
        return "stopped:" + r.stop_reason, None
    # 轮数用完：报告被迫停止
    return "stopped:max_turns", None
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 截断分支要放在 `messages.append` 之前，否则半截的 `tool_use` 进了历史，下次请求会因为没有对应的 `tool_result` 而报错。
- `tool_use_id` 取的是 `b.id`，不是 `b.name`；列表推导式里要过滤 `b.type == "tool_use"`，因为同一条响应里可能还有文字块。
- 最后一行 `return` 在 `for` 循环外面，缩进和 `for` 对齐；写进循环里就只会跑一轮。

**昨日复习（只做第③步）**：第二轮 Day 2 的「用中间件拼系统提示词并记录结果」（`build` 函数）。只看你自己写的注释重写，重点核对：运行时信息在 `req.runtime.context` 里；`middleware=[prompt, spy]` 的顺序；`spy` 最后要 `return handler(req)`。

## 动手练习（30 分钟）

在 `harness_demo.py` 的基础上做两件事：

1. 往 `SCRIPT` 里插一条 `msg("refusal", {"type": "text", "text": ""})`，确认 `run` 返回 `stopped:refusal`。再加一个参数 `fallback_client`：遇到 `refusal` 时，用同一份 `messages` 改请求备用客户端一次，备用的也拒答才返回停止。
2. 加「重复调用检测」：记录每次 `(工具名, 参数)`，同样的调用连续出现 3 次，就不再执行，而是回一条 `is_error: true` 的 `tool_result`，内容写「已连续 3 次相同调用，请换一种方法或说明无法完成」。在脚本里放 3 条一样的 `tool_use` 验证一下。

想一想：第 2 题为什么回一条错误结果，而不是直接 `return` 停止？这和 `ToolCallLimitMiddleware` 默认的 `"continue"` 是什么关系？

## 读后自测

1. Agent 循环里，除了 `tool_use` 和 `end_turn`，还有哪些 `stop_reason`？`max_tokens` 截断在工具调用中间时该怎么办？——见「核心概念」第 2 节
2. 「步数上限」到了，Agent 应该返回什么？LangChain 里的两种调用上限有什么区别？——见「核心概念」第 3 节
3. 一个要跑好几个小时、跨多个上下文窗口的 Agent，怎么保证下一个会话能接着干，而不是从头猜或者提前宣布完成？——见「核心概念」第 4 节

## AI 历史知识点：ReAct（2022，Google Research Brain 团队与普林斯顿大学）

**是什么**：ReAct 让语言模型交替输出「推理」和「行动」：先写一段想法（Thought），再发出一个动作（Action，比如调用维基百科搜索），拿到观察结果（Observation）后再想下一步。推理负责制定、跟踪和调整计划，行动负责从外部获取信息。

**何时由谁提出**：Shunyu Yao、Jeffrey Zhao、Dian Yu、Nan Du、Izhak Shafran、Karthik Narasimhan、Yuan Cao 于 2022 年 10 月 6 日提交到 arXiv，后发表于 ICLR 2023。Google Research 博客于 2022 年 11 月 8 日介绍了这项工作，作者来自 Google Research Brain 团队，合作方是普林斯顿 NLP 组。

**为什么重要**：在此之前，思维链只在模型内部推理，无法接触外部世界，容易产生幻觉并层层放大；而只做行动的方法又缺少对目标的抽象推理。ReAct 把两者合在一个循环里。在 HotpotQA、FEVER 上，它借助简单的维基百科 API 减少了思维链的幻觉；在 ALFWorld 和 WebShop 上，只用一两个示例，成功率就比模仿学习和强化学习方法分别高出 34 和 10 个百分点。

**和今天的趋势有什么联系**：今天的 Agent 循环就是 ReAct 的工程化版本：Thought 变成了模型的思考内容，Action 变成了 `tool_use`，Observation 变成了 `tool_result`。ReAct 论文里的循环只在答案出来时停止；放到生产环境，还要补上今天正文讲的截断、暂停、上限和跨会话交接这些边界。

**原始来源**：
- [ReAct: Synergizing Reasoning and Acting in Language Models（arXiv:2210.03629）](https://arxiv.org/abs/2210.03629)
- [Google Research 博客：ReAct: Synergizing Reasoning and Acting in Language Models](https://research.google/blog/react-synergizing-reasoning-and-acting-in-language-models/)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `stop_reason` 全集 | `end_turn`、`max_tokens`、`stop_sequence`、`tool_use`、`pause_turn`、`refusal`、`model_context_window_exceeded` |
| `stop_details` | 只有 `refusal` 时非 `null`，含策略类别；拒答是 HTTP 200，不走异常 |
| `pause_turn` | 服务端工具循环达到迭代上限（默认 10）；原样追加 assistant 内容再发；响应里有 `container` 时要把它的 `id` 带上，否则之前的文件会丢；客户端工具等待执行时不会出现 `pause_turn` |
| 服务端工具未完成 | `tool_use` 响应里有 `server_tool_use` / `mcp_tool_use` 块但没有对应结果块；回传时用户消息只放 `tool_result`，`tools` 数组保持不变；`tool_result` 后加文字会 400 |
| `model_context_window_exceeded` | 响应写满上下文窗口，视为截断；Python 里目前只在 beta 命名空间有类型（`client.beta.messages.create`）；Sonnet 4.5 及以后不需要 beta 头，更早的模型需要 `model-context-window-exceeded-2025-08-26` |
| 空 `end_turn` | 原因：`tool_result` 后追加了文字，或把已完成的响应原样重发；修复：只回 `tool_result`；仍为空时追加一条用户消息「Please continue」 |
| 拒答备用模型 | Fable 5.1、Fable 5、Opus 5.5、Opus 5、Sonnet 5.5、Haiku 5.5 支持；Haiku 5.5 没有服务端备用，需在客户端实现 |
| 流式 | `message_start` 里 `stop_reason` 为 `null`，在 `message_delta` 事件中给出 |
| Tool Runner（beta） | `@beta_tool`（异步用 `@beta_async_tool`）；`client.beta.messages.tool_runner(...)`；`for message in runner` 逐轮迭代或 `runner.until_done()`；`max_iterations`；`runner.generate_tool_call_response()`、`runner.append_messages()`（调用后由你负责历史）、`runner.set_messages_params()`；工具异常默认回传为 `is_error: true`；`ANTHROPIC_LOG=info/debug` 打开日志；压缩用 `compact_before_next_turn()` 或服务端 `context_management`，二选一 |
| `ModelCallLimitMiddleware` | `thread_limit`（跨多次调用，需 checkpointer）、`run_limit`（单次调用）；`exit_behavior`：`"end"`（默认，平稳结束）/ `"error"`（抛异常） |
| `ToolCallLimitMiddleware` | `tool_name`（可选，只限某个工具）、`thread_limit`、`run_limit`（至少设一个）；`exit_behavior`：`"continue"`（默认，拦截超额调用并回错误消息）/ `"error"`（抛 `ToolCallLimitExceededError`）/ `"end"`（立即结束，仅限单个工具时可用） |
| 长任务 harness | initializer 会话产出 `init.sh`、`claude-progress.txt`、功能清单（JSON，字段含类别、描述、测试步骤、`passes`）、首次 git 提交；后续会话每次只做一项；测试用浏览器自动化按真实用户方式进行 |

### 来源说明

- 代码用 SDK 的 `Message` 类型构造脚本化响应，没有调用真实模型（本次运行环境没有 API Key），只验证控制流。
- 历史知识点来自 arXiv 论文和 Google Research 博客（非 LangChain/Claude 来源）。

### 延伸学习

- Claude Academy《Building with the Claude API》的「Workflows vs agents」一课（链接见第一轮 Day 3，本次未能重新打开）。
- Anthropic 工程博客《Effective harnesses for long-running agents》（2025-11-26）。

### 参考资料

- [Stop reasons and fallback - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
- [Tool runner (SDK) - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)
- [Effective harnesses for long-running agents - Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
