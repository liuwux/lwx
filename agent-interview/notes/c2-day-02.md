# Day 2 · Prompt 工程（第二轮：模型迁移、语气强度与缓存友好的排布）（2026-10-30）

## 今日 AI 动态（2026-10-30）

### 趋势观察

过去 24 小时的官方发布有两条主线。一条是安全对齐：OpenAI 同一天发了两篇文章，一篇讲新版 Model Spec（模型该怎么做），一篇讲怎么在线上发现模型和 Agent 的越权行为。另一条是 Agent 的输出形式：Google 推的 A2UI 让 Agent 输出可交互的界面，而不只是文字。推测：「Agent 怎么被监控」和「Agent 怎么跟用户界面交互」会越来越常出现在系统设计面试里。

### 今日要点

- OpenAI 发布新版 Model Spec，用「五项核心价值」重新组织模型的行为规范。
- OpenAI 公开了用分层监控发现前沿模型未对齐和 Agent 越权行为的做法。
- Google 发文介绍 A2UI 和 CopilotKit 的集成：Agent 用数据描述界面，由 App 用自己的组件渲染，官方渲染器包括 Jetpack Compose。

### 分公司动态

#### 1. OpenAI

**[An updated Model Spec for more capable and beneficial AI](https://openai.com/index/updated-model-spec)**（2026-10-29）

Model Spec 是 OpenAI 规定模型默认行为的规范文档。新版改为围绕「五项核心价值」来组织，目标是让能力更强的模型保持良好判断、尊重人的自主性、忠于人的意图。正文页面本次没能打开，日期和摘要来自 OpenAI 官方 RSS，五项价值的具体内容未核实。

**为什么重要**：模型行为规范从「规则清单」转向「价值框架」，和 Claude 文档里「解释动机，而不是只下命令」是同一个思路，也就是今天正文要讲的内容。

**[How we detect frontier model misalignment](https://openai.com/index/how-we-detect-frontier-model-misalignment)**（2026-10-29）

文章介绍 OpenAI 用分层监控来发现前沿模型的未对齐行为和 Agent 的越权行为，目标是在问题扩大之前让人介入。本条同样只核实了官方 RSS 中的摘要，没有具体指标。

**为什么重要**：对齐工作正在从训练阶段延伸到上线后的实时监控，这是 Agent 可观测性和护栏的延伸。

#### 2. Google

**[Build Generative UI Apps with A2UI & CopilotKit Agents](https://developers.googleblog.com/build-generative-ui-apps-with-a2ui-copilotkit-agents/)**（2026-10-29）

A2UI 是 Google 的开放生成式 UI 标准：Agent 把界面描述成一串小的 JSON 消息，App 用自己信任的组件把它渲染出来，所以可以边生成边显示。官方渲染器覆盖 React、Lit、Angular、SwiftUI、Jetpack Compose 和 Flutter，基础组件库有 18 个通用组件。CopilotKit 基于 AG-UI 协议接入 A2UI，支持 Google ADK、LangChain 等框架；用户在界面上的操作会回传给 Agent，可以做多步流程。

**为什么重要**：对 Android 开发者来说，「Agent 输出 Compose 界面」有了官方渲染器，这是转岗时能直接用上的结合点。

### 动态来源

- [An updated Model Spec for more capable and beneficial AI](https://openai.com/index/updated-model-spec)
- [How we detect frontier model misalignment](https://openai.com/index/how-we-detect-frontier-model-misalignment)
- [OpenAI News RSS](https://openai.com/news/rss.xml)
- [Build Generative UI Apps with A2UI & CopilotKit Agents](https://developers.googleblog.com/build-generative-ui-apps-with-a2ui-copilotkit-agents/)

---

> 第二轮说明：第一轮 Day 2 讲过清晰直接、解释动机、XML 分区、few-shot、CoT、结构化输出和「一次只改一处」的迭代方法。今天不再重复这些基础，改讲面试官的追问：提示词为什么要跟着模型版本重新调，强硬语气为什么会起反作用，以及怎样排布提示词才不浪费缓存。

## 读前三问

1. 把线上 Agent 从旧模型换到新模型，提示词能原样沿用吗？迁移时具体要做哪几件事？
2. 为什么官方不建议在提示词里写「CRITICAL：你必须……」？安全底线又该怎么写？
3. 每次请求都要带当前时间、车速这类变化的信息，放在提示词的什么位置？为什么？

## 从一个场景开始

你的车机助手跑了三个月，表现稳定。这周公司把模型升级到新版本，提示词一个字没改，问题却接连出现：

- 用户问「胎压多少合适」，助手先说「我来查一下手册」，再回答，最后又补一句「我再确认一下」，TTS 念了 20 秒。
- 系统提示词里有一句「CRITICAL：涉及车辆状态时你必须调用 get_vehicle_status」。升级后，用户只是问「今天天气怎么样」，它也要先查一遍车辆状态。
- 账单上的输入费用比上个月高了很多。你查日志才发现，提示词第一行是「当前时间：08:00:01」，每次请求都不一样。

这三件事都不是模型「变笨了」，而是提示词没有跟着模型和计费方式一起调整。

## 一句话总结

提示词是针对某个模型版本调出来的，换模型要用评测集重新验证。新模型对指令更敏感，规则要用平常语气写并说明原因，不要用强硬语气。不变的内容放前面、会变的内容放后面，才能命中缓存。

## 先跑起来

用 LangChain 的 `@dynamic_prompt` 给车机助手拼系统提示词，对比「动态信息放开头」和「动态信息放末尾」两种排布。模型用 LangChain 自带的假模型，不需要 API Key。

```python
# prompt_layout_demo.py —— 同一份车机助手提示词，两种排布对前缀缓存的影响（用假模型，无需 API Key）
import os
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, wrap_model_call, ModelRequest
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

MANUAL = "车辆手册：" + "胎压标准 2.5bar；保养周期 1 万公里；" * 300   # 静态的大段资料
RULES = "回答会被 TTS 朗读给正在开车的司机，所以不超过 30 个字。"           # 规则 + 动机

@dataclass
class Ctx:
    now: str        # 每次请求都会变的信息
    speed: int

def build(order: str):
    @dynamic_prompt
    def prompt(req: ModelRequest) -> str:
        c = req.runtime.context
        dyn = f"当前时间 {c.now}，车速 {c.speed} km/h。"
        return dyn + MANUAL + RULES if order == "动态在前" else MANUAL + RULES + dyn
    seen = []
    @wrap_model_call
    def spy(req: ModelRequest, handler):                # 记录真正发给模型的 system prompt
        seen.append(req.system_prompt)
        return handler(req)
    model = GenericFakeChatModel(messages=iter([AIMessage("好的")] * 2))
    agent = create_agent(model, tools=[], middleware=[prompt, spy], context_schema=Ctx)
    for now, speed in [("08:00:01", 60), ("08:00:09", 62)]:
        agent.invoke({"messages": [{"role": "user", "content": "胎压多少合适？"}]},
                     context=Ctx(now=now, speed=speed))
    return seen

for order in ["动态在前", "动态在后"]:
    a, b = build(order)
    shared = len(os.path.commonprefix([a, b]))
    print(f"{order}: 总长 {len(a)} 字，两次请求相同前缀 {shared} 字（{shared/len(a):.1%} 可命中缓存）")
```

**运行**：`pip install -U langchain && python prompt_layout_demo.py`

**实测输出**（2026-10-30，Python 3.13，langchain 1.4.7，langchain-core 1.7.3）：

```
动态在前: 总长 6962 字，两次请求相同前缀 12 字（0.2% 可命中缓存）
动态在后: 总长 6962 字，两次请求相同前缀 6949 字（99.8% 可命中缓存）
```

**该看什么**：
- 两种排布内容完全一样，只是顺序不同，可复用的前缀却从 12 个字变成了 6949 个字。前 12 个字相同只是因为「当前时间 08:00:0」这一段碰巧一样。
- 这里按字符统计，只用来演示原理。真实的缓存按 token 前缀匹配，规则相同：从第一个不同的地方开始，后面全部要重新计算。
- `spy` 排在 `prompt` 后面，所以看到的是已经拼好的提示词。中间件按列表顺序一层套一层，调试时这个顺序很重要。

## 核心概念

### 1. 提示词跟着模型版本走

- **是什么**：一份提示词是在某个模型上调出来的。换了模型，同样的话可能被理解得更重、更字面，或者模型本来就会做的事你又让它做了一遍。
- **没有它会怎样**：场景里的「我再确认一下」就是这样来的。旧提示词写了「回答前先检查一遍」，而新模型本来就会主动验证，两者叠加，就多做了一遍。
- **对应你熟悉的**：升级 `targetSdkVersion`。代码不改也能编译，但行为变了，必须重新跑测试。

Claude 官方为每个主要模型都写了单独的提示词指南。以 Opus 5 的迁移建议为例：删掉「最后加一步验证」「再检查一遍答案」这类指令，因为它本身就会验证；回答长度要明确写出来，因为 effort 只控制思考多少，不控制回答多长。

迁移的做法和第一轮讲的一样，以评测为准：用同一套评测集跑新旧两个模型，先删掉为旧模型打的「补丁」，再逐条决定加什么。

### 2. 语气强度：强硬语气会让模型过度执行

- **是什么**：新模型对系统提示词更敏感。以前为了防止模型「偷懒」写的「CRITICAL」「你必须」「拿不准就调用这个工具」，到了新模型上会变成过度执行。
- **没有它会怎样**：问天气也要先查车辆状态（工具被过度调用）；或者为了遵守一条措辞很重的规则，把其他规则都牺牲掉了。
- **对应你熟悉的**：Log 级别全都用 `Log.e`，最后谁也分不清哪条才真正重要。

官方的改法是用平常语气写，并说明原因：把「CRITICAL：你必须使用这个工具……」改成「在……时使用这个工具」。安全底线不是靠加重语气保证的，而是要写清楚原因（比如「误操作车控比识别不出更危险」），同时在代码层面做权限检查和人工确认（见 Day 24）。

和语气相关的还有两件事：
- 「动作还是建议」：问「能不能给些修改建议？」，模型给的是建议；说「把这个函数改快一点」，模型才会动手改。默认该怎么做，要在系统提示词里写清楚。
- 「范围」：新模型更容易做多，要写明「只做被要求的和明显必要的事」。

### 3. 排布与缓存：不变的在前，变化的在后

- **是什么**：Claude 的提示词缓存按 tools → system → messages 的顺序做前缀匹配。哪一层变了，这一层和后面所有层的缓存都会失效。
- **没有它会怎样**：像上面的演示一样，开头放一个时间戳，后面 7000 字就全部要按原价重新计算，而且首字延迟也会变长。
- **对应你熟悉的**：Gradle 的增量编译。改了最底层的模块，依赖它的模块全部要重新编译。

具体做法：工具定义和大段资料放在最前面并保持稳定；每次都会变的信息（时间、车速、用户输入）放到最后，或者放进用户消息。工具列表也不要每次随便换，因为它在最前面，一改就全部失效。

这条规则和第一轮讲的「长文档放前面、问题放最后」（官方测试最多提升约 30% 的质量）方向一致：一次排布同时照顾了效果和成本。

### 4. 会话中途改规则：不要改开头

- **是什么**：会话进行到一半要换规则（比如车速超过 100 后只回答导航和安全问题），有两种改法：改最开头的 `system` 字段；或者在消息末尾追加一条 `role: "system"` 消息（Claude 文档称为 mid-conversation system messages）。
- **没有它会怎样**：改开头会让前面已经缓存的几十轮对话全部失效。追加到末尾的话，前面的内容不变，缓存依然能命中。
- **对应你熟悉的**：往 RecyclerView 末尾追加一条用 `notifyItemInserted`，而不是 `notifyDataSetChanged` 刷新全部。

需要注意：系统消息相当于运营方的指令，权限比用户消息高。检索到的文档、网页、工具返回的结果绝不能放进去，否则就给了提示注入可乘之机（见 Day 23），这些内容要放在 `tool_result` 里。LangChain 的 `@dynamic_prompt` 解决的是另一个问题：每次调用时按运行时信息重新生成系统提示词。它很灵活，但如果把变化的内容拼在开头，就会出现演示里缓存全部失效的情况。

## 高频面试题

**Q1：线上 Agent 要换新模型，提示词怎么迁移？**
- 先跑一遍现有评测集，拿到新旧两个模型的分数和失败样例。
- 读新模型的官方提示词指南，删掉为旧模型打的补丁（重复验证、强硬语气、prefill）。
- 重新扫一遍 effort 等参数，回答长度、主动性、范围这些要明确写出来。
- 一次只改一处，跑回归测试，达标后再灰度上线。

**Q2（追问）：不让写 CRITICAL / MUST，那「绝对不能做」的安全规则怎么保证？**
- 用平常语气写规则，并说明原因，模型能根据原因推广到没写到的情况。
- 提示词只是第一层，不可逆操作要靠代码里的权限检查和人工确认。
- 用对抗样例做评测，验证规则确实生效，而不是靠语气重来「保证」。
- 强硬语气的代价是其他场景下的过度执行，评测里也要覆盖这种情况。

**Q3（场景题）：车机助手的输入成本很高，日志显示几乎没有命中缓存，怎么排查和改？**
- 看 `usage` 里的 `cache_read_input_tokens`，确认是否命中；两个缓存字段都是 0 说明根本没缓存。
- 找出提示词前部每次都会变的内容：时间戳、车速、随机顺序的工具列表等。
- 按「工具、静态资料、规则在前，动态信息在后」重新排布，并检查是否达到最小可缓存长度。
- 改完对比命中率、成本和首字延迟，同时跑评测，确认效果没有下降。

**Q4（追问）：会话中途要加一条新规则，怎么做才不破坏缓存？有什么风险？**
- 不改开头的 `system`，在消息末尾追加一条系统消息，前面的缓存仍然有效。
- 注意位置限制：它必须紧跟在用户消息之后，不能作为第一条消息。
- 风险：系统消息权限高，外部内容（检索结果、网页、工具输出）绝不能放进去。
- 不支持这个功能的模型，就把动态信息放进用户消息。

## 常见误区

- **「多写一句『请仔细检查』总没坏处」**：新模型本来就会主动验证，叠加之后变成重复检查，又慢又贵。换模型时这类指令应该先删掉，再用评测决定要不要加回来。
- **「用 prefill 预填 `{` 逼它输出 JSON」**：Claude 4.6 及以后的模型已经不支持预填最后一轮 assistant 消息，会直接返回 400。格式要用结构化输出或指令来控制。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：用中间件拼系统提示词并记录结果**（取自 `prompt_layout_demo.py`）

```python
# 定义构建函数：按指定排布方式创建 Agent 并跑两次请求
def build(order: str):
    # 声明一个动态系统提示词中间件
    @dynamic_prompt
    def prompt(req: ModelRequest) -> str:
        # 取出这次调用的运行时上下文
        c = req.runtime.context
        # 拼出每次都会变的那段信息
        dyn = f"当前时间 {c.now}，车速 {c.speed} km/h。"
        # 按排布方式决定动态信息放在开头还是末尾
        return dyn + MANUAL + RULES if order == "动态在前" else MANUAL + RULES + dyn
    # 准备一个列表，存每次真正发出去的提示词
    seen = []
    # 声明一个包裹模型调用的中间件，用来偷看请求
    @wrap_model_call
    def spy(req: ModelRequest, handler):
        # 记下已经拼好的系统提示词
        seen.append(req.system_prompt)
        # 把请求原样交给下一层继续执行
        return handler(req)
    # 用假模型代替真模型，固定回答两次
    model = GenericFakeChatModel(messages=iter([AIMessage("好的")] * 2))
    # 创建 Agent：先拼提示词再偷看，并声明上下文类型
    agent = create_agent(model, tools=[], middleware=[prompt, spy], context_schema=Ctx)
    # 用两组不同的时间和车速各请求一次
    for now, speed in [("08:00:01", 60), ("08:00:09", 62)]:
        # 发出请求，运行时信息通过 context 传入
        agent.invoke({"messages": [{"role": "user", "content": "胎压多少合适？"}]},
                     context=Ctx(now=now, speed=speed))
    # 返回两次请求的提示词
    return seen
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 运行时信息在 `req.runtime.context` 里，不是 `req.context`；调用时用 `invoke(..., context=Ctx(...))` 传进去，不是放在 `config` 里。
- `middleware=[prompt, spy]` 的顺序不能反：`spy` 在后面才能看到拼好的提示词。
- `spy` 最后必须 `return handler(req)`，漏掉的话模型根本不会被调用。

**昨日复习（只做第③步）**：第二轮 Day 1 的「带 KV cache 的生成循环」（`generate` 函数）。只看你自己写的注释重写，重点核对：`np.vstack` 的参数是列表；注意力用的是 `x @ Wq`；有缓存的分支每步只加 1。

## 动手练习（30 分钟）

在 `prompt_layout_demo.py` 的基础上做两件事：

1. 加第三种排布「动态进用户消息」：系统提示词只放 `MANUAL + RULES`，把时间和车速拼到用户消息前面。`spy` 里改为同时记录 `req.system_prompt` 和最后一条消息的内容，打印三种排布的前缀复用比例。
2. 写一个 `lint_prompt(text)` 函数，找出提示词里的「CRITICAL」「必须」「绝对」「务必」，列出每处的位置，并检查同一句里有没有「因为」「所以」这类说明原因的词。拿场景里那句「CRITICAL：涉及车辆状态时你必须调用 get_vehicle_status」试一下，再把它改写成平常语气并写上原因。

想一想：第 1 种做法里，用户消息每次都不一样，为什么不影响前面系统提示词的缓存？

## 读后自测

1. 把线上 Agent 从旧模型换到新模型，提示词能原样沿用吗？迁移时具体要做哪几件事？——见「核心概念」第 1 节
2. 为什么官方不建议在提示词里写「CRITICAL：你必须……」？安全底线又该怎么写？——见「核心概念」第 2 节
3. 每次请求都要带当前时间、车速这类变化的信息，放在提示词的什么位置？为什么？——见「核心概念」第 3、4 节

## AI 历史知识点：Toolformer（2023，Meta AI）

**是什么**：Toolformer 是一个能自己学会调用工具的语言模型。它会在文本中插入 API 调用，自己决定什么时候调用、调用哪个工具、传什么参数。论文里用了五个工具：问答系统、维基百科搜索、计算器、日历和机器翻译。

**何时由谁提出**：Meta AI 的 Timo Schick、Thomas Scialom 等人在 2023 年 2 月 9 日提交到 arXiv，后来发表在 NeurIPS。

**为什么重要**：它几乎不需要人工标注。每个工具只写几条示例，让模型自己在大量文本里标出候选的 API 调用并实际执行，然后只保留「加上调用结果后，模型预测后文更准」的那些调用，再用它们微调。基础模型是 6.7B 参数的 GPT-J，在数学、事实填空和日期类任务上超过了约大 25 倍的 GPT-3，但在开放问答上仍不如 GPT-3。

**和今天的趋势有什么联系**：Toolformer 的标注是从「几条示例」的提示词开始的，这说明 few-shot 提示词也能当作数据生成的起点。今天的模型已经原生支持工具调用，问题反过来变成了「工具被过度调用」，所以才需要正文第 2 节讲的用平常语气写工具使用条件。

**原始来源**：
- [Toolformer: Language Models Can Teach Themselves to Use Tools（arXiv:2302.04761）](https://arxiv.org/pdf/2302.04761)
- [Meta AI Research 论文页](https://ai.meta.com/research/publications/toolformer-language-models-can-teach-themselves-to-use-tools/)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| 缓存失效规则 | 改工具定义：tools/system/messages 全部失效；改 system：system 和 messages 失效；改 messages、`tool_choice`、增删图片：只有 messages 失效；改思考参数或 `output_config.effort`：messages 一定失效，前两层视模型而定；切换 `speed`：system 和 messages 失效 |
| 缓存参数 | 最多 4 个 `cache_control` 断点；顶层加一个 `cache_control={"type": "ephemeral"}` 即自动缓存（占用一个断点）；默认 TTL 5 分钟，`"ttl": "1h"` 可选 |
| 缓存价格 | 5 分钟写入 1.25 倍、1 小时写入 2 倍基础输入价；读取 0.1 倍（Opus 5.5、Sonnet 5.5 为 0.05 倍，Fable 5.1、Mythos 5.1 为 0.025 倍） |
| 最小可缓存长度 | Opus 5.5、Sonnet 5.5、Haiku 5.5、Opus 5、Fable 5 等 512 token；Opus 4.8、Sonnet 5、Sonnet 4.6 为 1024；Opus 4.7 为 2048；Opus 4.6/4.5、Haiku 4.5 为 4096 |
| 用量字段 | 总输入 = `cache_read_input_tokens` + `cache_creation_input_tokens` + `input_tokens`；后两个缓存字段都为 0 表示没缓存 |
| 中途系统消息 | `{"role": "system", "content": ...}` 放在 `messages` 中；必须紧跟用户消息（或以服务端工具结果结尾的 assistant 消息）；要么是最后一条，要么后面紧跟 assistant；不能是第一条；Sonnet 5 不支持；Managed Agents 用 `system.message` 事件 |
| Prefill | Claude 4.6 及以后、Mythos Preview 不支持预填最后一轮 assistant 消息，返回 400；改用结构化输出或指令 |
| 思考配置 | 4.6 及以后推荐 `thinking={"type": "adaptive"}` + `output_config={"effort": ...}`；4.7 及以后设 `budget_tokens` 返回 400；Opus 5 默认 effort 为 `high`；`disabled` 配 `xhigh`/`max` 返回 400 |
| 官方提示词片段 | `<default_to_action>`（默认动手改）、`<do_not_act_before_instructions>`（只建议不动手）、`<use_parallel_tool_calls>`（提高并行调用）、`<investigate_before_answering>`（先读代码再下结论） |
| Opus 5 迁移清单 | 重扫 effort；删「最后验证一步」「用子 Agent 复核」；删「再检查一遍」；重新验证视觉相关的旧补丁；删「不要思考」类规则；升级 SDK；回答长度要明确写出 |
| LangChain | `create_agent(system_prompt=str 或 SystemMessage)`；动态提示词 `@dynamic_prompt`，函数签名 `(request: ModelRequest) -> str`；`request.messages`、`request.runtime.context`、`request.runtime.store`；官方建议先用静态提示词，需要时再加动态 |

### 来源说明

- 代码使用 LangChain 自带的 `GenericFakeChatModel`，没有调用真实模型（本次运行环境没有 API Key）；前缀按字符统计，只用于演示原理。
- OpenAI 两条动态的正文页面没能打开，日期和摘要来自 OpenAI 官方 RSS；历史知识点来自 arXiv 论文和 Meta AI 论文页（非 LangChain/Claude 来源）。

### 延伸学习

- Claude Academy《Building with the Claude API》的「Prompt evaluation」和「Prompt engineering」两章，包括「A typical eval workflow」和「Providing examples」两课。

### 参考资料

- [Prompting best practices - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Prompting Claude Opus 5 - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5)
- [Prompt caching - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Mid-conversation system messages and tool changes - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages)
- [Context engineering in agents - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/context-engineering)
- [Agents - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/agents)
- [A typical eval workflow · Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/a-typical-eval-workflow)
