# Day 13 · 上下文工程：上下文预算、prompt caching、上下文污染与隔离（2026-10-11）

## 一句话总结

上下文工程就是**每一轮都决定"这次往模型窗口里放哪些 token"**，目标是 Anthropic 那句话："用尽可能少的高信号 token，最大化得到期望结果的概率"。三件事：**预算**（窗口有限，token 越多注意力越分散，即 context rot）、**复用**（稳定内容放前面、打 prompt caching 断点，读缓存只要 0.1 倍价格）、**隔离**（工具结果清理、compaction 压缩、子 Agent 独立窗口，防止无关或有害内容污染后续推理）。

## 核心概念

Day 9 讲记忆时提到过裁剪、摘要和 context editing，Day 10/11 提到过"小知识库直接放进上下文 + 缓存"。今天把这些串成一套方法：**上下文是一种预算，要分配、要复用、要隔离**。

### 1. 上下文工程 vs Prompt 工程

| | Prompt 工程（Day 2） | 上下文工程 |
|---|---|---|
| 关注点 | 指令怎么写、怎么组织 | 每次推理时窗口里**放哪些 token**（system、工具、历史、工具结果、外部数据） |
| 时机 | 一次性写好 | **迭代式**：每次调用模型前都要重新筛选 |
| 典型问题 | 指令不清、格式不对 | 上下文太长、关键信息被淹没、旧结果干扰、缓存失效 |

LangChain 官方文档的判断："更多时候" Agent 失败是因为**缺了正确的上下文**，而不是模型能力不够。

**为什么要有预算？** Anthropic 的解释：Transformer 里 n 个 token 两两之间都有注意力关系（n² 个），上下文越长，注意力越分散，模型从中准确回忆信息的能力越差，这叫 **context rot**。所以上下文是一种**有限的注意力预算**，"窗口还没满"不等于"放进去没坏处"。

### 2. 上下文里有什么：LangChain 的三类划分

```
┌──────────── Model context（每次调用模型时的输入，临时的）────────────┐
│ system prompt │ tools 定义 │ messages 历史 │ response_format │ model  │
└────────────────────────────────────────────────────────────────────┘
        ▲ wrap_model_call / @dynamic_prompt：只改本次调用，不写回 state
Tool context（工具读写的 state / store / runtime）——持久
Life-cycle context（模型与工具调用之间：摘要、护栏、日志）——持久
        ▲ before_model / after_model / SummarizationMiddleware：会写回 state
```

面试抓住一个区分即可：**临时修改**（只影响这一次调用，比如按用户角色过滤工具、注入当前文件）和**持久修改**（写回 state，比如把旧消息换成摘要，以后每次调用都生效）。

### 3. 上下文预算：先量，再分

```
200K 窗口（举例）
├─ 固定区   system + tools + 常驻文档     ← 稳定，打缓存断点
├─ 历史区   多轮消息 + 工具调用记录          ← 增长最快，需要清理或压缩
├─ 本轮区   当前问题 + 动态数据（时间、车况）  ← 每次都变，放在最后
└─ 输出区   max_tokens                     ← 也要预留
```

- **先量**：Claude 的 `client.messages.count_tokens(...)` 接受和 `messages.create` 相同的参数，返回 `input_tokens`。**免费**，但有单独的 RPM 限制；结果是估算值。注意 Claude 4.7 及以后的模型换了新分词器，同样文本约多 30% token，换模型要重新量。
- **再分**：给每一区设上限，超了就按"成本从低到高"处理：先清旧工具结果 → 再压缩历史 → 再把子任务交给子 Agent。

### 4. 复用：Prompt Caching

**原理**：缓存的是**前缀**。请求按 `tools → system → messages` 的顺序拼接，从开头到断点为止**逐字节相同**才能命中。

两种用法（Claude 官方文档）：
- **自动缓存**：请求顶层加 `cache_control={"type": "ephemeral"}`，系统把断点放在最后一个可缓存块上，并**随对话增长自动后移**，适合多轮对话。
- **显式断点**：在某个内容块上加 `cache_control`，精确控制缓存到哪里。每个请求**最多 4 个断点**，自动缓存占用其中一个。

关键数字（以官方文档为准，会变）：

