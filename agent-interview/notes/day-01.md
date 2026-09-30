# Day 01 · LLM 基础（2026-09-30）

## 一句话总结

LLM 本质是一个「给定前文、预测下一个 token 概率分布」的自回归模型：输入按 token 计费和计长，所有内容（system、历史消息、工具定义、工具结果）都要塞进有限的上下文窗口；采样参数决定从分布里怎么挑词；幻觉来自「它在生成最像的续写，而不是查证事实」，所以工程上要靠 grounding、引用和允许说「不知道」来压制。

## 核心概念

### 1. Transformer 直觉（面试够用版）

```
文本 ──tokenizer──> token ids ──embedding──> 向量序列
                                         │
                   ┌─────────────────────▼─────────────────────┐
                   │  N 层 [ Self-Attention → FeedForward ]     │  每个位置"看"前面所有位置，
                   │  (因果 mask：只能看左边，不能偷看未来)       │  按相关性加权汇总信息
                   └─────────────────────┬─────────────────────┘
                                         ▼
                     最后一个位置 → 词表上的概率分布 P(next token)
                                         ▼
                      采样（temperature / top_p / top_k）→ 新 token
                                         ▼
                      拼回输入，循环，直到 end_turn / max_tokens / stop_sequence
```

- **Self-Attention**：每个 token 对前文所有 token 算相关性权重再加权求和，所以能处理长距离依赖；代价是计算量随序列长度大约平方增长——这就是长上下文贵、慢的直觉来源。
- **自回归**：一次只生成一个 token，生成的 token 再作为输入。所以**输出 token 比输入 token 慢且贵**，也解释了"流式输出"为什么天然存在。
- **训练三阶段**（Claude 官方术语表）：预训练（在大规模无标注语料上预测下一个词，此时"并不擅长回答问题或遵循指令"）→ 指令微调 → RLHF（人类对多个回答排序，强化学习让模型偏向高分回答）。

### 2. Token

- 官方定义：token 是模型最小处理单元，可对应词、子词、字符甚至字节。**对 Claude，1 个 token 约等于 3.5 个英文字符**，不同语言差异较大（中文通常更"费" token）。
- 为什么重要：计费、上下文上限、延迟都按 token 算。
- 精确计数用官方 `count_tokens` 接口：**免费**，但结果是**估计值**，受 RPM 限流。

### 3. 上下文窗口（Context Window）

- 官方定义：模型生成回复时能参考的全部文本，**包括回复本身**，是模型的"工作记忆"，不是训练语料。
- **什么都算进去**：system prompt、`messages` 里的每条消息（含工具结果、图片、文档），以及**工具定义**。
- **多轮线性增长**：每轮的用户消息和助手回复都会完整保留、累积到下一轮输入里 → 这就是 Agent 跑久了会爆上下文的根源。
- **超限行为**：输入本身超长 → `400 invalid_request_error`（prompt is too long）；在 Claude 4.5+ 上，输入 + `max_tokens` 超过窗口时请求仍会被接受，生成撞到上限会以 `stop_reason: "model_context_window_exceeded"` 停止。
- **Context rot**：上下文越长，准确率和召回会下降。所以"放什么进上下文"和"窗口有多大"同样重要——这是后面第 13 天「上下文工程」的伏笔。
- 当前规格（官方模型总览，2026-09）：Opus 5.5 / Sonnet 5.5 等为 1M token 窗口、最大输出 128K；Haiku 4.5 为 200K / 64K。
- 扩展思考（extended thinking）：Sonnet 4.5 及更早的模型会自动剥离历史轮次的 thinking 块；Opus 4.5+、Sonnet 4.6+ 默认保留，并计入上下文。

### 4. 采样参数：temperature / top_p / top_k

| 参数 | 含义 | 经典用法 |
|---|---|---|
| temperature | 调节分布"尖锐度"，低 → 保守确定，高 → 多样有创意 | 分析/选择题接近 0，创作接近 1 |
| top_p（nucleus） | 按概率从高到低累加，到 p 截断，只在这个集合里采样 | 高级场景才调 |
| top_k | 只从概率最高的 K 个里采样，砍掉长尾 | 高级场景才调 |

