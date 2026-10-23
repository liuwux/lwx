# Day 25 · 成本与延迟优化（2026-10-23）

## 读前三问

1. 为什么说 Agent 的成本要按「每完成一个任务」算，而不是按 token 单价算？
2. Prompt caching 为什么能同时省钱又降延迟？什么写法会让缓存一直不命中？
3. 车机语音助手「说完话要等 3 秒才有反应」，你会从哪几个方向优化？

## 从一个场景开始

车机助手内测一个月，财务和产品同时找上门。

财务说：账单是预算的 4 倍。一查，所有请求都走最强的模型，「空调调到 24 度」这种一句话指令，和「规划去机场的路线顺便预冷车厢」花的是同一档钱。

产品说：车主说完话要等两三秒才有声音，很多人以为没听见又说一遍，结果一次任务被执行两次。

还有一条日志让人后背发凉：某次「找停车位」，工具一直返回「暂无空位」，模型就一直换关键词重查，一个任务调了 40 次模型。

这三件事就是今天的主题：**成本与延迟优化**。核心不是「换便宜模型」，而是让每次调用都花在该花的地方。

## 一句话总结

先按任务算账；简单请求走小模型，难的才升级；稳定的前缀用缓存复用；用流式和并行让用户更早听到回应；给每个任务设调用次数和输出上限。

## 先跑起来

一个文件演示三件事：按难度路由模型、每次任务最多 3 次模型调用、按任务记账。无 API Key，模型用剧本代替（记录自己被调用的档位）；路由和预算中间件是真实库代码。

```python
# car_cost.py —— 车机 Agent 省钱三件套：按难度路由模型 + 调用次数预算 + 按任务记账（langchain 1.4.7）
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware, wrap_model_call
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI
from langchain_core.tools import tool

PRICE = {"small": 1, "big": 5}   # 相对单价：大模型一次调用 ≈ 小模型 5 次
calls = []                       # 记账：本次任务每次模型调用用的是哪档

class Scripted(GenericFakeChatModel):  # 剧本模型：代替真实 Claude，记录自己被调用
    tier: str = "small"
    bind_tools = lambda self, tools, **kw: self
    def _generate(self, *a, **kw):
        calls.append(self.tier)
        return super()._generate(*a, **kw)

def tc(name, **args): return AI("", tool_calls=[{"name": name, "args": args, "id": name}])

@tool
def set_ac(temp: int) -> str:
    """设置空调温度"""
    return f"空调已设为 {temp} 度"
@tool
def find_parking(area: str) -> str:
    """查找附近停车场"""
    return "暂无空位"

small = Scripted(tier="small", messages=iter([tc("set_ac", temp=24), AI("好的，24 度。")]
                                             + [tc("find_parking", area="公司")] * 9))
big = Scripted(tier="big", messages=iter([tc("set_ac", temp=22), AI("已按行程预冷到 22 度。")]))

@wrap_model_call  # 路由：短指令走小模型，复杂请求才上大模型
def route(request, handler):
    text = request.state["messages"][0].content
    hard = len(text) > 12 or "规划" in text
    return handler(request.override(model=big if hard else small))

budget = ModelCallLimitMiddleware(run_limit=3, exit_behavior="end")  # 每次任务最多 3 次模型调用
agent = create_agent(small, tools=[set_ac, find_parking], middleware=[budget, route])

for q in ["空调调到 24 度", "规划一下：半小时后出发去机场，到之前把车里预冷好", "帮我找个停车位"]:
    calls.clear()
    r = agent.invoke({"messages": [{"role": "user", "content": q}]})
    print(f"【{q}】→ 调用 {calls}，成本 {sum(PRICE[t] for t in calls)}，结尾：{r['messages'][-1].content[:60]}")
```

**运行**：`pip install langchain && python car_cost.py`

**实测输出**（2026-10-23，Python 3.13，langchain 1.4.7）：

```
【空调调到 24 度】→ 调用 ['small', 'small']，成本 2，结尾：好的，24 度。
【规划一下：半小时后出发去机场，到之前把车里预冷好】→ 调用 ['big', 'big']，成本 10，结尾：已按行程预冷到 22 度。
【帮我找个停车位】→ 调用 ['small', 'small', 'small']，成本 3，结尾：Model call limits exceeded: run limit (3/3)
```