| 项 | 值 |
|---|---|
| TTL | 默认 5 分钟（每次命中免费刷新）；可选 `"ttl": "1h"` |
| 写缓存价格 | 5 分钟：基础输入价 × **1.25**；1 小时：× **2** |
| 读缓存价格 | 基础输入价 × **0.1**（Opus 5.5 / Sonnet 5.5 为 0.05） |
| 最小可缓存长度 | 随模型不同：512 / 1024 / 2048 / 4096 token（如 Sonnet 4.6 是 1024，Haiku 4.5 是 4096）；不够长**不报错、只是不缓存** |
| 回看窗口 | 每个断点最多往前检查 20 个位置 |
| 隔离 | 不同组织之间**永不共享**；Claude API 上还按 workspace 隔离 |

**usage 怎么读**：`总输入 = cache_read_input_tokens + cache_creation_input_tokens + input_tokens`。后两个缓存字段都是 0，说明这次没缓存上。

**什么会让缓存失效**（官方表格，面试常考）：

| 改了什么 | tools 缓存 | system 缓存 | messages 缓存 |
|---|---|---|---|
| 工具定义（名称、描述、参数、顺序） | ✘ | ✘ | ✘ |
| 开关 web search / citations | ✓ | ✘ | ✘ |
| `tool_choice` | ✓ | ✓ | ✘ |
| 增删图片 | ✓ | ✓ | ✘ |
| thinking 参数 | 视模型 | 视模型 | ✘ |

规律：**改动越靠前，失效范围越大**。所以：
1. 稳定的放前面，变化的放后面。**system prompt 里不要放时间戳、请求 ID**，这些动态信息放到断点之后的 user 消息里。
2. 工具列表固定且顺序稳定，JSON schema 的 key 顺序也要固定。
3. 历史只追加、不修改，assistant 内容和工具结果原样回传。
4. 中途不要换模型（缓存按模型区分），路由、A/B 测试、降级都会导致缓存失效。

两个新能力，用来解决"想改指令又不想丢缓存"：
- **对话中途的 system 消息**：在 `messages` 末尾追加 `{"role": "system", ...}`，前面的前缀不变，缓存照常命中（Sonnet 5 不支持）。
- **对话中途增删工具（beta `inline-tools-2026-09-15`）**：一开始声明完整工具集，之后用 `tool_removal` / `tool_addition` 块收回或提供工具，不改 `tools` 数组本身。

**排查缓存未命中**：Claude API 的 **cache diagnostics**：每轮请求带 `diagnostics={"previous_message_id": 上一轮 id}`，返回 `cache_miss_reason.type`，如 `system_changed`（常见原因是 system 里插了时间戳）、`tools_changed`、`messages_changed`、`model_changed`。只报告**最早的分歧点**，先修那一处。

Claude Academy 的讲法："缓存只在相同内容被重复发送时有用"，最适合缓存的是**大 system prompt、复杂工具 schema**，因为它们很少变。

### 5. 污染与隔离

**上下文污染**是指窗口里留着对后续推理有害的内容，常见四种：

| 污染源 | 表现 | 隔离手段 |
|---|---|---|
| 用过的大段工具结果 | 早已用完的检索、日志结果仍占着窗口，稀释注意力 | **工具结果清理**（最轻量） |
| 冗长的历史 | 早期细节淹没当前目标 | **compaction**（摘要后开新窗口）/ 结构化笔记 |
| 子任务的探索过程 | 搜了 20 次、试错 5 次的中间过程全进主上下文 | **子 Agent**：独立干净窗口，只回传 1000~2000 token 的精炼结果 |
| 不可信内容 | 网页、文档中夹带的注入指令（Day 12） | XML 标签包裹、声明"只当数据"、高风险工具需审批 |

Claude API 对应的三种工具：

```
越轻 ───────────────────────────────────────────────> 越重
clear_tool_uses_20250919    compact_20260112           子 Agent / memory tool
（清旧工具结果，换占位符）   （摘要旧轮次，服务端做）    （换一个窗口 / 写到外部）
beta: context-management-    beta: compact-2026-01-12   架构层设计
2025-06-27
```

- **工具结果清理**：默认输入超过 100K token 时触发，保留最近 3 次工具调用，`exclude_tools` 可指定永不清理的工具。**清理会让缓存失效**，所以有 `clear_at_least`：每次至少清掉这么多，否则不清，保证"破坏缓存"这笔代价值得付。
- **服务端 compaction**：官方说它是长对话的**首选**策略。`trigger` 默认 150K、最小 50K；响应开头会出现 `compaction` 块，**必须原样回传**，之后 API 会忽略它之前的内容。和缓存配合：在 system 末尾打断点，compaction 不会让 system 缓存失效。注意：`usage` 顶层不含压缩那一步的 token，要对 `usage.iterations` 求和才是真实成本。
- **Thinking 清理**（`clear_thinking_20251015`）：和工具结果清理组合使用时必须排在 `edits` 第一位。

