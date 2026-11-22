# Day 25 · 成本与延迟优化（第二轮）（2026-11-22）

## 今日 AI 动态（2026-11-22）

过去 24 小时关注的厂商没有新的官方发布。

---

## 读前三问

1. 开了 prompt caching，命中率却很低。你会按什么顺序排查？
2. 给 Agent 加了「清理旧工具结果」，上下文变短了，账单反而变贵。为什么？怎么改？
3. 「小模型先上，不行再升级大模型」和「主模型报错就换备用模型」有什么区别？各自对缓存有什么影响？

## 从一个场景开始

第一轮我们给车机诊断 Agent 开了 prompt caching。上线后它要逐个 ECU 读 CAN 日志，每读一次，工具就返回约 3000 token。读到第 10 个 ECU，上下文涨到 3 万多，响应也慢了。

于是同事加了 LangChain 的 `ContextEditingMiddleware`，只保留最近 2 次工具结果，旧的换成 `[cleared]`。上下文确实降到了 1 万左右。

但月底账单出来，这个 Agent 的成本**涨了 50%**。

原因是两项优化在互相抵消。缓存靠的是「前缀和上次一模一样」，清理却每一轮都在改前缀的中间部分。第二轮要讲的就是这类**优化之间的冲突**：怎么算、怎么查、怎么排先后。

## 一句话总结

缓存要的是前缀稳定，清理、换模型、改参数都会破坏前缀。所以要先度量命中率，再决定改什么；必须改前缀时，就攒到一起少改几次。

## 先跑起来

用真实的 LangChain `ClearToolUsesEdit` 生成每一轮发给模型的消息，再用一个按 Claude 缓存规则写的小计费器来算钱。计费规则是：只在断点处写缓存，最多往回找 20 个块；读按 0.1 倍计价，写按 1.25 倍计价。整个脚本不需要 API Key。

```python
# cache_vs_clear.py —— 上下文清理 vs 提示缓存：清得越勤，缓存越难命中（langchain 1.5）
from langchain.agents.middleware import ClearToolUsesEdit
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage, ToolMessage

READ, WRITE = 0.1, 1.25                      # 读缓存 / 写缓存 相对基础输入价的倍数
count = lambda msgs: sum(len(m.content) for m in msgs)   # 用字符数代替 token 数

def turn_msgs(i):  # 第 i 轮：模型调一次诊断工具，工具回 3000 token 的车辆日志
    call = AIMessage("", tool_calls=[{"name": "read_can_log", "args": {"ecu": i}, "id": f"c{i}"}])
    return [call, ToolMessage("x" * 3000, tool_call_id=f"c{i}", name="read_can_log")]

def run(edit=None, persist=False, sys_breakpoint=False, turns=12):
    history = [SystemMessage("s" * 4000), HumanMessage("车机报故障灯，帮我逐个 ECU 查日志")]
    cached, cost, sizes = set(), 0.0, []     # cached：之前在断点处写入过的前缀
    for i in range(turns):
        history += turn_msgs(i)
        prompt = [m.model_copy() for m in history]
        if edit: edit.apply(prompt, count_tokens=count)           # 真实的 LangChain 清理逻辑
        if persist: history = prompt                              # 清理结果写回历史，下轮沿用
        keys = [tuple(m.content for m in prompt[:k]) for k in range(1, len(prompt) + 1)]
        hit = max((k for k in range(len(keys)) if keys[k] in cached and k >= len(keys) - 20), default=-1)
        read = count(prompt[:hit + 1]) if hit >= 0 else 0
        cost += read * READ + (count(prompt) - read) * WRITE      # 命中部分按读价，其余按写价
        cached.add(keys[-1])                                      # 自动缓存：断点在最后一个块
        if sys_breakpoint: cached.add(keys[0])                    # 显式断点：system 单独缓存
        sizes.append(count(prompt))
    return f"总成本 {cost/1000:6.1f}k  最后一轮上下文 {sizes[-1]:>6}  最大上下文 {max(sizes):>6}"

clear = lambda: ClearToolUsesEdit(trigger=20000, keep=2)
print("A 不清理            ", run())
print("B 每轮清理          ", run(clear()))
print("C 每轮清理+system断点", run(clear(), sys_breakpoint=True))
print("D 超阈值才清并写回   ", run(clear(), persist=True, sys_breakpoint=True))
```