**该看什么**：
- 一个任务至少调两次模型（一次发工具调用、一次总结结果），所以成本要按任务算，不能只看单次。
- 简单指令全程走小模型，成本是复杂请求的 1/5。路由函数每轮都会执行，所以同一任务里的每次调用都按同一规则选模型。
- 第三个任务剧本里本来会无限重查，预算中间件在第 3 次调用后直接结束，失控循环被截断在 3 次。

## 核心概念

### 1. 按「每完成一个任务」算账

- **是什么**：衡量成本的单位是「完成一个任务花了多少钱」，延迟也分两种：**首 token 时间**（TTFT，发出请求到收到第一个字）和**总耗时**。
- **没有它会怎样**：只比单价会选错模型。Anthropic 官方建议按「每个完成的任务」比较：便宜模型如果要多绕几轮、多失败几次，每个任务未必更省。另外 Agent 每轮都要把整段对话重新发一遍，官方估计不开缓存时成本大致随轮数的平方增长。
- **对应你熟悉的**：Android 优化启动时不看单个方法耗时，而是看冷启动总时长和首帧时间。

先在 trace（Day 22）里按任务统计调用次数、输入/输出 token、缓存命中，再动手优化。

### 2. 模型路由：好钢用在刀刃上

- **是什么**：在每次调用模型前，按请求难度选模型：一句话指令走小模型（如 Haiku），多步规划才用大模型。除了换模型，还可以调 **effort**（让模型想多深，低档更快更便宜）。
- **没有它会怎样**：80% 的简单请求在付「最难请求」的价钱。
- **对应你熟悉的**：图片加载按场景选尺寸：列表用缩略图，详情页才加载原图。

路由规则可以是关键词、长度，也可以是一个小模型分类器。Anthropic 文档还给了一个进阶思路：**先用低档跑，只把失败的重跑高档**，实测比全部用高档便宜约 40%、分数还更高。前提是你有可靠的「成功/失败」判定（Day 21 的评测）。

### 3. Prompt caching：别让模型反复读同一段话

- **是什么**：把请求里不变的前半段（工具定义、system prompt、早期对话）缓存起来，下次请求前缀完全相同时直接复用。读缓存约为正常输入价的 1/10，写缓存略贵（默认 5 分钟有效期写入价 1.25 倍）。
- **没有它会怎样**：Agent 每轮都重读几千上万 token 的工具和系统提示，又慢又贵。官方把它列为成本优化的**第一杠杆**，健康的 Agent 循环约 84% 的输入来自缓存。
- **对应你熟悉的**：OkHttp 的 HTTP 缓存，但要求是**逐字节一致**，改一个字就失效。

缓存按 **工具 → system → messages** 的顺序匹配前缀，前面任何一处变了，后面全部失效。最常见的坑：把当前时间、车速这类每次都变的值写进 system prompt 开头，导致每次都不命中。正确做法是把变化的内容放在最后。

```python
client.messages.create(model="claude-haiku-5-5", max_tokens=512,
    cache_control={"type": "ephemeral"},     # 自动缓存：断点随对话增长后移
    system=CAR_RULES,                        # 几千 token 的固定车控规则，放前面
    messages=[{"role": "user", "content": f"车速 {speed}km/h。{query}"}])  # 变化的值放最后
```

（片段未实测：本次运行环境无 API Key。写法取自官方 Prompt caching 文档。）

### 4. 流式与并行：让用户更早听到回应

- **是什么**：**流式**是模型边生成边返回，语音助手可以拿到第一句就开始 TTS 播报；**并行**是互不依赖的工具调用（查天气、查路况）同时发出，而不是一个接一个。
- **没有它会怎样**：用户等的是「整段生成完 + 所有工具串行跑完」的总时间。
- **对应你熟悉的**：列表分页加载、协程 `async { } + awaitAll()` 并发请求。

注意区分：流式只改善**感知延迟**（首字更早），总耗时不变；并行才真正缩短总耗时。不急的离线任务（夜间批量生成行程摘要、跑评测）用 Batch API，价格打五折，但结果最长 24 小时内返回。

### 5. 预算上限：给失控循环装保险丝

- **是什么**：给每个任务设模型调用次数上限、工具调用次数上限和单次输出上限（`max_tokens`）。
- **没有它会怎样**：工具一直失败时模型会一直重试，就像场景里的 40 次调用。
- **对应你熟悉的**：网络请求的超时和重试次数上限。