LangChain v1 里同样的能力都是 middleware，可以叠加使用：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit
from langchain_anthropic import ChatAnthropic
from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware

agent = create_agent(
    model=ChatAnthropic(model="claude-sonnet-4-6"),
    tools=[...], system_prompt="<很长的稳定 system prompt>",
    middleware=[
        AnthropicPromptCachingMiddleware(ttl="5m"),                        # 复用
        ContextEditingMiddleware(edits=[ClearToolUsesEdit(trigger=100000, keep=3)]),  # 隔离
    ],
)
```

## 代码示例

用 Anthropic SDK 写一个"会算账"的多轮助手：稳定的长手册放进 system 并打显式断点，对话尾部用自动缓存，动态时间放进 user 消息，每轮先用 `count_tokens` 检查预算，再打印缓存命中情况。

```python
# pip install -U anthropic ;  export ANTHROPIC_API_KEY=...
import datetime
import anthropic

client = anthropic.Anthropic()
MODEL = "claude-sonnet-4-6"          # 该模型最小可缓存长度 1024 token
BUDGET = 150_000                     # 自己给应用定的输入预算
MANUAL = open("x7_manual.txt", encoding="utf-8").read()   # 任意一份 >1024 token 的长文本

SYSTEM = [                           # 稳定前缀：不要放时间戳等动态内容
    {"type": "text", "text": "你是 X7 车型售后助手，只依据 <manual> 回答，查不到就说不知道。"},
    {"type": "text", "text": f"<manual>\n{MANUAL}\n</manual>",
     "cache_control": {"type": "ephemeral"}},             # 显式断点 1：缓存 system + 手册
]

def ask(history: list, question: str) -> str:
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M")
    # 动态信息放进最新的 user 消息，位于断点之后，不破坏前缀
    history.append({"role": "user", "content": f"[当前时间 {now}]\n{question}"})

    n = client.messages.count_tokens(model=MODEL, system=SYSTEM, messages=history).input_tokens
    if n > BUDGET:                   # 超预算：这里简单丢最早一轮，生产中换成 compaction
        del history[:2]
    resp = client.messages.create(
        model=MODEL, max_tokens=1024, system=SYSTEM, messages=history,
        cache_control={"type": "ephemeral"},   # 自动缓存：断点随对话尾部后移（断点 2）
    )
    history.append({"role": "assistant", "content": resp.content})   # 只追加、不修改

    u = resp.usage
    print(f"预估 {n} | 写缓存 {u.cache_creation_input_tokens} | "
          f"读缓存 {u.cache_read_input_tokens} | 未缓存 {u.input_tokens}")
    return "".join(b.text for b in resp.content if b.type == "text")

history: list = []
for q in ["胎压报警阈值是多少？", "那标准胎压呢？", "冬天需要调高吗？"]:
    print(ask(history, q), "\n")
