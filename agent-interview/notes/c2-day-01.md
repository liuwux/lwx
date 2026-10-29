# Day 1 · LLM 基础（第二轮：推理过程、上下文代价与采样边界）（2026-10-29）

## 今日 AI 动态（2026-10-29）

### 趋势观察

过去 24 小时最重要的官方发布是 Google 的 Gemini 4 Carbon。它的卖点不是聊天，而是连续几天跑的长任务 Agent 和网络安全能力：找漏洞、验证漏洞、打补丁。这款模型先只开放给网络安全防御方和受信任的合作伙伴，没有直接上 API。推测：攻防能力很强的前沿模型，「先给防御方、再逐步开放」可能会成为常见的发布方式。

### 今日要点

- Google 发布 Gemini 4 Carbon，定位是长任务 Agent 和网络安全，先开放给网络安全防御方。
- Meta 和美国五所大学合作，提供自家模型、研究资助和 200 万美元资金，做医学和基础科学研究。

### 分公司动态

#### 1. Google

**[Gemini 4 Carbon: our next frontier model](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-carbon/)**（2026-10-28）

Gemini 4 Carbon 是 Gemini 4 Argon 发布两周后推出的新旗舰。Google 说内部团队用它连续跑过好几天的工程任务，比如在大型代码库里定位并修复 bug。和 Argon 相比，它生成安全代码的成功率高出一倍多，没有依据的说法少了大约一半；在 Gray Swan 间接提示注入测试中攻击成功率是 0%。首发阶段只给一部分网络安全防御方和受信任的合作伙伴用，面向开发者和普通用户的开放时间还没公布。首发价和 Argon 相同：每百万 token 输入 2 美元、输出 10 美元。

**为什么重要**：「能连续跑几天的 Agent」和「抗提示注入」已经被厂商当作旗舰模型的主要卖点，这两点正是 Agent 面试的高频考点。

#### 2. Meta