一个反直觉的点：官方说明 **调低 `max_tokens` 不会让模型说得更省**，只会把回答截断，被截断的部分照样计费。想让输出短，要在提示词里直接要求「只回一句话」。输出 token 单价通常是输入的 5 倍，在 Agent 循环里还会被下一轮当输入再读一遍。

## 高频面试题

**1. 你的 Agent 成本太高，排查和优化的顺序是什么？**
- 先度量：按任务统计调用次数、输入/输出 token、缓存命中率。
- 免费的先做：开 prompt caching、把变化内容移出前缀、删掉冗余工具结果和过时提示词。
- 再做取舍：模型路由、调低 effort、离线任务走 Batch。
- 最后兜底：调用次数上限和账户级消费上限。

**2.（追问）开了 prompt caching，命中率却很低，可能是什么原因？**
- 前缀里有每次都变的值（时间戳、车速、请求 ID）。
- 工具列表顺序或描述在变，导致从工具层开始全部失效。
- 中途改了 effort、thinking 或 `tool_choice` 等设置。
- 前缀太短没到最小缓存长度，或两次请求间隔超过有效期。

**3.（场景题）车机助手说完话要等 3 秒才出声，怎么优化？**
- 流式输出 + 第一句话就送 TTS，先播「好的，正在查」。
- 简单车控指令走小模型 + 低 effort，或用本地规则直接执行，不进大模型。
- 查天气、查路况等无依赖的工具并行调用。
- 固定的车控规则和工具定义开缓存，降低首 token 时间。

**4.（追问）用小模型路由，怎么保证质量不掉？**
- 用评测集分别测各档模型在各类请求上的成功率，按任务成本对比。
- 路由规则保守：拿不准的走大模型。
- 小模型失败或结果校验不过时自动升级到大模型重跑。
- 线上按路由分组监控成功率和用户重说率。

## 常见误区

1. **「换个便宜模型就省钱了」**：便宜模型绕路多、失败多，每个任务反而可能更贵。要按任务成本对比。
2. **「流式输出能让 Agent 变快」**：流式只让首字更早出现，总耗时不变；要缩短总耗时，靠并行、少调用、选更快的模型。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：剧本模型记账 + 按难度路由 + 调用次数预算**（取自 `car_cost.py`）