```

运行：准备一个超过 1024 token 的 `x7_manual.txt`，设置 `ANTHROPIC_API_KEY` 后 `python day13.py`。第 1 轮应看到"写缓存"较大、"读缓存"为 0；第 2、3 轮"读缓存"约等于手册加历史的长度。再把时间戳挪进 `SYSTEM` 第一块重跑，对比"读缓存"变成多少。

## 与 Android / 车载场景的联系

车机语音助手的车速、位置、电量每秒都在变，要放在断点之后的最新消息里，不能拼进 system；而语音交互往往间隔好几分钟，5 分钟 TTL 容易过期，高频车型手册这类固定前缀可以考虑 1 小时 TTL（写入价 2 倍，要算清楚是否划算）。

## 高频面试题

**1. 什么是上下文工程？和 prompt 工程有什么区别？**
- Prompt 工程关注"指令怎么写"，是一次性的；上下文工程关注"每次推理时窗口里放哪些 token"，是每轮都要做的迭代式筛选。
- 原因：context rot，上下文越长，注意力越分散、召回越差，窗口是有限的注意力预算。
- 目标：最少的高信号 token；手段：预算、缓存复用、清理/压缩/子 Agent 隔离、即时检索（Day 12）。

**2. Prompt caching 的原理是什么？哪些操作会让缓存失效？（追问：为什么 system prompt 里放当前时间会让成本暴涨？）**
- 缓存的是前缀，顺序是 tools → system → messages，到断点为止逐字节相同才命中；默认 TTL 5 分钟，命中刷新。
- 失效规律：改动越靠前，失效范围越大。改工具定义三层全失效；改 `tool_choice`、增删图片只让 messages 层失效；换模型全部失效。
- 追问：时间戳每次都不同，位于 system 层，导致 system 及之后的 messages 全部无法命中，每轮都按全价甚至 1.25 倍写入价计费。修法：system 保持不变，时间放进断点之后的 user 消息；用 cache diagnostics 会看到 `system_changed`。

**3. 计算题：system prompt + 工具共 2 万 token，5 分钟内调用 100 次，用 Sonnet 4.6（输入 $3/MTok），缓存能省多少？**
- 不缓存：100 × 2 万 = 200 万 token × $3 = **$6.00**。
- 缓存：1 次写入 2 万 × $3.75/MTok = $0.075；99 次读取 198 万 × $0.30/MTok = $0.594；合计约 **$0.67**，节省约 89%。
- 补充：首次写入比不缓存贵 25%，所以**只用一次的内容不要缓存**；调用间隔超过 TTL 时要比较 1 小时 TTL（写入 2 倍）是否划算。延迟也会下降，因为前缀不用重新处理。

**4. 工具结果清理、compaction、子 Agent 三者怎么选？**
- 工具结果清理：最轻量，适合工具输出大、用完即弃（检索、日志、文件内容），会使缓存失效，用 `clear_at_least` 保证值得。
- Compaction：长对话、长任务的首选，摘要旧轮次并在新窗口继续；风险是摘要丢细节，自定义 `instructions` 时要强调保留决策、未完成事项、关键数字。
- 子 Agent：子任务需要大量探索（搜索、读代码），用独立窗口，只回传精炼结果，从源头防止主上下文被污染；代价是 token 总量更高、协调更复杂（Day 16 详讲）。
- 实际生产会叠加使用：Claude Academy 的说法是生产级 Agent 通常同时用即时上下文、compaction、prompt caching 和 memory tool，分别解决成本、窗口大小和无状态问题。

**5. 场景题：车载语音助手一次行程可能聊 1 小时、调几十次导航/车控/知识库工具，你怎么设计它的上下文？**
- 分区：固定区（角色、安全规则、工具定义、常用手册）打断点缓存；动态车况放在每轮 user 消息里；历史区设上限。
- 隔离：导航路线、POI 列表这类大结果用完就清（工具结果清理，`exclude_tools` 保留"当前目的地"）；长行程开启 compaction，摘要里强制保留"当前目的地、用户偏好、未完成指令"。
- 工具：车控工具较多时按场景用 `tool_addition` / `tool_removal` 收放，避免改动 `tools` 数组导致缓存全部失效；高风险车控需确认。
- 成本与延迟：每轮用 `count_tokens` 或 `usage` 监控；交互间隔长就评估 1 小时 TTL；中途不要切换模型。
- 验证：用 cache diagnostics 和 `cache_read_input_tokens` 观察命中率，设置告警阈值。

**6. 追问：上下文窗口已经有 100 万 token 了，还需要做上下文工程吗？**
- 需要。窗口变大不会消除 context rot，无关 token 仍会稀释注意力；而且每个输入 token 都要计费、都会增加延迟。
- 大窗口改变的是"可以放进去什么"（比如小知识库可以全量放入 + 缓存，替代 RAG，见 Day 10），而不是"可以不加筛选"。
- Claude Academy 原话的意思："一百万 token 听起来很多，但做真实 Agent 时用得比你想的快"。

## 常见误区

1. **"窗口没满就随便放"**：context rot 让长上下文效果下降，成本和延迟也线性增长。判断标准是"这段 token 对这一轮有没有用"，不是"放不放得下"。
2. **把动态内容放在缓存前缀里**：system prompt 拼时间戳、用户名、请求 ID，或每次随机打乱工具顺序，缓存永远不会命中，而且每次都付 1.25 倍写入价。动态内容一律放到最后。
3. **清理/压缩时不考虑缓存和成对性**：频繁小幅清理会反复破坏缓存，反而更贵（所以要用 `clear_at_least`）；手动删历史时把 `tool_use` 留下却删了对应 `tool_result` 会直接报错（Day 9）；compaction 块必须原样回传。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带预算检查和缓存的一轮对话 `ask`**（20 行代码）

```python
def ask(history, question):
    # 取当前时间，作为本轮的动态信息
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M")
    # 动态信息拼进新的 user 消息，放在缓存前缀之后
    history.append({"role": "user", "content": f"[当前时间 {now}]\n{question}"})
    # 发请求前先量一下这次会用多少输入 token
    n = client.messages.count_tokens(model=MODEL, system=SYSTEM, messages=history).input_tokens
    # 超出预算就丢掉最早的一问一答
    if n > BUDGET:
        del history[:2]
    # 调模型：顶层 cache_control 开启自动缓存
    resp = client.messages.create(
        model=MODEL, max_tokens=1024, system=SYSTEM, messages=history,
        cache_control={"type": "ephemeral"},
    )
    # 把模型回复原样追加进历史，只追加不修改
    history.append({"role": "assistant", "content": resp.content})
    # 取出用量信息
    u = resp.usage
    # 打印写缓存、读缓存、未缓存三部分 token
    print(u.cache_creation_input_tokens, u.cache_read_input_tokens, u.input_tokens)
    # 只拼接文本块作为最终回答
    return "".join(b.text for b in resp.content if b.type == "text")