**运行**：`pip install langchain && python cache_vs_clear.py`

**实测输出**（2026-11-22，Python 3，langchain 1.5.2）：

```
A 不清理             总成本   74.2k  最后一轮上下文  40019  最大上下文  40019
B 每轮清理           总成本  116.6k  最后一轮上下文  10109  最大上下文  19019
C 每轮清理+system断点 总成本   98.2k  最后一轮上下文  10109  最大上下文  19019
D 超阈值才清并写回    总成本   73.9k  最后一轮上下文  16091  最大上下文  19055
```

**该看什么**：
- B 的上下文只有 A 的四分之一，成本却高了 57%。原因是 LangChain 的清理中间件每次调用都在副本上重新清理，一旦超过阈值，每一轮被清的位置都会往后挪一格。前缀每轮都变，每轮几乎都要重新写缓存。
- C 给 system 单独加了一个断点，至少保住了 4000 token 的固定部分，成本降了一些，但仍然比不清理更贵。
- D 只在超过阈值时清一次，并把结果写回历史。在两次清理之间前缀保持稳定，成本和 A 持平，上下文却始终压在 2 万以内。

## 核心概念

### 1. 缓存失效是分层的：先查是哪一层变了

- **是什么**：Claude 按 **tools → system → messages** 的顺序拼出缓存前缀。某一层变了，这一层和它后面的层全部失效。
- **没有它会怎样**：只知道「命中率低」，不知道该去哪一层找原因。
- **对应你熟悉的**：Gradle 的增量构建。改了根模块，依赖它的模块全部要重新编译；改了叶子模块，只重编它自己。

排查时按层从上往下看：
- **tools 层**：工具定义或顺序变了，三层全部失效。常见原因是动态拼工具列表。
- **messages 层**：改了 `tool_choice`、加减了图片，或者改了 thinking / effort 设置，消息层就会失效。
- **工具会自动报原因**：Cache diagnostics（缓存诊断）是一个 beta 功能。请求时带上上一条响应的 id，API 会直接告诉你是哪一层先变了（`system_changed` / `tools_changed` / `messages_changed` / `model_changed`），以及因此多花了多少 token。

度量时看 usage 里的三个数：`cache_read_input_tokens`（读命中）、`cache_creation_input_tokens`（新写入）、`input_tokens`（断点之后没缓存的部分）。三者相加才是这次的总输入。

### 2. 清理和缓存的冲突：要清就攒着一次清够

- **是什么**：Claude 官方文档明确写着，清理工具结果「会让已缓存的提示前缀失效」。所以 Anthropic 服务端的 `clear_tool_uses_20250919` 提供了 `clear_at_least` 参数：这次至少能清掉这么多 token 才动手，清不够就不清。
- **没有它会怎样**：就是上面的 B。每轮只省几千 token 的上下文，却要重写一万多 token 的缓存。写缓存的价格是正常输入的 1.25 倍。
- **对应你熟悉的**：RecyclerView 用 `notifyItemChanged` 做局部刷新，而不是动不动就调用 `notifyDataSetChanged` 全量重绘。

还要注意：缓存**只在断点处写入**。清理点之后的内容，就算上一轮读过，也不会有现成的缓存可用，必须重新写。所以至少要给 tools/system 这类固定内容单独留一个断点，也就是 C 的做法。

### 3. 级联 vs 回退：都会换模型，触发条件不同

- **级联（cascade）**：先用小模型跑，结果**校验不过**就升级到大模型。这需要一个可靠的「成功/失败」判定，可以用 Day 20 的 schema 校验，也可以用 Day 21 的评测。
- **回退（fallback）**：主模型**报错**（限流、超时、服务不可用）时，自动换到备用模型。LangChain 的 `ModelFallbackMiddleware` 就是这种，它只捕获异常，不判断答得好不好。
- **共同的代价**：缓存是按模型分开存的，换模型就等于全量未命中（诊断里显示为 `model_changed`）。如果升级频繁，省下的单价会被重写缓存吃掉。
- **对应你熟悉的**：级联像图片先加载缩略图，用户点开才加载原图；回退像 CDN 节点挂了，切到备用域名。

### 4. 并行请求与缓存预热：先发一个，再发其余