```python
# 定义一个剧本模型，代替真实大模型，并能记录自己被调用
class Scripted(GenericFakeChatModel):
    # 档位字段：小模型还是大模型，默认小模型
    tier: str = "small"
    # 绑定工具时什么都不做，直接返回自己
    bind_tools = lambda self, tools, **kw: self
    # 重写生成方法：每被调用一次就记一笔账
    def _generate(self, *a, **kw):
        # 把当前档位追加到记账列表
        calls.append(self.tier)
        # 交给父类按剧本返回下一条消息
        return super()._generate(*a, **kw)

# 声明一个包住「模型调用」的中间件，用来做路由
@wrap_model_call
# 接收本次请求和真正执行调用的处理函数
def route(request, handler):
    # 取出用户的第一句话
    text = request.state["messages"][0].content
    # 判断难度：句子长或提到「规划」就算难
    hard = len(text) > 12 or "规划" in text
    # 用替换了模型的请求去调用，难的走大模型
    return handler(request.override(model=big if hard else small))

# 预算中间件：每次任务最多调 3 次模型，超了就正常结束
budget = ModelCallLimitMiddleware(run_limit=3, exit_behavior="end")
# 组装 Agent：默认小模型，挂上两个工具和两个中间件
agent = create_agent(small, tools=[set_ac, find_parking], middleware=[budget, route])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 换模型用 `request.override(model=...)` 生成新请求，不是直接给 `request.model` 赋值。
- 一定要 `return handler(...)`：忘了 `return`，模型结果就丢了。
- 预算参数名是 `run_limit`（单次任务）；`thread_limit` 是整条会话，需要 checkpointer。

**昨日复习（只做第③步）**：Day 24 的「输入护栏 `@before_agent` + 条件审批 `when` + edit 恢复」。只看你自己写的注释重写那三段，重点核对：`can_jump_to=["end"]` 是列表，返回里 `"jump_to": "end"` 是字符串。

## 动手练习（30 分钟）

在 `car_cost.py` 上改：

1. 把路由规则换成关键词表：包含「规划」「顺便」「然后」的走大模型，其余走小模型。造 3 句测试话术，打印每句走了哪档。
2. 加「失败升级」：在 `route` 里先用小模型调用，如果返回的消息内容包含「无法」，就改用大模型再调一次。观察记账列表的变化。
3. 给 `find_parking` 加 `ToolCallLimitMiddleware(tool_name="find_parking", run_limit=2)`，对比它和模型调用上限截断的位置有什么不同。

## 读后自测

1. 为什么按任务算成本，而不是按 token 单价？→ 见核心概念 1、误区 1
2. Prompt caching 为什么又省钱又降延迟？什么写法让缓存不命中？→ 见核心概念 3、面试题 2
3. 车机助手出声慢，从哪几个方向优化？→ 见核心概念 4、面试题 3

---

## 参考区（用时查，不必通读）

### 细节速查

**Claude Prompt caching**
- 用法：顶层 `cache_control={"type":"ephemeral"}`（自动，断点随对话后移）；或在单个块上标（显式），每请求最多 4 个断点。
- 有效期：默认 5 分钟（写入 1.25×，命中即续期）；`"ttl":"1h"`（写入 2×）。混用时长 TTL 在前。
- 读价：多数模型 0.1×；Opus 5.5 / Sonnet 5.5 为 0.05×；Fable 5.1 为 0.025×。命中不计入速率限制。
- 最小长度：Opus 5.5 / Sonnet 5.5 / Haiku 5.5 为 512 token；Sonnet 4.6 等 1024；Haiku 4.5 等 4096。不够长时静默不缓存。
- 失效：改工具 → 全部失效；改 system / 联网搜索开关 → system 及之后；改 `tool_choice`、thinking、`output_config.effort`、增删图片 → messages。
- 计量：`cache_creation_input_tokens`、`cache_read_input_tokens`、`input_tokens`（三者之和为总输入）。
- 预热：`max_tokens: 0` 的请求只写缓存不生成。

**Claude 成本 / 延迟其他杠杆**
- Batch API：所有 token 五折，24 小时内返回，适合评测、回填、定时任务。
- effort：`output_config={"effort": "low"}`；官方建议在自己流量上扫 2–3 档，每档单独会话（中途改 effort 会使缓存失效）。
- 输出：输出单价约为输入 5 倍；Agent 场景建议 `max_tokens` 设 64000 并流式；`stop_reason: max_tokens` 当失败处理。
- 其他：工具多时用 tool search（`defer_loading`）；大数据文件上传后让代码执行去查，不要贴进提示词；任务边界清理过期工具结果。
- 延迟指标：baseline latency、TTFT；手段：选更快模型、精简提示与输出、流式。

**LangChain 成本相关中间件**（`from langchain.agents.middleware import ...`）
- `wrap_model_call` + `request.override(model=...)`：动态选模型。
- `ModelCallLimitMiddleware(thread_limit, run_limit, exit_behavior="end"|"error")`；`thread_limit` 需 checkpointer。
- `ToolCallLimitMiddleware(tool_name, thread_limit, run_limit, exit_behavior="continue"|"error"|"end")`。
- `ModelFallbackMiddleware(first_model, *additional_models)`：主模型出错时按序降级。
- `LLMToolSelectorMiddleware(model, max_tools, always_include)`：小模型先筛工具，减少提示 token。
- `SummarizationMiddleware(model, trigger=("tokens", n), keep=("messages", 20))`：压缩旧历史。
- `ModelRetryMiddleware(max_retries=2, backoff_factor=2.0, initial_delay=1.0, jitter=True)`：重试会增加延迟。
- 计量：`AIMessage.usage_metadata`、`get_usage_metadata_callback()`（按模型名汇总）。

### 来源说明

- 示例代码的模型是剧本，成本为相对单价，不是真实账单；prompt caching 片段未实测（无 API Key）。
- 本篇全部来自 LangChain 和 Anthropic 官方资料；文中百分比为 Anthropic 内部测试数据，仅供参考方向。

### 延伸学习

- Claude Academy · Building with the Claude API ·「Prompt caching」「Rules of prompt caching」「Prompt caching in action」
- LangChain 文档 · Prebuilt middleware、Models（动态模型、token 用量）

### 参考资料

- [Optimizing for cost and intelligence - Claude Platform Docs](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)
- [Prompt caching - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Reducing latency - Claude Platform Docs](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-latency)
- [Rules of prompt caching - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/rules-of-prompt-caching)
- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Models - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/models)
- [Agents - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/agents)