```

**三步走**
1. 看着上面的注释和代码，先手敲一遍全部注释，再手敲一遍代码（不复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `cache_control` 写成 `{"type": "cache"}` 或 `True`：唯一合法的 type 是 `"ephemeral"`，TTL 用 `"ttl": "1h"` 另外指定。
- 把时间戳写进 `SYSTEM`：代码能跑，但缓存每轮都失效，只有看 usage 才能发现，这是最隐蔽的错误。
- `count_tokens` 返回的是对象，要取 `.input_tokens`；它的参数要和 `create` 保持一致（同样的 `system`、`messages`），否则估算不准。

**昨日复习（只做第③步）**：Day 12 的 `agentic_rag`（带步数预算的 Agentic RAG 循环）。只看你自己写过的注释，重写代码，重点核对 `stop_reason != "tool_use"` 的退出条件，以及所有 `tool_result` 合并在一条 user 消息里送回。

## 动手练习（30 分钟）

在代码示例基础上做一个"缓存命中率实验"：① 正常跑 3 轮，记录每轮的读缓存/写缓存/未缓存 token；② 把时间戳移进 `SYSTEM[0]` 再跑 3 轮；③ 在第 2 轮前给请求加上 `tool_choice` 或调换两个工具的顺序（先随便加两个工具）再跑。把三组数字填进一张表，按 Sonnet 4.6 价格（输入 $3、写缓存 $3.75、读缓存 $0.30 / MTok）算出每组成本，写一句结论："把动态内容移出前缀后，3 轮成本从 $___ 降到 $___。"

## 延伸学习

- Claude Academy · Building with the Claude API：[Rules of prompt caching](https://academy.claude.com/courses/building-with-the-claude-api/rules-of-prompt-caching)、[Prompt caching in action](https://academy.claude.com/courses/building-with-the-claude-api/prompt-caching-in-action)（给 tools 和 system 加断点、观察 usage 字段）
- Claude Academy · Claude Platform 101 · [Context management](https://academy.claude.com/courses/claude-platform-101/context-management)：即时上下文、compaction、prompt caching、memory tool 四件套
- LangChain · [Context engineering in agents](https://docs.langchain.com/oss/python/langchain/context-engineering)：model / tool / life-cycle 三类上下文与 middleware

## 参考资料

- Anthropic Engineering · Effective context engineering for AI agents：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Claude Docs · Prompt caching：https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Claude Docs · Cache diagnostics：https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics
- Claude Docs · Mid-conversation system messages and tool changes：https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages
- Claude Docs · Context editing：https://platform.claude.com/docs/en/build-with-claude/context-editing
- Claude Docs · Compaction at a token threshold：https://platform.claude.com/docs/en/build-with-claude/compaction-threshold
- Claude Docs · Token counting：https://platform.claude.com/docs/en/build-with-claude/token-counting
- LangChain Docs · Context engineering in agents：https://docs.langchain.com/oss/python/langchain/context-engineering
- LangChain Docs · Prebuilt middleware：https://docs.langchain.com/oss/python/langchain/middleware/built-in
- LangChain Docs · Anthropic middleware integration：https://docs.langchain.com/oss/python/integrations/middleware/anthropic
- Claude Academy · Rules of prompt caching：https://academy.claude.com/courses/building-with-the-claude-api/rules-of-prompt-caching
- Claude Academy · Prompt caching in action：https://academy.claude.com/courses/building-with-the-claude-api/prompt-caching-in-action
- Claude Academy · Context management（Claude Platform 101）：https://academy.claude.com/courses/claude-platform-101/context-management