**⚠️ 2026 年的重要变化（面试加分点）**：Claude API 文档已把这三个参数标为 **Deprecated**——**Opus 4.6 之后发布的模型不再支持设置它们**：temperature 只接受 1.0，top_p 只接受 ≥0.99，top_k 任何值都会被 400 拒绝。另外，官方明确说明**即使 temperature=0，结果也不是完全确定的**。

面试时的正确说法：「采样参数是经典的输出控制手段，但新一代模型把它收回了；想要稳定输出，应当靠 prompt 约束、结构化输出、few-shot 和评测，而不是指望 temperature=0。」

### 5. 幻觉成因与对策

**成因**（直觉层面）：
1. 训练目标是"生成最像的续写"，不是"说真话"——模型对不熟悉的事实也能流畅编出看似合理的答案。
2. 知识有截止日期，且参数里的知识是有损压缩。
3. 上下文太长或有噪声（context rot）、提示词含糊，都会放大幻觉。

**Claude 官方给出的对策**：
- 基础：**允许说"我不知道"**；长文档（>20k token）先**逐字摘引原文再回答**；要求**每条结论附引用**，找不到引用就撤回。
- 进阶：思维链验证、**Best-of-N 对比**（多次生成看是否一致）、迭代复核、**限制只用给定资料**。
- 官方原话的要点：这些方法能显著降低但**不能消除**幻觉，关键信息必须校验。

## 代码示例

用 Anthropic SDK 先数 token，再调用并查看 `usage` 和 `stop_reason`；最后用 LangChain 做同样的调用。

```python
# pip install -U anthropic "langchain[anthropic]"
# export ANTHROPIC_API_KEY=sk-...
import anthropic
from langchain.chat_models import init_chat_model
from langchain_core.callbacks import UsageMetadataCallbackHandler

MODEL = "claude-sonnet-5-5"
SYSTEM = "你是车载语音助手。不确定时直接说「我不知道」，不要编造。"
messages = [{"role": "user", "content": "用一句话解释什么是上下文窗口。"}]

client = anthropic.Anthropic()

# 1) 调用前估算输入 token（免费，但结果是估计值）
count = client.messages.count_tokens(model=MODEL, system=SYSTEM, messages=messages)
print("预估输入 tokens:", count.input_tokens)

# 2) 真正调用。注意：新模型不再支持自定义 temperature/top_p/top_k，这里不传
resp = client.messages.create(
    model=MODEL,
    max_tokens=200,          # 输出上限，可能提前停止
    system=SYSTEM,
    messages=messages,
)
print(resp.content[0].text)
print("stop_reason:", resp.stop_reason)   # end_turn / max_tokens / stop_sequence ...
print("usage:", resp.usage.input_tokens, "in /", resp.usage.output_tokens, "out")

# 3) 多轮：把回复追加回 messages —— 上下文就是这样线性增长的
messages += [
    {"role": "assistant", "content": resp.content[0].text},
    {"role": "user", "content": "再举一个 Android 开发者能理解的类比。"},
]
print("第二轮预估输入 tokens:",
      client.messages.count_tokens(model=MODEL, system=SYSTEM, messages=messages).input_tokens)

# 4) LangChain 写法：统一接口 + 用量回调
model = init_chat_model(MODEL, max_tokens=200, timeout=30, max_retries=6)
cb = UsageMetadataCallbackHandler()
out = model.invoke("token 和字符是什么关系？一句话。", config={"callbacks": [cb]})
print(out.content)
print(cb.usage_metadata)
```

运行说明：安装依赖、设置 `ANTHROPIC_API_KEY` 后 `python day01.py`；观察第二轮 token 数明显大于第一轮。模型 ID 以你账号可用的官方模型列表为准。

## 与 Android / 车载场景的联系

- 上下文窗口可以类比成 **App 进程内存**：历史消息像不断 add 的对象，不做回收（摘要/裁剪）早晚 OOM（400 prompt too long）。
- 车载语音对延迟敏感：输出 token 是逐个生成的，所以要**短回答 + 流式播报（TTS 边收边播）**。

## 高频面试题

