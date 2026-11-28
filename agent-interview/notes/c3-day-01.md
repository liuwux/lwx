# Day 1 · LLM 基础（第三轮：stop_reason、token 账本与「不完整回答」）（2026-11-28）

## 今日 AI 动态（2026-11-28）

本期没有能写进来的动态：这次运行打不开各厂商的官方页面，网页抓取的权限请求无人确认，超时了。规则要求每条动态都用官方原文核实，所以没有收录任何条目。

二手新闻摘要里提到了几条 11 月 26–27 日的消息，都还没有官方确认：OpenAI 修正了 GPT-6.2 的计费错误、Qwen-Audio-3.1 扩展了方言能力。想看的话请自己去官网核对。

---

## 读前三问

1. 一次 Claude 回复结束时，`stop_reason` 有哪几类？哪几类代表「话没说完」，分别该怎么接着处理？
2. `max_tokens` 截断和「上下文窗口溢出」有什么区别？为什么处理方法完全不同？
3. 为什么 token 数要让 API 来数，不能自己用字数估？思考过程的 token 记在哪本账上？

## 从一个场景开始

你在做车载语音助手。用户问「回家路况怎么样」，模型回答：「前方 3 公里拥堵，建议走机场高速，预计」——然后 TTS 就停了。

日志里没有报错，HTTP 200，内容也是正常文本。问题出在你只取了 `content`，没有看这次回复**为什么结束**。其实那次的 `stop_reason` 是 `max_tokens`：你给的输出上限是 64，模型写到一半就被截断了。

还有一种情况：用户问了个越界的问题，模型拒答，返回空文本。你的代码照样把空字符串交给 TTS，车里一片安静，用户以为助手死机了。

第一轮讲了 token 和采样，第二轮讲了 prefill/decode 和 KV cache。这一轮往工程上落一步：**一次回复是怎么结束的，你的代码怎么分辨「说完了」「没说完」和「不肯说」。**

## 一句话总结

模型的每次回复都带一个结束原因，就像进程的退出码。生产代码要按它分流：说完了就播报，被截断就续写，被拒就走兜底话术，窗口满了就先压缩历史，不能只看有没有文本。

## 先跑起来

```python
# stop_guard.py —— 按 stop_reason 分流：截断续写、暂停续跑、拒答兜底（anthropic SDK 类型，剧本客户端离线可跑）
from anthropic.types import Message

def fake(text, stop, out=40, details=None):  # 造一条「服务端返回」，字段与真实 Message 一致
    return Message.model_validate({"id": "m", "type": "message", "role": "assistant", "model": "claude-sonnet-5-5",
        "content": [{"type": "text", "text": text}], "stop_reason": stop, "stop_sequence": None,
        "stop_details": details, "usage": {"input_tokens": 900, "output_tokens": out}})

SCRIPT = iter([fake("前方 3 公里拥堵，建议走机场高速，预计", "max_tokens", 64),
               fake("节省 12 分钟。是否切换路线？", "end_turn", 15),
               fake("", "refusal", 0, {"type": "refusal", "category": "cyber", "explanation": None})])

def create(messages, max_tokens):  # 真实代码里是 client.messages.create(model=..., max_tokens=..., messages=...)
    return next(SCRIPT)

def ask(messages, max_tokens=64, max_rounds=3):
    parts = []
    for _ in range(max_rounds):
        r = create(messages, max_tokens)
        text = "".join(b.text for b in r.content if b.type == "text")
        print(f"  stop_reason={r.stop_reason:<10} out={r.usage.output_tokens:<3} 片段={text!r}")
        if r.stop_reason in ("max_tokens", "pause_turn"):  # 没说完：把半截回复原样放回，让模型接着写
            parts.append(text)
            messages = messages + [{"role": "assistant", "content": r.content}]
            if r.stop_reason == "max_tokens":  # 截断还要补一句「继续」；pause_turn 原样回传即可
                messages.append({"role": "user", "content": "接着上文继续，不要重复"})
            continue
        if r.stop_reason == "refusal":  # 拒答：不播半句话，记录类别后走兜底话术
            print(f"  拒答类别={r.stop_details.category}")
            return "这个问题我帮不了，可以换个说法。"
        if r.stop_reason == "model_context_window_exceeded":  # 窗口满了：续写也没用，要先压缩历史
            return "[需要先压缩对话历史]"
        return "".join(parts) + text  # end_turn / stop_sequence：完整结束
    return "".join(parts) + "……（已达续写上限）"

print("播报1:", ask([{"role": "user", "content": "帮我看下回家路况"}]))
print("播报2:", ask([{"role": "user", "content": "教我破解隔壁车的蓝牙钥匙"}]))
```

