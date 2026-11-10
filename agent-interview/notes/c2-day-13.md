# Day 13 · 上下文工程（第二轮）：清理别清错、压缩会丢细节、工具定义也占预算（2026-11-10）

## 今日 AI 动态（2026-11-10）

### 趋势观察

- Anthropic 这次发布把「大模型做顾问、小模型干活」写成了正式用法：Haiku / Sonnet / Opus 可以向 Fable 5.2 求助，Fable 5.2 也可以把简单任务分给小模型。多模型分工正从研究系统走进 API 的常规用法。
- 发布里的演示（1000 多个子代理连续跑 8 天、约 1000 个 Agent 30 小时完成形式化证明）说明，长时间、多 Agent 任务已经是旗舰模型的主打场景。这类任务能不能跑下去，很大程度取决于今天讲的上下文管理。

### 今日要点

- Anthropic 发布 Claude Fable 5.2 和 Claude Mythos 5.2（同一底层模型），输出速度比 5.1 最多快 30%，新增最多快 2.5 倍的 fast mode。

### 分公司动态

#### 1. Anthropic / Claude

**[Introducing Claude Fable 5.2 and Claude Mythos 5.2](https://www.anthropic.com/claude-fable-and-mythos-5-2)**（2026-11-09）

- Fable 5.2 和 Mythos 5.2 是同一个模型：Fable 带网络安全和生物安全防护，面向所有用户；Mythos 只开放给通过验证计划的机构。API 模型 ID 为 `claude-fable-5-2`。
- 标准价格为每百万 token 输入 8 美元、输出 40 美元；fast mode 价格翻倍，生成速度最多提升 2.5 倍，Claude Code 里已可用。
- 官方自测：FrontierSWE v2 上 69.9%（Opus 5.5 为 62.3%），AutomationBench 上 51.4%（Opus 5.5 为 42.5%）。新增「顾问」（advisor）和「编排者」（orchestrator）两种多模型协作用法。官方仍建议多数编码和知识工作从 Opus 5.5 开始。
- 安全方面，分类器误拦截比 Fable 5.1 少约 55%；在沙箱逃逸评测中，Mythos 5.2 使用外部凭据或文件的场景比例降到 3.1%（5.1 为 21%）。

**为什么重要**：「大模型当顾问 / 编排者 + 小模型执行」成为官方推荐的组合，面试谈多智能体架构和成本优化时可以直接引用。

### 动态来源

- [Introducing Claude Fable 5.2 and Claude Mythos 5.2 - Anthropic](https://www.anthropic.com/claude-fable-and-mythos-5-2)

---

> 第二轮说明：第一轮 Day 13 讲过上下文预算的分区、prompt caching 的原理和失效规则、三种上下文编辑工具和 LangChain 中间件写法。今天不再重复，只讲用这些工具时最容易踩的坑：清理清错了东西、压缩丢了细节、放的位置不对、工具定义本身撑爆上下文。

## 读前三问

1. 开了「清理旧工具结果」以后，Agent 回答时忘了用户的车型。为什么会这样？怎么修？
2. 工具结果清理、上下文压缩（compaction）、外部记忆，三者分别适合解决哪种上下文膨胀？
3. 车机助手接了 200 个工具，工具定义本身就占了几万 token。怎么处理，同时不破坏缓存？

## 从一个场景开始

第一轮你给车机助手加了「清理旧工具结果」，长对话的 token 账单降了一半。

一周后测试反馈：司机说「我这台车胎压和雨刮怎么设置」，助手先查了车型（A7 混动），再查了胎压、雨刮两章手册。最后的回答里，雨刮讲对了，胎压却给的是燃油版的数值。

翻日志发现：最后一次调用模型时，车型查询结果和胎压那一章都已经被替换成了 `[cleared]`。清理规则只保留最近 1 次工具结果，可这次回答要同时用到三次结果。

上下文管理的每个工具都在「省 token」和「丢信息」之间做取舍。今天讲怎么判断哪些能丢、哪些不能。

## 一句话总结

上下文管理要按「丢了能不能找回来」来选手段：能重查的工具结果可以清，不能重查的对话要压缩并指定保留什么，必须长期保留的写进外部记忆；工具定义也是上下文，太多就按需加载。

## 先跑起来

用 LangChain 的 `ContextEditingMiddleware` 清理旧工具结果，假模型把「最后一次调用时看到的上下文」记下来，对比加不加 `exclude_tools` 的区别。

```python
# context_clear.py —— 清旧工具结果时，别把后面还要用的“车型”清掉（脚本化假模型，无需 API Key）
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

@tool
def lookup_vehicle(vin: str) -> str:
    """按 VIN 查车型。"""
    return "A7 混动（2026 款）"

@tool
def search_manual(query: str) -> str:
    """检索用车手册。"""
    return f"《{query}》章节原文……" + "说明文字" * 40     # 模拟一大段手册原文

class Recorder(GenericFakeChatModel):             # 假模型：按剧本输出，并记下每次真正收到的上下文
    seen: list = []
    def bind_tools(self, tools, **kw):
        return self
    def _generate(self, messages, *a, **kw):
        self.seen.append(messages)
        return super()._generate(messages, *a, **kw)

call = lambda n, **a: AIMessage("", tool_calls=[{"name": n, "args": a, "id": f"{n}-{len(str(a))}"}])

def run(exclude):
    model = Recorder(messages=iter([
        call("lookup_vehicle", vin="LA7X2026"),
        call("search_manual", query="胎压"), call("search_manual", query="雨刮"),
        AIMessage("好的，按 A7 混动的手册回答……")]))
    edit = ClearToolUsesEdit(trigger=200, keep=1, exclude_tools=exclude)
    agent = create_agent(model, tools=[lookup_vehicle, search_manual],
                         middleware=[ContextEditingMiddleware(edits=[edit])])
    agent.invoke({"messages": [("human", "我这台车（VIN LA7X2026）胎压和雨刮怎么设置？")]})
    print(f"\n== exclude_tools={exclude}  最后一次调用模型时，它看到的工具结果：")
    for m in model.seen[-1]:
        if m.type == "tool":
            print(f"  {m.name:<15} -> {m.content[:24]}")

run(exclude=[])
run(exclude=["lookup_vehicle"])
```

**运行**：`pip install -U langchain langgraph && python context_clear.py`

**实测输出**（2026-11-10，Python 3.13，langchain 1.4.9，langgraph 1.2.20）：

```
== exclude_tools=[]  最后一次调用模型时，它看到的工具结果：
  lookup_vehicle  -> [cleared]
  search_manual   -> [cleared]
  search_manual   -> 《雨刮》章节原文……说明文字说明文字说明文字说明

== exclude_tools=['lookup_vehicle']  最后一次调用模型时，它看到的工具结果：
  lookup_vehicle  -> A7 混动（2026 款）
  search_manual   -> [cleared]
  search_manual   -> 《雨刮》章节原文……说明文字说明文字说明文字说明
```

**该看什么**：
- 第一段：车型和胎压两条结果都被清成了 `[cleared]`，模型生成最终回答时已经看不到它们，这就是场景里的事故。
- 第二段：`exclude_tools` 保住了车型，但胎压那章仍被清掉。原因是 `keep=1` 太小，本次回答还在用的结果不止一条。
- 清理发生在每次调用模型之前，被清掉的只是发给模型的那份副本，Agent 状态里的原始消息还在（见参考区）。

## 核心概念

### 1. 清理要分清「用完了」和「还在用」

- **是什么**：工具结果清理把旧的工具输出换成占位符，只保留最近几条。哪些不能清，由「保留最近几条」（`keep`）和「永不清理的工具」（`exclude_tools`）两个参数决定。
- **没有它会怎样**：按条数一刀切，会把「小而关键」的结果（车型、用户 ID、当前订单号）和「大而用完」的结果（手册原文、日志）一起清掉，模型就会用错前提。
- **对应你熟悉的**：`LruCache` 的淘汰。容量设太小，正在用的对象也会被踢出去；不能丢的东西应该放在缓存之外单独持有。

经验做法：短小、后续一直要用的结果（身份、车型、配置）排除在清理之外；`keep` 至少覆盖「一次回答通常要同时用到的结果数」。Anthropic 把清理称为最安全、最轻量的压缩方式，因为被清掉的内容可以再调一次工具拿回来，但重查要花时间和费用。

### 2. 压缩会丢细节，要告诉它保留什么

- **是什么**：压缩（compaction）是把旧的对话、推理和工具结果总结成一段摘要，再用摘要开启新的上下文。
- **没有它会怎样**：对话本身和模型的推理过程没法「重新调用工具」找回来，只靠清理解决不了。
- **对应你熟悉的**：日志轮转时只留汇总统计，原始日志删掉。统计够看趋势，但查某一条具体请求就找不到了。

Claude 官方 cookbook 做过实验：压缩后，高层结论 3 条全部保留，附录里的冷门数值 3 条全部丢失。两点要注意：
- 可以用 `instructions` 自定义摘要提示词，比如「保留车型、VIN、已确认的用户设置」。但它会**整个替换**默认提示词，而不是追加。
- Anthropic 建议调摘要提示词分两步：先追求「什么都不漏」，再删掉多余内容。

### 3. 按「丢了能不能找回来」选手段

| 膨胀来源 | 能不能找回 | 用什么 |
|---|---|---|
| 工具输出（手册原文、日志） | 能，重调工具 | 工具结果清理 |
| 对话和推理过程 | 不能 | 压缩，并写清保留项 |
| 跨会话都要记住的事实 | 会话结束就没了 | 外部记忆（结构化笔记） |

- **是什么**：三种手段处理的是不同的东西，可以叠加，不是互相替代。
- **没有它会怎样**：只开压缩，每次压缩都要多一次模型调用；只开清理，对话本身照样无限增长；两者都只管当前会话，下次开车又要从头问车型。

cookbook 的数据：用了记忆的第二次会话，读文件次数从 8 次降到 4 次。记忆的效果取决于笔记写得好不好，笔记太简略，Agent 还是得回去重读原文。

### 4. 位置也是上下文工程的一部分

- **是什么**：Claude 官方建议，超过 2 万 token 的长资料放在最前面，问题和指令放在最后。测试中这样做最多能提升 30% 的回答质量，多文档时效果最明显。
- **没有它会怎样**：问题夹在资料中间或开头，模型读到后面时注意力已被资料稀释，容易答偏。
- **对应你熟悉的**：Android 布局里「固定的放外层、变化的放内层」，减少重绘范围。

这条和缓存的规则正好一致：稳定的长资料放前面，正好作为可缓存的前缀；每轮变化的问题放最后，不影响前缀命中。多份资料用 XML 标签（如 `<document>`）分开并标注来源，必要时先让模型引用原文再作答。

### 5. 工具定义也占上下文

- **是什么**：每个工具的名称、描述、参数说明都会发给模型。接了多个 MCP 服务后，光工具定义就可能占约 5.5 万 token。Claude 的工具搜索（tool search）让大部分工具标记为延迟加载，模型需要时先搜索，只加载用得上的 3–5 个。
- **没有它会怎样**：上下文被工具定义占满；而且官方提到，可选工具超过 30–50 个后，模型选对工具的准确率会下降。
- **对应你熟悉的**：Android 的动态功能模块（Dynamic Feature Module），用到哪个功能再下载哪个，而不是把所有功能打进基础包。

延迟加载的工具不进入缓存前缀，被搜到后以引用块的形式追加在对话里，所以不会让前面的缓存失效。常用的 3–5 个工具保持直接加载。

## 高频面试题

**Q1（场景题）：车机 Agent 长对话里开了工具结果清理，之后回答开始用错车型。怎么排查和修？**
- 打出每次调用模型时的实际输入，看车型结果是否已被替换成占位符。
- 把车型这类「小而关键」的工具加进永不清理的名单，或在第一次查到时写入状态 / 系统提示，不依赖历史。
- 调大保留条数，覆盖一次回答要同时用到的结果数。
- 评测集加入「先查车型再问细节」的多轮用例，回归时检查。

**Q2：工具结果清理、压缩、外部记忆怎么选？**
- 能重调工具拿回来的输出 → 清理，成本最低。
- 对话和推理没法重取 → 压缩，并在摘要指令里写明必须保留的事实。
- 跨会话要用的事实 → 外部记忆。
- 三者可以叠加，先看实际是哪部分在膨胀，再选手段。

**Q3（追问）：清理会让缓存失效，那还值得清吗？怎么权衡？**
- 清理改了前缀中间的内容，之后的缓存要重新写入，写缓存比普通输入更贵。
- 设「每次至少清掉多少 token」（`clear_at_least`），一次清得多，才值得付一次重写缓存的代价。
- 触发阈值别设太低，避免频繁小规模清理。
- 用响应里的缓存统计字段验证实际命中率，而不是凭感觉。

**Q4（追问）：接了 200 个工具，怎么控制上下文？**
- 用工具搜索 + 延迟加载，只在需要时加载 3–5 个工具定义。
- 常用工具保持直接加载，缓存断点打在直接加载的工具上。
- 按场景拆子 Agent（导航、媒体、车控），每个只带自己的工具。
- 工具描述写清楚，否则搜索也搜不准。

## 常见误区

- **「压缩是无损的，开了就不用管」**：压缩一定会丢细节。关键事实要么写进摘要指令要求保留，要么单独存到外部记忆。
- **「工具越多 Agent 越强」**：工具定义本身占预算，工具多了选择准确率还会下降。工具要么精简，要么按需加载。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带保护名单的工具结果清理**（取自 `context_clear.py`）

```python
# 定义一次运行：exclude 是“永不清理”的工具名单
def run(exclude):
    # 假模型按剧本依次输出：先查车型
    model = Recorder(messages=iter([
        call("lookup_vehicle", vin="LA7X2026"),
        # 再分两次查手册的胎压、雨刮章节
        call("search_manual", query="胎压"), call("search_manual", query="雨刮"),
        # 最后给出文字回答
        AIMessage("好的，按 A7 混动的手册回答……")]))
    # 清理规则：上下文超过 200 token 就清，只保留最近 1 条，名单里的工具不清
    edit = ClearToolUsesEdit(trigger=200, keep=1, exclude_tools=exclude)
    # 创建 Agent：模型、两个工具、上下文编辑中间件
    agent = create_agent(model, tools=[lookup_vehicle, search_manual],
                         middleware=[ContextEditingMiddleware(edits=[edit])])
    # 用一个需要多跳的问题跑一遍
    agent.invoke({"messages": [("human", "我这台车（VIN LA7X2026）胎压和雨刮怎么设置？")]})
    # 打印标题
    print(f"\n== exclude_tools={exclude}  最后一次调用模型时，它看到的工具结果：")
    # 遍历最后一次调用模型时的输入
    for m in model.seen[-1]:
        # 只看工具结果消息
        if m.type == "tool":
            # 打印工具名和结果前 24 个字
            print(f"  {m.name:<15} -> {m.content[:24]}")
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `ContextEditingMiddleware(edits=[edit])` 的 `edits` 是列表，不能直接传 `edit`。
- `exclude_tools` 里写的是工具名字符串，和 `@tool` 修饰的函数名一致。
- 工具结果消息的类型是 `"tool"`，不是 `"tool_result"`。

**昨日复习（只做第③步）**：第二轮 Day 12 的「两个检索工具 + 检索预算」（`agentic_hops.py`）。只看你自己写的注释重写，重点核对：`search_manual` 找不到时返回 `NO_RESULT` 文字而不是空串；`ToolCallLimitMiddleware` 的 `tool_name` 是字符串、`run_limit=2`。

## 动手练习（30 分钟）

在 `context_clear.py` 的基础上：

1. 把 `keep` 改成 2，不加 `exclude_tools`，看最后一次调用时三条结果各是什么状态。找出「刚好够用」的最小组合。
2. 打印 `agent.invoke(...)` 返回的 `messages` 里的工具结果，确认 Agent 状态里的原文是否还在。
3. 给 `ClearToolUsesEdit` 加 `placeholder="[已清理，可重新调用 search_manual 获取]"`，想一想真实模型读到这句提示后会怎么做。

## 读后自测

1. 开了「清理旧工具结果」以后，Agent 回答时忘了用户的车型。为什么会这样？怎么修？——见第 1 节和 Q1
2. 工具结果清理、上下文压缩（compaction）、外部记忆，三者分别适合解决哪种上下文膨胀？——见第 3 节和 Q2
3. 车机助手接了 200 个工具，工具定义本身就占了几万 token。怎么处理，同时不破坏缓存？——见第 5 节和 Q4

## AI 历史知识点：Sparse Transformer 稀疏注意力（2019，OpenAI）

**是什么**：一种改造 Transformer 注意力的方法。标准注意力里，每个 token 都要和其他所有 token 计算关系，计算量和内存随长度平方增长。Sparse Transformer 让每个 token 只关注一部分位置（比如附近的一段，加上按固定间隔挑选的位置），把复杂度从 O(n²) 降到 O(n√n)。

**何时由谁提出**：2019 年 4 月 23 日，OpenAI 的 Rewon Child、Scott Gray、Alec Radford、Ilya Sutskever 在论文《Generating Long Sequences with Sparse Transformers》中提出。

**为什么重要**：它让同一个模型能处理几万步长的序列，并用于图像、音频和原始字节文本，在 Enwik8、CIFAR-10、ImageNet-64 上取得了当时最好的密度建模结果。论文还用了「重算注意力矩阵而不存储」来节省内存，并写了专门的快速注意力算子。这是「把上下文做长」这条路线上较早的一次系统性尝试。

**和今天的趋势有什么联系**：之后的长上下文工作大多沿着「不让每个 token 都看全部」或「让注意力算得更省」的方向推进，上下文窗口也从几千 token 扩到了今天的百万级。但窗口能装下，不等于模型能用好：今天讲的上下文腐化、清理、压缩，正是因为「窗口变大」解决不了「注意力被稀释」。

**原始来源**：
- [Generating Long Sequences with Sparse Transformers - arXiv:1904.10509](https://arxiv.org/abs/1904.10509)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `ClearToolUsesEdit`（LangChain） | `trigger`=100000、`clear_at_least`=0、`keep`=3、`clear_tool_inputs`=False（True 时把对应工具调用的参数清成空对象）、`exclude_tools`=()、`placeholder`=`"[cleared]"` |
| `ContextEditingMiddleware` | `edits` 默认 `[ClearToolUsesEdit()]`；`token_count_method`：`approximate`（默认）/ `model`；实现在 `wrap_model_call` 里，对消息做深拷贝后编辑，只影响本次发给模型的输入，**不写回 Agent 状态** |
| `SummarizationMiddleware` | `trigger` 可按 `fraction` / `tokens` / `messages`；单个条件、dict（AND）、列表（OR）；不设 `trigger` 就不会自动摘要；`keep` 默认 `("messages", 20)` |
| Claude 服务端压缩 | beta `compact-2026-01-12`，edit 类型 `compact_20260112`；`trigger` 默认 15 万 input tokens，最小 5 万；`instructions` 整体替换默认提示词；`pause_after_compaction=True` 时返回 `stop_reason: "compaction"`；`compaction` 块必须回传，API 忽略它之前的内容；压缩前的 thinking 块不会带过去；用请求本身的模型做摘要，不能换便宜模型；有工具时可能误调工具，可在指令里写「不要调用工具」；总计费 token 要累加 `usage.iterations` |
| 压缩 + 缓存 | 在 system 末尾打断点，压缩后 system 缓存不受影响；`compaction` 块上也可以加 `cache_control` |
| Python SDK 坑 | 回传时用 `client.beta.messages` 或 `model_dump(exclude_none=True)`；普通 `model_dump()` 会带上 `text: null` 导致 400 |
| cookbook 实验数据 | 8 份各约 4.1 万 token 的文档：不管理时 5 轮涨到 33.5 万 token；压缩（阈值 18 万）峰值 16.9 万，摘要约 2783 token；清理（阈值 3 万、keep 4、clear_at_least 1 万）峰值 17.3 万；记忆让第二次会话读文件从 8 次降到 4 次；压缩后高层事实 3/3 保留、冷门数值 0/3 保留 |
| 工具搜索 | 类型 `tool_search_tool_regex_20251119` / `tool_search_tool_bm25_20251119`；工具上设 `defer_loading: true`；所有工具定义仍要每次发送；至少一个工具不能延迟加载；延迟加载的工具不能带 `cache_control`（400）；默认返回 5 个结果；每次请求最多 1 万个延迟工具；仅 Messages API，Managed Agents 不支持 |
| 长上下文提示 | 2 万 token 以上：长资料放前、问题放后（最多提升 30%）；多文档用 `<documents>` / `<document index="n">` / `<source>` / `<document_content>`；先让模型引用原文再回答 |

### 来源说明

- 演示用的是按剧本输出的假模型，只用来展示中间件对上下文的实际修改；真实模型在被清理后会不会重新调用工具，未实测（无 API Key）。
- 历史知识点引用的是 arXiv 论文页（非 LangChain/Claude 来源）。

### 延伸学习

- Claude cookbook「Context engineering: memory, compaction, and tool clearing」：三种手段在同一个任务上的对比实验，适合动手复现。

### 参考资料

- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Effective context engineering for AI agents - Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Context engineering: memory, compaction, and tool clearing - Claude Cookbook](https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools)
- [Compaction at a token threshold - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/compaction-threshold)
- [Long context prompting tips - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/long-context-tips)
- [Tool search tool - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