**Q1. 什么是 token？为什么 Agent 开发者要关心 token？**
- 模型处理文本的最小单位（词/子词/字符/字节），Claude 约 3.5 英文字符/token，中文更费。
- 决定三件事：成本、上下文上限、延迟（输出逐 token 生成）。
- 工程手段：调用前 `count_tokens` 估算、监控 `usage`、控制工具返回体积。

**Q2. 上下文窗口里都装了什么？多轮 Agent 为什么容易爆？**
- system prompt + 全部历史消息 + 工具定义 + 工具结果 + 图片文档 + 本次输出。
- 每轮完整累积，线性增长；Agent 每次工具调用都会追加结果，增长更快。
- 应对：摘要压缩、裁剪旧工具结果、子 Agent 隔离上下文、prompt caching 降成本。

**Q3（追问）. 既然窗口已经 1M token，是不是把所有资料都塞进去就行？**
- 不行。官方明确提到 **context rot**：上下文越长，准确率和召回下降。
- 成本和首 token 延迟也随输入增长。
- 正确做法：检索只放相关内容（RAG）、按需加载、把关键指令放在清晰位置、做评测验证。

**Q4. temperature、top_p、top_k 分别是什么？temperature=0 就确定吗？**
- temperature 调节分布尖锐度；top_p 按累计概率截断；top_k 只取前 K 个。
- 官方明确：temperature=0 也**不完全确定**（浮点并行、批处理等因素）。
- 加分点：Claude 新模型（Opus 4.6 之后发布的）已不支持自定义这三个参数，稳定性要靠 prompt、结构化输出和评测。

**Q5. 幻觉是怎么产生的？你在项目里怎么降低？**
- 成因：目标是生成"像"的文本而非"真"的文本；知识截止与有损记忆；上下文噪声。
- 对策：允许说不知道、先摘引再回答、要求引用并可撤回、限定只用给定资料、Best-of-N 一致性检查、关键信息外部校验。
- 架构层：RAG/工具查实时数据，让模型"查"而不是"背"。

**Q6（场景）. 你做一个车载语音助手，用户问"附近哪家加油站最便宜"，模型回答了一个根本不存在的加油站。怎么排查和修复？**
- 排查：看 trace——是没调用 POI/价格工具就直接回答，还是工具返回为空后模型"补全"了？
- 修复：system prompt 规定"价格和位置必须来自工具结果，没有就说查不到"；工具返回空时给明确的结构化结果（如 `{"results": [], "reason": "no_data"}`）；输出做校验（回答中的站点 ID 必须出现在工具结果里）。
- 兜底：建评测集覆盖"无结果""网络失败"场景，做回归测试。

## 常见误区

1. **"temperature=0 就是确定性输出"**：官方明确不是；而且新模型已不允许改 temperature。
2. **"上下文越大越好"**：context rot + 成本 + 延迟，精挑细选比堆量重要。
3. **"只有对话消息占上下文"**：工具定义和工具返回常常才是大头，Agent 场景尤其如此。

## 动手练习（30 分钟）

写一个脚本，模拟 10 轮对话（每轮把助手回复追加回 `messages`），每轮用 `count_tokens` 记录输入 token 数并打印成表格；然后实现一个简单策略：当 token 数超过某阈值时，把最早的若干轮替换成一段由模型生成的摘要，对比压缩前后的 token 数和回答质量。

## 参考资料

- Claude 官方术语表（token、上下文窗口、temperature、预训练、RLHF）：https://platform.claude.com/docs/en/about-claude/glossary
- Context windows：https://platform.claude.com/docs/en/build-with-claude/context-windows
- Create a Message（temperature/top_p/top_k 弃用说明、stop_sequences、usage）：https://platform.claude.com/docs/en/api/messages/create
- Token counting：https://platform.claude.com/docs/en/build-with-claude/token-counting
- Reduce hallucinations：https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations
- Models overview：https://platform.claude.com/docs/en/about-claude/models/overview
- LangChain Models（init_chat_model、标准参数、UsageMetadataCallbackHandler）：https://docs.langchain.com/oss/python/langchain/models
- Introducing Claude 2.1（诚实性与长上下文的早期数据）：https://www.anthropic.com/news/claude-2-1