运行：`pip install -U anthropic && python stop_guard.py`

真实输出（2026-11-28，anthropic 2.14.0，剧本客户端，不调用真实 API）：

```text
  stop_reason=max_tokens out=64  片段='前方 3 公里拥堵，建议走机场高速，预计'
  stop_reason=end_turn   out=15  片段='节省 12 分钟。是否切换路线？'
播报1: 前方 3 公里拥堵，建议走机场高速，预计节省 12 分钟。是否切换路线？
  stop_reason=refusal    out=0   片段=''
  拒答类别=cyber
播报2: 这个问题我帮不了，可以换个说法。
```

该看的地方：

- 第一行的 `out=64` 正好等于 `max_tokens=64`。截断时输出 token 用满了上限，这是排查「回答为什么只有半句」最快的线索。
- 续写之后拼起来是一句完整的话。TTS 应该等 `end_turn` 之后再播，不要每拿到一段就播。
- 拒答那次 HTTP 是成功的，文本是空的，只有 `stop_reason` 和 `stop_details.category` 告诉你发生了什么。

## 核心概念

### 1. stop_reason：每次回复的「退出码」

- **是什么**：每条回复都会说明它为什么停下。非流式请求里这个字段一定有值。流式请求里，它在 `message_start` 事件中是 null，后面的事件里才有值。
- **没有它会怎样**：你只能靠「有没有文本」判断成功与否。半句话、空回复和完整回答在 HTTP 层看起来一模一样。
- **对应你熟悉的**：`Process.exitValue()`，或者 `onActivityResult` 的 resultCode。拿到数据之前，先看是怎么结束的。

按处理方式可以分成四组：**说完了**（`end_turn`，以及命中你设的停止词时的 `stop_sequence`）；**没说完**（`max_tokens`、`pause_turn`）；**要你干活**（`tool_use`，要先执行工具再回传结果）；**不能继续**（`refusal`、`model_context_window_exceeded`）。

### 2. 两种「满了」：输出上限和上下文窗口

- **是什么**：`max_tokens` 是你给**这一次输出**定的上限。上下文窗口是模型**输入加输出总共**能装多少。对应的结束原因分别是 `max_tokens` 和 `model_context_window_exceeded`。
- **没有它会怎样**：把两种情况混为一谈，你就会对窗口溢出也去续写。续写会把历史再塞长一点，只会更快撞墙。
- **对应你熟悉的**：前者像单次 `read()` 的 buffer 大小，读不完就再读一次。后者像进程内存上限，再读也没用，得先释放（压缩或截断历史）。

续写的做法：把截断的 assistant 内容原样放回 `messages`，再追加一句「继续」。`pause_turn` 不一样，它是服务端把一个很长的回合暂停了，SDK 文档的说法是把这条回复**原样**放回下一次请求，模型就会接着跑，不需要补 user 消息。

### 3. token 账本：让 API 数，别自己估

- **是什么**：每次回复的 `usage` 会列出 `input_tokens`、`output_tokens`，以及缓存读写用掉的 token。发请求之前想知道有多大，可以调用 `client.messages.count_tokens(...)`，它返回 `input_tokens`，把 system、tools 和图片都算进去了。
- **没有它会怎样**：按「一个汉字约等于几个 token」来估，换个模型分词器就不准了。工具定义和 system prompt 也很容易漏算，于是预算和截断判断全部偏掉。
- **对应你熟悉的**：APK 大小以打包后的产物为准，不按源码行数估。

还有一个坑：思考过程也算输出。`usage.output_tokens_details.thinking_tokens` 记的是推理用掉的 token，它总是小于等于 `output_tokens`。所以开了思考之后，同样的 `max_tokens` 留给正文的空间变小了，更容易被截断。SDK 还提醒：`usage` 和你看到的文本并不是一一对应的，因为服务端有格式转换和解析这一步。

### 4. 三种「不完整」要分开处理：截断、拒答、幻觉

- **是什么**：截断是**没写完**，拒答是**不肯写**，这两种都有 `stop_reason` 标记。幻觉不一样：模型把话写完了（`end_turn`），但内容是错的，而且**没有任何字段会告诉你**。
- **没有它会怎样**：有人以为 `end_turn` 就代表答对了。其实 `end_turn` 只说明模型自己决定停下，不保证内容正确。
- **对应你熟悉的**：编译通过不等于测试通过。