- **是什么**：官方文档说明，缓存条目要等**第一个响应开始返回**后才可用。如果同时发出 5 个共享长前缀的子 Agent 请求，它们会各自写一遍缓存，没有一个能命中。
- **怎么做**：先发一个请求，等它开始返回，再把其余请求并发出去。这样多花一点首个请求的延迟，换来其余请求全部命中缓存。
- **对应你熟悉的**：冷启动时先初始化共享的单例，再并发启动依赖它的模块，避免每个模块各自初始化一遍。

## 高频面试题

**1. 命中率低，你怎么排查？**
- 先看 usage：三项 token 数里，读命中占多少、新写入占多少。
- 打开 cache diagnostics，直接看是哪一层先变了。
- 按层从上往下查：工具列表是否稳定、system 里有没有时间戳这类每次都变的值、`tool_choice`/图片/thinking 设置是否变了。
- 再看其他原因：前缀没到最小可缓存长度、两次请求间隔超过 TTL、并行请求没有先预热。

**2.（追问）为什么清理旧工具结果反而更贵了？**
- 清理改的是前缀中间，清理点之后的缓存全部失效。
- LangChain 的中间件每次调用都在副本上重新清理，超过阈值后清理点每轮都在移动。
- 改法：批量清理（服务端用 `clear_at_least`，或者把清理结果写回 state），并给 tools/system 单独加断点。
- 清理的主要目的是不超上下文窗口、保住回答质量，不是省钱，上线前要按任务成本算一遍账。

**3.（场景题）车机诊断 Agent 要并行调 5 个子 Agent 分析不同 ECU，它们共享 8k token 的车型手册。怎么设计才又快又省？**
- 手册放在 system 里，加显式断点，所有子 Agent 的工具列表和顺序保持一致。
- 先发 1 个子请求预热缓存，等它开始返回后，再并发发出其余 4 个。
- 每个子 Agent 只把摘要交回主 Agent，原始日志不进主上下文。
- 子 Agent 用小模型加低 effort，汇总时才用大模型；夜间批量复盘交给 Batch API。

**4.（追问）用户 1 小时内会多次回来提问，缓存 TTL 选 5 分钟还是 1 小时？**
- 5 分钟缓存写入是基础价的 1.25 倍，1 小时是 2 倍，读取价一样。
- 如果间隔经常超过 5 分钟，每次都要以 1.25 倍重写，选 1 小时更划算。
- 粗算：在 1 小时里，同一前缀用 5 分钟缓存要写两次或更多（2×1.25 > 2），就该换成 1 小时。
- 用线上的请求间隔分布来定，不要凭感觉选。

## 常见误区

1. **「上下文越短越省钱」**：在开了缓存的情况下不成立。缓存读取只要 0.1 倍价格（部分新模型更低），一段稳定的长前缀可能比一段每轮都变的短前缀更便宜。
2. **「有了回退中间件，小模型答错时也会自动升级」**：`ModelFallbackMiddleware` 只在模型调用抛异常时才换模型，答错不算异常。要按质量升级，得自己写校验和级联逻辑。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：缓存计费循环**（取自 `cache_vs_clear.py` 的 `run`）