**[Meta Partners With Five US Universities to Accelerate Medical and Scientific Breakthroughs](https://about.fb.com/news/2026/10/meta-partners-with-five-us-universities-to-accelerate-medical-and-scientific-breakthroughs/)**（2026-10-28）

Meta 和加州理工、俄亥俄州立大学医学院等五所大学合作，提供 Meta 的模型、研究资助和研究员协作，另有 200 万美元资金。文中举了三个医学项目：用开放权重的 SAM 模型做婴儿气管切开护理监测，用多模态 AI 帮助和痴呆老人对话，用 CT、病历和心电图预测心衰风险。

**为什么重要**：开放权重模型正在进入医院床旁设备这类对可靠性要求很高的场景。

### 动态来源

- [Gemini 4 Carbon: our next frontier model](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-carbon/)
- [Meta Partners With Five US Universities to Accelerate Medical and Scientific Breakthroughs](https://about.fb.com/news/2026/10/meta-partners-with-five-us-universities-to-accelerate-medical-and-scientific-breakthroughs/)

---

> 第二轮说明：第一轮 Day 1 已经讲过 Transformer 直觉、token、上下文窗口、temperature/top-p 和幻觉。今天不再重复这些定义，改讲面试官的追问：一次请求在模型内部分成哪两个阶段，长上下文到底贵在哪，以及采样参数在新版 API 上遇到的边界情况。

## 读前三问

1. 为什么「首个 token 出来的时间」和「后面每秒出几个 token」是两个独立指标？各自主要受什么影响？
2. 上下文窗口有 1M，为什么还不能把资料全塞进去？
3. `temperature=0` 为什么不能保证每次输出一样？在不允许设采样参数的新模型上，怎么让输出稳定？

## 从一个场景开始

你在给车机做「行程助手」：每次请求都把 3 万 token 的车辆手册和最近行程塞进上下文，再让模型回答。

上线后接连出了三件事：

- 用户说完话要等 4 秒才看到第一个字，但字一旦开始出，就出得很快。
- 有些回答说到一半就断了，日志里 `stop_reason` 是 `max_tokens`，可是看得见的回答明明只有两百字。
- 为了让回答稳定，你设了 `temperature=0`。升级 SDK 后所有请求都返回 400；而在升级前，同一个问题也偶尔会给出不同答案。

这三件事都要从「模型生成一个回答时内部发生了什么」来解释。

## 一句话总结

模型先一次性读完整个提示词，再一个 token 一个 token 地往外写；读得越长，首字越慢、显存越贵，效果也会下降。输出上限包括思考部分，而要让输出稳定，应该靠提示词和结构约束，不能靠采样参数。

## 先跑起来

用 numpy 写一个单头注意力的玩具模型，对比有 KV cache 和没有 KV cache 时的计算量，再估算真实规模下 KV cache 占多少显存。不需要 API Key。

```python
# kv_cache_demo.py —— 单头注意力玩具模型：看清 prefill / decode、KV cache 和浮点不确定性
import numpy as np
rng = np.random.default_rng(0)
d = 64                                            # 隐藏维度（真实模型是几千）
Wq, Wk, Wv = (rng.standard_normal((d, d)) / d**0.5 for _ in range(3))

def attend(q, K, V):                              # 一个 query 对所有历史 K/V 做注意力
    s = K @ q / np.sqrt(d)
    w = np.exp(s - s.max()); w /= w.sum()
    return w @ V

def generate(prompt, new, use_cache):
    xs, outs, kv_rows = prompt, [], 0             # kv_rows：一共算了多少行 K/V
    if use_cache:
        K, V = prompt @ Wk, prompt @ Wv           # prefill：提示词的 K/V 一次算完（可并行）
        kv_rows += len(prompt)
    for x in new:                                 # decode：一次只能出一个 token
        xs = np.vstack([xs, x])
        if use_cache:                             # 有缓存：只算新 token 的 k/v，追加进去
            K, V = np.vstack([K, x @ Wk]), np.vstack([V, x @ Wv]); kv_rows += 1
        else:                                     # 没缓存：整段历史重算一遍
            K, V = xs @ Wk, xs @ Wv; kv_rows += len(xs)
        outs.append(attend(x @ Wq, K, V))
    return np.array(outs), kv_rows

prompt, new = rng.standard_normal((2000, d)), rng.standard_normal((200, d))
a, rows_a = generate(prompt, new, use_cache=False)
b, rows_b = generate(prompt, new, use_cache=True)
print(f"结果一致: {np.allclose(a, b)} | 无缓存算 K/V 行数: {rows_a:,} | 有缓存: {rows_b:,}")

def kv_gb(layers, kv_heads, head_dim, seq, bytes_=2):   # 每条请求的 KV cache 显存
    return 2 * layers * kv_heads * head_dim * seq * bytes_ / 1e9
print(f"32 层/32 个 KV 头(MHA)/128K 上下文: {kv_gb(32, 32, 128, 131072):.1f} GB")
print(f"32 层/ 8 个 KV 头(GQA)/128K 上下文: {kv_gb(32, 8, 128, 131072):.1f} GB")

x = rng.standard_normal(100_000).astype(np.float32)     # 同一批数，换一种加法顺序
s1, s2 = x.sum(), np.cumsum(x)[-1]                      # 分块两两相加 vs 从头逐个相加
print(f"float32 两种求和顺序结果相同吗: {s1 == s2}（{s1} vs {s2}）")
```

**运行**：`pip install numpy && python kv_cache_demo.py`

**实测输出**（2026-10-29，Python 3.13，numpy 2.5.3）：

```
结果一致: True | 无缓存算 K/V 行数: 420,100 | 有缓存: 2,200
32 层/32 个 KV 头(MHA)/128K 上下文: 68.7 GB
32 层/ 8 个 KV 头(GQA)/128K 上下文: 17.2 GB
float32 两种求和顺序结果相同吗: False（127.1640625 vs 127.16365814208984）
```

**该看什么**：
- 结果完全一致，但计算量差了约 190 倍。KV cache 只是省掉重复计算，不改变结果。
- 显存估算是按一条 128K 的请求算的，几十 GB 只属于一个用户。把 KV 头从 32 个减到 8 个（GQA），显存直接降到四分之一。
- 同样的数字换个加法顺序，结果就差了一点点。GPU 上并行计算的顺序会随批次和硬件变化，这就是 `temperature=0` 也不能保证输出完全一样的原因之一。

## 核心概念

### 1. Prefill 与 Decode：一次请求的两个阶段

- **是什么**：Prefill 阶段把整个提示词一次性读进去，所有 token 可以并行计算；Decode 阶段每次只生成一个 token，生成完再接着算下一个，只能串行。
- **没有它会怎样**：分不清这两个阶段，就看不懂延迟。首字慢通常是提示词太长（prefill 慢），整段回答慢通常是输出太长（decode 慢）。
- **对应你熟悉的**：App 冷启动（一次性加载、初始化）和之后每一帧的渲染，是两个不同的性能指标。

Claude 文档把前者叫 TTFT（首 token 时间），流式输出时它直接决定用户觉得「快不快」。优化手段也分成两类：缩短输入、复用前缀缓存，改善的是 prefill；限制输出长度、换更快的模型、降低 effort，改善的是 decode。

### 2. KV cache：用显存换算力

- **是什么**：生成每个新 token 时，都要和前面所有 token 的 Key/Value 做注意力。KV cache 把算过的 K/V 存起来，下一步直接复用。
- **没有它会怎样**：如上面的输出所示，每一步都要把整段历史重算一遍，计算量随长度平方增长。
- **对应你熟悉的**：RecyclerView 的 ViewHolder 复用：不重复 inflate，代价是要多占一些内存。

代价是显存：KV cache 的大小和上下文长度、层数、KV 头数成正比，而且每条并发请求各占一份。所以服务端能同时处理多少条长请求，往往受显存限制而不是算力。GQA（多个 query 头共用一组 KV）和 MLA（把 KV 压缩成低维向量，见文末历史知识点）都是在压缩这部分开销。

### 3. 长上下文的两种代价：成本和质量

- **是什么**：上下文变长，一方面是钱和延迟（token 计费、prefill 时间、KV 显存）变多，另一方面是效果变差。Anthropic 把后者叫 context rot（上下文腐化）：token 越多，模型找回和使用信息的准确度越低。
- **没有它会怎样**：以为「窗口有 1M 就全塞进去」，结果又贵又慢，关键信息还被淹没了。
- **对应你熟悉的**：一个 Activity 里塞几千行代码，能跑，但谁都看不清了。

原因是注意力要处理 n 个 token 两两之间的 n² 种关系，长度增加后注意力被摊薄；而且训练数据里长序列较少，模型对超长距离依赖的经验也少。下降是逐渐发生的，不是到某个长度突然失效。所以思路是挑选最有用的上下文，而不是尽量多塞。

### 4. 输出上限包括思考：`max_tokens` 是硬上限

- **是什么**：`max_tokens` 限制的是这一轮的全部输出，包括模型的思考（thinking）和你看到的回答。思考部分按输出 token 计费，即使不显示出来也一样。
- **没有它会怎样**：就像场景里那样，看得见的回答只有两百字就被截断了，因为大部分额度已经用在思考上。
- **对应你熟悉的**：给 OkHttp 设了整体超时，DNS 和握手占掉的时间也算在里面。

判断依据是 `stop_reason`：`max_tokens` 表示被你设的上限截断，`model_context_window_exceeded` 表示撑满了上下文窗口，两种都要当作「回答不完整」处理。如果截断发生在工具调用中间，JSON 是残缺的，要调大上限后重试，不能直接解析。

### 5. 采样参数的边界：temperature=0 ≠ 确定

- **是什么**：temperature=0 只是每一步都选概率最高的 token。但 GPU 并行计算的顺序会变化，浮点误差会让概率很接近的两个候选互换位置，后面的输出就分叉了。
- **没有它会怎样**：把「设成 0」当作可复现的保证，测试和评测都会偶尔对不上。
- **新情况**：Claude 的新模型（Opus 4.7 及以后等）设置非默认的采样参数会直接返回 400；从 API 版本 `2026-10-21` 起，所有模型都不再接受 `temperature`、`top_p`、`top_k`。官方建议用提示词来引导行为，并说明 temperature=0 在以前的模型上也从来不保证输出完全一致。

要让输出稳定，可以用这些办法：写清楚的指令和示例、结构化输出约束格式、把结果缓存起来、评测时多跑几次看分布（Best-of-N 结果不一致本身也是发现幻觉的信号）。

## 高频面试题

**Q1：用户抱怨「助手反应慢」，你怎么拆解？**
- 先区分首字慢（TTFT）和输出慢（每秒 token 数），分别在日志里打点统计。
- 首字慢：缩短输入、把固定前缀放前面用 prompt caching、减少思考量（effort）。
- 输出慢：限制回答长度、换更快的模型、开流式输出改善体感。
- 优化前先确认效果达标，不能为了快牺牲正确率。

**Q2（追问）：KV cache 为什么能省算力？它的代价和常见优化是什么？**
- 每步只算新 token 的 K/V，避免整段历史重算，计算量从平方级降到线性。
- 代价是显存：和长度、层数、KV 头数成正比，每条并发请求各一份。
- 优化：GQA/MQA 减少 KV 头数，MLA 压缩 KV，量化，分页管理显存，共享相同前缀。
- 所以长上下文服务的并发上限经常受显存限制。

**Q3：temperature=0 为什么还会有不同输出？Claude 新模型不让设采样参数，你怎么保证稳定？**
- 浮点运算顺序会随批次和硬件变化，概率接近的候选可能互换，后面的输出随之分叉。
- 新模型设采样参数会返回 400，只能靠提示词、示例和结构化输出来约束。
- 关键结果要做校验和缓存，不要依赖模型每次说的一样。
- 评测时多次采样看一致性，不一致的地方重点排查幻觉。

**Q4（场景题）：车机 Agent 的回答经常半句被截断，偶尔工具参数 JSON 解析失败，怎么排查？**
- 先看 `stop_reason`：`max_tokens` 说明额度不够，注意额度包括思考部分。
- 截断发生在工具调用中间的，丢弃残缺结果，调大 `max_tokens` 重试。
- 用 `usage` 里的思考 token 数确认额度去了哪里，再决定是调大上限还是降低 effort。
- 换了新模型要重新评估 token 数：新 tokenizer 处理同样的文本可能多用最多约 35% 的 token。

## 常见误区

- **「窗口越大越好，能塞就塞」**：长上下文会带来 context rot，还更贵更慢。窗口大小是上限，不是目标。
- **「max_tokens 只限制我看到的回答」**：它也包括思考 token，而且思考 token 按输出计费。开启思考的模型要预留足够的额度。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带 KV cache 的生成循环**（取自 `kv_cache_demo.py`）

```python
# 定义生成函数：传入提示词、要生成的新 token、是否用缓存
def generate(prompt, new, use_cache):
    # 初始化：完整序列从提示词开始，输出列表为空，K/V 计算行数清零
    xs, outs, kv_rows = prompt, [], 0
    # 用缓存时先做 prefill
    if use_cache:
        # 把提示词所有 token 的 K 和 V 一次性算出来
        K, V = prompt @ Wk, prompt @ Wv
        # 记下这次算了多少行
        kv_rows += len(prompt)
    # decode 阶段：逐个处理新 token
    for x in new:
        # 把新 token 接到完整序列末尾
        xs = np.vstack([xs, x])
        # 有缓存的分支
        if use_cache:
            # 只算新 token 的 k、v，追加到缓存末尾，计数加 1
            K, V = np.vstack([K, x @ Wk]), np.vstack([V, x @ Wv]); kv_rows += 1
        # 没缓存的分支
        else:
            # 对整段历史重新算 K、V，计数加上整段长度
            K, V = xs @ Wk, xs @ Wv; kv_rows += len(xs)
        # 用新 token 的 query 对全部 K/V 做注意力，存下结果
        outs.append(attend(x @ Wq, K, V))
    # 返回所有输出和总计算行数
    return np.array(outs), kv_rows
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 追加行用的是 `np.vstack([K, x @ Wk])`，参数是一个**列表**；直接写 `np.vstack(K, x @ Wk)` 会报错。
- 注意力用的是新 token 的 query：`x @ Wq`，不是 `xs @ Wq`。
- 没缓存的分支每步加的是 `len(xs)`，有缓存的分支每步只加 1。这两处正是两者计算量的差别。

**昨日复习（只做第③步）**：Day 30 的「带子 Agent 的研究 Agent」。只看你自己写的注释重写，重点核对：`create_deep_agent(...)` 的 `subagents` 是字典列表，必填键是 `name`、`description`、`system_prompt`；结果文件在 `r["files"]["/report.md"]["content"]`。

## 动手练习（30 分钟）

在 `kv_cache_demo.py` 的基础上模拟「前缀缓存」：车机每次请求都带同一段 2000 token 的车辆手册，后面接不同的用户问题（各 50 个 token）。

1. 写一个函数 `prefill(tokens)`，返回 K、V。
2. 第一次请求：手册和问题 A 一起 prefill；把手册部分的 K/V 单独保存下来。
3. 第二次请求：直接复用手册的 K/V，只对问题 B 做 prefill，再用 `np.allclose` 验证结果和完整计算的一致。
4. 打印两次请求一共算了多少行 K/V，和「每次全量 prefill」做对比。

想一想：如果把用户问题放在手册**前面**，前缀还能复用吗？这和 prompt caching 要求「固定内容放前面」是一个道理。

## 读后自测

1. 为什么「首个 token 出来的时间」和「后面每秒出几个 token」是两个独立指标？各自主要受什么影响？——见「核心概念」第 1 节
2. 上下文窗口有 1M，为什么还不能把资料全塞进去？——见「核心概念」第 2、3 节
3. `temperature=0` 为什么不能保证每次输出一样？在不允许设采样参数的新模型上，怎么让输出稳定？——见「核心概念」第 5 节

## AI 历史知识点：DeepSeek-V2 与多头潜在注意力 MLA（2024，DeepSeek-AI）

**是什么**：DeepSeek-V2 是一个混合专家（MoE）模型，总参数 236B，每个 token 只激活 21B，支持 128K 上下文。它最有影响的设计是 MLA（Multi-head Latent Attention，多头潜在注意力）：不再给每个注意力头分别缓存完整的 K 和 V，而是先压缩成一个低维的「潜在向量」存起来，用的时候再还原。

**何时由谁提出**：DeepSeek-AI 在 2024 年 5 月 7 日把论文提交到 arXiv。

**为什么重要**：论文报告，和自家 DeepSeek 67B 相比，KV cache 减少 93.3%，训练成本降低 42.5%，最大生成吞吐提高到 5.76 倍。它说明长上下文推理的瓶颈常在 KV cache 显存，可以靠改模型结构解决，不只是堆硬件。

**和今天的趋势有什么联系**：今天的代码里，GQA 把 KV 显存降到四分之一，MLA 压得更狠。各家都在推 1M 上下文和连续跑几天的长任务 Agent，这些都离不开更省显存的注意力结构。

**原始来源**：[DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model（arXiv:2405.04434）](https://arxiv.org/abs/2405.04434)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `stop_reason` 取值 | `end_turn` 正常结束；`max_tokens` 达到上限；`stop_sequence` 命中停止序列；`tool_use` 要调工具；`pause_turn` 服务端工具循环达到迭代上限（默认 10），原样发回继续；`refusal` 拒答（HTTP 200，看 `stop_details`）；`model_context_window_exceeded` 撑满上下文窗口 |
| 截断续写 | 把已生成的部分作为 assistant 消息发回，再追加一条 user 消息让它继续；截断在 `tool_use` 中间时调大 `max_tokens` 重试 |
| 采样参数 | Fable/Mythos 5.x、Opus 4.7 及以后、Sonnet 5/5.5、Haiku 5.5：非默认的 `temperature`/`top_p`/`top_k` 返回 400，同时传 `temperature` 和 `top_p` 也报错；API 版本 `2026-10-21` 起所有模型都拒绝这三个参数；更早的版本只在开启思考时有限制（`top_p` 允许 0.95–1） |
| 思考与 `max_tokens` | 思考 token 计入 `max_tokens`，按输出计费；`usage.output_tokens_details.thinking_tokens` 可以看思考用了多少；`max_tokens` > 21,333 时 SDK 要求用流式 |
| Opus 5.5 思考 | 思考一直开启（adaptive），`{"type":"disabled"}` 和 `budget_tokens` 会被拒绝；effort 默认 `medium`，可选 `low`/`medium`/`high`/`xhigh`/`max`；`thinking.display` 默认 `omitted` |
| 上下文溢出 | 光输入就超出窗口：400「prompt is too long」；输入 + `max_tokens` 超出窗口：Claude 4.5 及更新的模型接受请求，生成到上限时以 `model_context_window_exceeded` 停止 |
| 窗口大小 | Opus 4.6 及以后、Sonnet 4.6 及以后、Haiku 5.5 等默认 1M，单次最多输出 128k；其他模型 200k |
| 历史思考块 | Opus 4.5 及以后、Sonnet 4.6 及以后等默认保留（下一轮按输入计费）；更早的模型由 API 自动去掉；工具调用回合必须原样带回思考块和签名 |
| context awareness | Sonnet 5、Sonnet 4.6、Haiku 4.5 会收到 `<budget:token_budget>` 和剩余用量提示；Opus 4.7 及以后等没有，改用 task budgets（beta） |
| Tokenizer | Opus 4.7 起用新 tokenizer，同样的文本 token 数约为原来的 1–1.35 倍；用 `/v1/messages/count_tokens` 计数，不要按固定的字符/token 比例估算 |
| LangChain 参数 | `init_chat_model(model, temperature=, max_tokens=, timeout=, max_retries=6)`；token 用量看 `AIMessage.usage_metadata` 或 `UsageMetadataCallbackHandler`；推理强度用 `reasoning_effort`；`InMemoryRateLimiter` 只限制请求数，不限制大小 |
| KV cache 公式 | 每条请求的字节数 = 2（K 和 V）× 层数 × KV 头数 × 每头维度 × 序列长度 × 每个数的字节数 |
| 降低幻觉（官方） | 允许说「不知道」；先摘原文再回答；要求引用，没有引用支撑的说法就撤回；看思考摘要、Best-of-N 一致性检查、限定只用给定资料 |

### 来源说明

- 代码只用了 numpy，没有调用模型 API（本次运行环境没有 API Key，也连不上 Hugging Face），演示的是原理，不代表真实模型的数字。
- 「今日 AI 动态」来自 Google 和 Meta 的官方博客；历史知识点来自 arXiv 论文（非 LangChain/Claude 来源）。

### 延伸学习

- Claude Academy《Building with the Claude API》：「Accessing Claude with the API」一章里的 Temperature 和 Response streaming 两课，以及「Features of Claude」里的 Extended thinking 和 Prompt caching。

### 参考资料

- [Context windows - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Stop reasons and fallback - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
- [Thinking - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/thinking)
- [Adaptive thinking - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Claude Opus 5.5 migration guide](https://platform.claude.com/docs/en/models/opus-5-5/migration-guide)
- [Reduce hallucinations - Claude Platform Docs](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)
- [Reduce latency - Claude Platform Docs](https://platform.claude.com/docs/fr/test-and-evaluate/strengthen-guardrails/reduce-latency)
- [Effective context engineering for AI agents - Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Models - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/models)
- [Building with the Claude API · Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