幻觉只能靠外部手段压：允许模型说「不知道」；要求它先引用原文再作答；需要事实的地方走工具或 RAG，再拿外部数据做校验（比如路况数字必须来自导航 API）。

## 高频面试题

**Q1：线上发现语音助手经常只播半句话，你怎么排查？**

- 先看日志里的 `stop_reason`。如果是 `max_tokens`，再看 `output_tokens` 是不是正好等于上限。
- 开了思考的话，看 `thinking_tokens` 占了多少，思考挤占了正文的空间。
- 处理办法：调大上限，加上续写循环，并且等 `end_turn` 之后再交给 TTS。

**Q2（追问）：续写循环会不会死循环？还有什么副作用？**

- 一定要设最大轮数，代码里的 `max_rounds`。达到上限就明确告诉用户没说完。
- 每续写一次，就要把之前的全部内容再作为输入发一遍，输入成本和延迟都会上升。
- 续写可能出现重复或衔接不上，提示里要写「不要重复」，必要时在拼接处去重。

**Q3：`stop_reason` 是 `end_turn`，能说明回答正确吗？**

- 不能。它只说明模型主动结束了，幻觉的回答通常也是 `end_turn`。
- 正确性只能靠外部校验：工具返回的事实、结构化输出校验、评测集。
- 可以允许模型说「不知道」，并要求它给出引用，降低编造的概率。

**Q4（场景题）：车载多轮对话跑了半小时，突然返回 `model_context_window_exceeded`，怎么设计？**

- 这时候不能续写，要先把旧的对话轮次摘要压缩，或者只保留最近 N 轮加关键状态。
- 预防优先：每次请求前用 `count_tokens` 或上一轮的 `usage` 估算，快到阈值就主动压缩。
- 给用户一个没有感知的降级：本轮先回一句简短的话，后台完成压缩后再继续。

## 常见误区

1. **「HTTP 200 加上有文本就是成功」**：截断的回复和拒答的回复都是 200。成功的标准要包括 `stop_reason`。
2. **「`max_tokens` 调到最大就一劳永逸」**：这会让最坏情况下的延迟和成本跟着变大，而且挡不住上下文窗口溢出。上限要按场景设，比如语音播报本来就应该短。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：按 stop_reason 分流的 ask 函数**（取自 `stop_guard.py`）

```python
# 定义问答函数：消息列表、单次输出上限、最多续写几轮
def ask(messages, max_tokens=64, max_rounds=3):
    # 收集被截断的半截文本
    parts = []
    # 最多请求 max_rounds 次，防止无限续写
    for _ in range(max_rounds):
        # 发一次请求
        r = create(messages, max_tokens)
        # 把所有文本块拼成一段字符串
        text = "".join(b.text for b in r.content if b.type == "text")
        # 没说完：截断或服务端暂停
        if r.stop_reason in ("max_tokens", "pause_turn"):
            # 先存下这一段
            parts.append(text)
            # 把这次的回复原样作为 assistant 消息放回历史
            messages = messages + [{"role": "assistant", "content": r.content}]
            # 只有截断才需要补一句「继续」
            if r.stop_reason == "max_tokens":
                messages.append({"role": "user", "content": "接着上文继续，不要重复"})
            # 进入下一轮请求
            continue
        # 拒答：返回兜底话术，不播空文本
        if r.stop_reason == "refusal":
            return "这个问题我帮不了，可以换个说法。"
        # 窗口满了：续写没用，交给上层先压缩
        if r.stop_reason == "model_context_window_exceeded":
            return "[需要先压缩对话历史]"
        # 正常结束：拼上之前的片段一起返回
        return "".join(parts) + text
    # 轮数用完还没结束，明确标注
    return "".join(parts) + "……（已达续写上限）"
```

**三步走**

1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**

- 放回历史的是 `r.content`（内容块列表），不是拼好的 `text`。如果里面有工具调用块或思考块，用纯文本放回去会丢信息。
- `pause_turn` 不要补 user 消息，原样回传就行。只有 `max_tokens` 才需要「继续」。
- 最后要 `"".join(parts) + text`。只返回 `text` 的话，前面截断的那几段就丢了。

**昨日复习（只做第③步）**：第二轮 Day 30 的「禁止改测试的工具拦截 + 搜索预算」。只看你自己写的注释重写，重点核对：`@wrap_tool_call` 的参数顺序 `(request, handler)`、`return handler(request)`、`ToolCallLimitMiddleware(tool_name="search", run_limit=2)`。

## 动手练习（30 分钟）

在 `stop_guard.py` 的基础上做三件事：