```python
# 每一轮：先把新的一轮工具调用和结果追加进历史
for i in range(turns):
    history += turn_msgs(i)
    # 复制一份作为这次真正发给模型的消息，避免直接改历史
    prompt = [m.model_copy() for m in history]
    # 有清理策略就在副本上执行清理
    if edit: edit.apply(prompt, count_tokens=count)
    # 选择写回：清理结果成为新历史，下一轮前缀保持稳定
    if persist: history = prompt
    # 为每个位置算出「到这里为止的前缀」，作为缓存的键
    keys = [tuple(m.content for m in prompt[:k]) for k in range(1, len(prompt) + 1)]
    # 在最近 20 个位置里找最长的、之前写过缓存的前缀
    hit = max((k for k in range(len(keys)) if keys[k] in cached and k >= len(keys) - 20), default=-1)
    # 命中部分的 token 数，没命中就是 0
    read = count(prompt[:hit + 1]) if hit >= 0 else 0
    # 命中部分按读价算，剩下的按写价算
    cost += read * READ + (count(prompt) - read) * WRITE
    # 自动缓存：在最后一个块写入整条前缀
    cached.add(keys[-1])
    # 有 system 断点时，单独把 system 也写入缓存
    if sys_breakpoint: cached.add(keys[0])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `keys` 的 `range` 从 1 开始，到 `len(prompt) + 1` 结束，这样 `keys[k]` 对应的是前 `k+1` 条消息，和后面的 `prompt[:hit + 1]` 对得上。
- `max(..., default=-1)`：一个都没命中时，生成器是空的，不写 `default` 会直接报错。
- 是 `count(prompt) - read` 按写价计费，不是 `count(prompt)` 全部按写价算。

**昨日复习（只做第③步）**：第二轮 Day 24 的「审批三节点 propose / approve / execute」。只看你自己写的注释重写，重点核对：`expires_at` 写在 `propose` 里、`interrupt(s["request"])` 的返回值直接放进 state、复核顺序是「同意 → 摘要 → 时效 → 车况」。

## 动手练习（30 分钟）

在 `cache_vs_clear.py` 上改：
1. 加 TTL：给每轮加一个时间戳（例如每轮 90 秒，第 6 轮后车主停了 10 分钟）。缓存条目超过 300 秒就失效，看 A 和 D 的成本差距怎么变。
2. 加 1 小时 TTL 的选项：写价改成 2 倍、有效期改成 3600 秒，在同一组时间序列下和 5 分钟 TTL 比较。
3. 实现一个简化版的 `clear_at_least`：只有能清掉 ≥ 9000 token 时才清理，否则这一轮不动。和 D 比较，看清理次数和总成本。

提示：可以把 `cached` 从 `set` 改成 `dict`，键是前缀，值是写入时间。

## 读后自测

1. 开了 prompt caching，命中率却很低。你会按什么顺序排查？见第 7 节「核心概念 1」
2. 给 Agent 加了「清理旧工具结果」，上下文变短了，账单反而变贵。为什么？怎么改？见第 6 节「先跑起来」和第 7 节「核心概念 2」
3. 「小模型先上，不行再升级大模型」和「主模型报错就换备用模型」有什么区别？各自对缓存有什么影响？见第 7 节「核心概念 3」

## AI 历史知识点：推测解码 Speculative Decoding（2022，Leviathan、Kalman、Matias）

**是什么**：一种让大模型生成更快、输出却完全不变的解码方法。先让一个便宜的小模型连续「猜」出几个 token，再让大模型一次性并行验证这些猜测。猜对的直接采用，猜错的地方由大模型改正，然后接着往下猜。论文还设计了一种采样方法，保证最终输出的分布和大模型单独生成时完全一致。

**何时由谁提出**：2022 年 11 月 30 日以 arXiv 预印本发表，作者是 Yaniv Leviathan、Matan Kalman、Yossi Matias，后来发表在 ICML 2023。

**为什么重要**：逐个 token 生成本质上是串行的，生成 K 个 token 就要按顺序跑 K 次大模型。推测解码把其中「容易的部分」交给小模型，把大模型的工作变成可以并行的验证。论文在 T5-XXL 上测到 2–3 倍加速，输出完全一致，而且不需要重新训练，也不用改模型结构。

**和今天的趋势有什么联系**：它和今天讲的「级联」用的是同一个思路：便宜的先上，贵的只负责把关。区别在层级：级联在 Agent 层按整个任务切换模型，推测解码在推理引擎内部按 token 切换。两者都说明一件事，降成本和降延迟的关键，是不让最贵的资源去做最容易的工作。

来源：[arXiv:2211.17192](https://arxiv.org/abs/2211.17192v2)、[ICML 2023 论文集（PMLR v202）](https://proceedings.mlr.press/v202/leviathan23a.html)

## 参考区（用时查，不必通读）

### 细节速查

**Claude prompt caching**
- 写法：自动缓存是在请求顶层加 `cache_control: {"type": "ephemeral"}`；显式缓存是加在具体的块上。显式断点最多 4 个，自动断点也占其中一个。
- lookback（往回查找）窗口是 20 个块，断点本身算第 1 个；连续的 `tool_use` 块算 1 个位置，连续的 `tool_result` 块也算 1 个。缓存只在断点处写入。
- 最小可缓存长度：Opus 5.5 / Sonnet 5.5 / Haiku 5.5 / Fable 5.x 等是 512 token；Opus 4.8 / Sonnet 5 / Sonnet 4.6 是 1024；Opus 4.6 / 4.5、Haiku 4.5 是 4096。长度不够时不会报错，只是两个缓存字段都为 0。
- TTL：默认 5 分钟，可选 `"ttl": "1h"`；有效期从请求开始时计算。
- 价格倍数（相对基础输入价）：5 分钟写入 1.25 倍，1 小时写入 2 倍，读取 0.1 倍。例外：Opus 5.5 / Sonnet 5.5 读取 0.05 倍，Fable / Mythos 5.1–5.2 读取 0.025 倍。
- 失效规则：工具定义变化，三层全部失效；开关 web search / citations / speed，system 和 messages 失效；`tool_choice` 或图片变化，只有 messages 失效；thinking / effort 变化，messages 失效（tools/system 层取决于模型）。
- 并行：第一个响应开始返回后，缓存条目才可用。
- 错误：显式断点再加自动缓存，超过 4 个断点会报 400；Bedrock runtime 不支持顶层 `cache_control`，会报 400。
- usage：总输入 = `cache_read_input_tokens` + `cache_creation_input_tokens` + `input_tokens`；`cache_creation_input_tokens` 可以按 `ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens` 拆分。

**Cache diagnostics（beta）**：每次请求都带上 `diagnostics={"previous_message_id": 上一条响应的 id}`，第一次请求传 `null`。返回的 `response.diagnostics.cache_miss_reason.type` 有这几种：`model_changed`、`system_changed`、`tools_changed`、`messages_changed`、`previous_message_not_found`、`unavailable`；属于 `*_changed` 的会附带 `cache_missed_input_tokens`。只支持 Messages API，同一个 workspace 内有效。

**Claude context editing（服务端，beta 头 `context-management-2025-06-27`）**
- 策略：`clear_tool_uses_20250919`（清理工具结果）、`clear_thinking_20251015`（清理 thinking，两者同时用时这个放前面）。
- `clear_tool_uses` 参数：`trigger`（默认 10 万输入 token）、`keep`（默认保留 3 次工具调用）、`clear_at_least`（至少能清这么多才执行，无默认值）、`exclude_tools`、`clear_tool_inputs`（默认 false）。
- 响应中的 `context_management.applied_edits` 会给出 `cleared_tool_uses` 和 `cleared_input_tokens`。

**LangChain 中间件（langchain 1.5.2）**
- `ContextEditingMiddleware(edits=[ClearToolUsesEdit(...)], token_count_method="approximate"|"model")`。
- `ClearToolUsesEdit(trigger=100000, clear_at_least=0, keep=3, clear_tool_inputs=False, exclude_tools=(), placeholder="[cleared]")`。
- 两个和 Claude 服务端不同的地方（来自源码）：① 每次调用都在 `deepcopy` 出来的副本上重新清理，不会写回 state；② `clear_at_least` 的含义是「清到够数就停」，而 Claude 服务端是「清不够就不清」。
- `ModelFallbackMiddleware(first_model, *additional_models)`：只在调用报错时才换模型。
- `SummarizationMiddleware(model, trigger, keep=("messages", 20), ...)`；`ModelCallLimitMiddleware(thread_limit, run_limit, exit_behavior="end")`；`ToolCallLimitMiddleware(tool_name, thread_limit, run_limit, exit_behavior="continue")`。

**官方 cookbook 的实测数据**：前缀保持字节级稳定后便宜 44%；固定内容放前面、可变内容放后面，便宜 54%。

### 来源说明

- 示例里的计费器是本篇按官方规则写的简化模型：用字符数代替 token 数，忽略 TTL 和最小长度。只有清理逻辑是真实的 LangChain 代码。
- 今日动态：OpenAI 新闻列表、blog.google AI 板块、Z.ai 更新日志、Qoder 更新日志这次没能打开。已核实的最新官方条目（Anthropic IPO 交割，11-20）昨天已经收录。
- Claude Academy 课程里写的 Haiku 4.5 最小缓存长度是 4096 token，和当前文档一致。较新模型的值以文档为准。
- 历史知识点来自 arXiv 和 PMLR 页面，属于非 LangChain/Claude 来源。

### 延伸学习

- Claude Academy · Building with the Claude API：「Prompt caching」「Rules of prompt caching」「Prompt caching in action」三课
- Claude Cookbook · Cost Optimization on the Claude API

### 参考资料

- [Prompt caching - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Cache diagnostics - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics)
- [Context editing - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Cost Optimization on the Claude API - Claude Cookbook](https://platform.claude.com/cookbook/cost-optimization-cost-optimization)
- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Rules of prompt caching - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/rules-of-prompt-caching)