1. 在 `SCRIPT` 里加一条 `model_context_window_exceeded`，再写一个 `compress(messages)`：只保留第一条和最后两条消息。然后让 `ask` 在压缩之后**重试一次**，不要直接返回。
2. 给 `fake` 加一个 `thinking` 参数，写进 `usage.output_tokens_details.thinking_tokens`。打印「正文可用 token = output_tokens − thinking_tokens」。
3. 加一个 `stop_sequence` 场景：模型输出到「是否切换路线？」就停下，`stop_sequence="？"`。确认代码把它当作正常结束处理。

## 读后自测

1. `stop_reason` 有哪几类？哪几类代表「话没说完」，分别该怎么接着处理？——见「核心概念 1、2」
2. `max_tokens` 截断和上下文窗口溢出有什么区别？——见「核心概念 2」
3. 为什么 token 要让 API 来数？思考过程的 token 记在哪本账上？——见「核心概念 3」

## AI 历史知识点：Nucleus Sampling / top-p 采样（2019，华盛顿大学与艾伦人工智能研究所）

**是什么**：论文《The Curious Case of Neural Text Degeneration》提出了 top-p 采样。每一步只在「累计概率达到 p 的最小候选集合」里随机挑一个 token，概率极低的长尾词直接排除。

**何时由谁提出**：2019 年由 Ari Holtzman、Jan Buys、Yejin Choi 等人提出，作者来自华盛顿大学和艾伦人工智能研究所（AI2），论文发表于 ICLR 2020。

**为什么重要**：论文指出两个问题。贪心解码和 beam search 这类「最大化概率」的方法会产出重复、空洞的文字。纯随机采样又会抽到长尾的怪词，越写越离谱。top-p 让候选集合的大小随模型的把握程度变化：模型很确定时只剩几个候选，不确定时多留一些。这个思路后来成了几乎所有 LLM API 的默认参数之一。

**和今天的联系**：今天的 API 仍然暴露 `temperature` 和 `top_p`，但官方普遍建议一般只调其中一个。推理模型兴起之后，开了思考时部分采样参数受限或被固定。采样从「调参技巧」变成了「平台替你管」的东西，这和今天讲的结束原因是同一个趋势：开发者更需要看懂模型行为的信号，而不是自己硬调。

来源：arXiv 1904.09751（https://arxiv.org/abs/1904.09751）。本次未能打开原文核实，属于非 LangChain/Claude 来源。

## 参考区（用时查，不必通读）

### 细节速查

| 项 | 内容（anthropic Python SDK 2.14.0） |
|---|---|
| `stop_reason` 全集 | `end_turn`、`max_tokens`、`stop_sequence`、`tool_use`、`pause_turn`、`refusal`、`model_context_window_exceeded` |
| `stop_sequence` | 命中自定义停止词时为该字符串，否则为 null |
| `stop_details` | 仅在有附加信息时非 null；拒答时 `type="refusal"`，`category` 为 `cyber`/`bio`/`frontier_llm`/`reasoning_extraction`/`general_harms` 或 null；`explanation` 文本不保证稳定 |
| `usage` 主要字段 | `input_tokens`、`output_tokens`、`cache_creation_input_tokens`、`cache_read_input_tokens`、`output_tokens_details.thinking_tokens`、`service_tier`（`standard`/`priority`/`batch`） |
| 计数接口 | `client.messages.count_tokens(model=..., messages=..., system=..., tools=...)` → `MessageTokensCount.input_tokens`（含 system、tools） |
| 流式 | `message_start` 中 `stop_reason` 为 null，之后的事件里非 null |

### 来源说明

- 本次运行网页抓取的权限请求无人确认，超时了，platform.claude.com 的文档页一个都没打开。面试部分的字段和含义以本机安装的 anthropic 2.14.0 源码（`types/stop_reason.py`、`types/message.py`、`types/refusal_stop_details.py`、`types/usage.py`、`types/output_tokens_details.py`、`resources/messages/messages.py` 中的 docstring）为准。核心概念 4 中压制幻觉的手段沿用前两轮已核实的内容，建议对照下面的官方指南原文。
- 「今日 AI 动态」的说明见该节。

### 延伸学习

- Claude Academy「Building with the Claude API」课程中关于请求参数、回复结构和 stop reason 的章节（本次未能打开，请自行在 academy.claude.com 查找）。

### 参考资料

- 以下页面本次没有打开，是 SDK docstring 里引用的官方链接，供自行查阅：
  - Token counting 指南：https://platform.claude.com/docs/en/build-with-claude/token-counting
  - Adaptive thinking：https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking
