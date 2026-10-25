# Day 27 · 模型适配（2026-10-25）

## 今日 AI 动态（2026-10-25）

过去 24 小时关注的厂商没有新的官方发布。

---

## 读前三问

1. 模型答得不好，Prompt、RAG、微调三条路怎么选？顺序是什么？
2. SFT、RLHF、DPO 分别在教模型什么？数据长什么样？
3. 什么情况下才值得微调？微调完怎么证明它真的更好？

## 从一个场景开始

车机语音助手上线一个月，车主说「有点冷」，助手有时回「已将空调调到 24℃」，有时却回「建议您多穿衣服」。产品经理第一反应：「拿日志去微调一个模型吧。」

先别急。把问题拆开看：

- 助手不知道这台车支持哪些空调指令？这是**缺知识**，加 RAG 或工具描述就够。
- 它知道指令，但不明白「有点冷」就该直接调温度？这是**不懂规矩**，先在 system prompt 写规则、加几个示例。
- 规则和示例都加了，评测还是卡在 85%，而且线上已经攒了几千条车主点 👍👎 的日志？这时才轮到**微调**：用好回答做 SFT，用「同一句话的一好一坏」做 DPO。

今天的主题：**模型适配**——什么时候改提示词、什么时候接知识库、什么时候动模型本身。

## 一句话总结

先用提示词和示例改行为，缺知识就接检索；只有评测证明这两条到顶、又有足够标注数据时才微调：SFT 教模型照着答，DPO 教模型在两个回答里偏向好的那个。

## 先跑起来

不调任何模型 API，用纯 Python 把车机日志变成 SFT 和 DPO 两种训练数据，再手算一次 DPO 损失，看「学对 / 学反」时 loss 怎么变。

```python
# car_adapt.py —— 车载语音助手：从线上日志造 SFT / DPO 数据，并手算一次 DPO 损失（纯 Python）
import json, math

logs = [  # 线上日志：用户说法、助手回复、车主点了👍还是👎
    {"q": "有点冷", "a": "已将空调调到 24℃", "vote": "up"},
    {"q": "有点冷", "a": "建议您多穿衣服", "vote": "down"},
    {"q": "导航回家", "a": "正在规划回家路线", "vote": "up"},
]

def msgs(q, a):  # 对话格式：一问一答
    return [{"role": "user", "content": q}, {"role": "assistant", "content": a}]

# SFT 数据：只要「好回答」，教模型照着答
sft = [{"messages": msgs(r["q"], r["a"])} for r in logs if r["vote"] == "up"]
# DPO 数据：同一个问题的一好一坏配成一对，教模型「更偏向哪个」
ups = {r["q"]: r["a"] for r in logs if r["vote"] == "up"}
dpo = [{"prompt": r["q"], "chosen": ups[r["q"]], "rejected": r["a"]}
       for r in logs if r["vote"] == "down" and r["q"] in ups]
print("SFT 条数:", len(sft), "| DPO 对数:", len(dpo))
print(json.dumps(dpo[0], ensure_ascii=False))

def dpo_loss(pi_w, pi_l, ref_w, ref_l, beta=0.1):
    # 参数是对数概率 log p(回答|问题)；ref 是训练前冻结的参考模型
    margin = beta * ((pi_w - ref_w) - (pi_l - ref_l))  # 好回答比参考模型多涨了多少，减去坏回答涨了多少
    return -math.log(1 / (1 + math.exp(-margin)))      # -log sigmoid(margin)

ref_w, ref_l = -12.0, -10.0  # 训练前：模型其实更想说「多穿衣服」
for name, pi_w, pi_l in [("刚开始=参考模型", -12.0, -10.0),
                         ("学对了：好回答↑ 坏回答↓", -9.0, -15.0),
                         ("学反了：坏回答↑", -14.0, -8.0)]:
    print(f"{name:<16} loss={dpo_loss(pi_w, pi_l, ref_w, ref_l):.3f}")
for beta in (0.1, 0.5):  # beta 大：稍微偏离参考就能把 loss 压低，模型不必走远（约束更紧）
    print(f"beta={beta} 学对了 loss={dpo_loss(-9.0, -15.0, ref_w, ref_l, beta):.3f}")
```

**运行**：`python car_adapt.py`（只用标准库）

**实测输出**（2026-10-25，Python 3.13）：

```
SFT 条数: 2 | DPO 对数: 1
{"prompt": "有点冷", "chosen": "已将空调调到 24℃", "rejected": "建议您多穿衣服"}
刚开始=参考模型         loss=0.693
学对了：好回答↑ 坏回答↓    loss=0.371
学反了：坏回答↑         loss=0.913
beta=0.1 学对了 loss=0.371
beta=0.5 学对了 loss=0.018
```

**该看什么**：
- 同一份日志，SFT 只用了 👍 的两条；DPO 只用了「同一句话既有 👍 又有 👎」的那一对。**数据形状决定能用哪种方法**。
- 刚开始策略模型和参考模型一样，loss = ln2 ≈ 0.693，相当于「不知道偏向谁」。学对了 loss 下降，学反了上升。
- loss 看的是**相对参考模型的变化**，不是绝对概率：训练前模型更想说「多穿衣服」也没关系，只要训练让好回答涨、坏回答跌就行。

## 核心概念

### 1. 适配阶梯：Prompt → RAG → 微调

- **是什么**：从便宜到昂贵依次尝试。改提示词以分钟计；接检索以天计；微调要准备数据、训练、评测、部署，以周计，而且每次换底座模型都要重来。
- **没有它会怎样**：一上来就微调，花了几周，结果发现问题只是 system prompt 没写清楚；或者想用微调「灌知识」，模型照样编造，知识一更新又得重训。
- **对应你熟悉的**：改配置（prompt）→ 接远程配置服务（RAG）→ 改代码重新发版（微调）。

判断口诀：**缺知识用 RAG，缺规矩用 Prompt，缺「手感」才微调**。Anthropic 的经验是：知识库小于约 20 万 token（约 500 页）时，可以直接整段放进提示词，配合 prompt caching，连 RAG 都不用搭。

### 2. 动态示例：微调之前的最后一招

- **是什么**：把标注好的「问法 → 正确回答」存进向量库，每次请求检索最相近的几条，作为 few-shot 示例塞进提示词。
- **没有它会怎样**：意图类别多、问法五花八门时，固定示例放不下；很多团队就在这里误以为「只能微调了」。
- **对应你熟悉的**：`RecyclerView` 不一次加载全部数据，按需取当前屏幕要的几条。

Claude 官方工单分类案例里，用相似度检索示例让准确率从 71% 提到 93%。它本质是「用 RAG 检索示例」，数据和微调要的是同一批标注，但不用训练，随时可以增删。

### 3. SFT：照着标准答案学

- **是什么**：监督微调。给模型「问题 + 标准回答」，让它提高说出这个回答的概率。数据只需要好样本。
- **没有它会怎样**：底座模型不熟悉你的输出格式、领域术语或语气，提示词写得再长也不稳定。
- **对应你熟悉的**：给新同事一本「标准话术手册」照着背。

SFT 擅长教**格式和风格**，不擅长教**知识**：模型会学会「语气像真的」，但事实仍可能编造。它也通常是 RLHF/DPO 之前的第一步，DPO 的参考模型一般就是 SFT 后的模型。

### 4. RLHF 与 DPO：学会「哪个更好」

- **是什么**：两者都用**偏好数据**（同一问题的一好一坏）教模型偏向好回答。RLHF 先训练一个奖励模型打分，再用强化学习（常用 PPO）优化；DPO 把这两步合成一个分类式损失，直接在偏好对上训练。
- **没有它会怎样**：很多质量差异写不成标准答案，只能说「A 比 B 好」，比如更简洁、更少反问。SFT 没法利用这类信号。
- **对应你熟悉的**：A/B 实验只告诉你哪个版本好，不告诉你完美版本长什么样。

DPO 有两个关键：**参考模型**（训练前冻结的副本，防止模型跑偏到胡说）和 **β**（控制允许偏离参考模型多远）。论文结论是效果不输 PPO 版 RLHF，但不需要单独的奖励模型、训练中也不用采样，简单稳定得多。

### 5. 何时值得微调，以及怎么证明

- **是什么**：微调前先满足三个条件：有评测集且 Prompt/RAG 已到顶；有足够的高质量标注（几百到几千条，按任务而定）；任务稳定，不会每月变。
- **没有它会怎样**：没有评测集，就没法证明微调后更好，也发现不了「学会新任务、忘了旧本事」（灾难性遗忘）。
- **对应你熟悉的**：重构前先写单测，重构后跑同一套测试。

还要考虑供应商：Claude API 目前不开放自助微调（需联系 Anthropic），想微调通常选开源模型（Qwen、Llama 等）自己训。车载端侧场景常见做法是：云端大模型负责难题，小模型微调后放在车机上处理高频指令。

## 高频面试题

**1. 用户反馈「回答不准」，你怎么判断是用 RAG 还是微调？**
- 先看错因：事实错/知识过时 → RAG；格式、语气、流程不对 → Prompt 或微调。
- 知识会更新的，一律别靠微调灌进去，重训成本高且不可追溯。
- 先建评测集量化，再按 Prompt → 动态示例 → RAG → 微调逐级试。
- 两者可叠加：微调学格式和工具调用习惯，RAG 提供最新知识。

**2.（追问）DPO 为什么需要参考模型？去掉会怎样？**
- 损失看的是「相对参考模型」好回答涨、坏回答跌的幅度。
- 没有参考约束，模型可以把坏回答概率压到极低来刷分，输出质量反而崩坏。
- 参考模型通常是 SFT 后的模型，训练中冻结不更新。
- β 越大约束越紧，越接近参考模型。

**3.（场景题）车机语音助手攒了 5000 条带 👍👎 的日志，你会怎么用？**
- 先清洗：去隐私、去重、抽样人工复核，留一部分做评测集。
- 👍 样本做动态示例库，先不训练就看能提升多少。
- 仍不够再训：👍 做 SFT，同一问法的 👍/👎 配对做 DPO。
- 上线前在评测集上对比旧模型，灰度发布，持续收集新反馈。

**4. SFT 和 RLHF/DPO 有什么区别？顺序呢？**
- SFT 要「标准答案」，教模型怎么答；偏好学习要「好坏对比」，教模型哪个更好。
- 通常先 SFT 打底，再做偏好对齐。
- RLHF 需要奖励模型 + 强化学习；DPO 直接优化，更省资源。
- 偏好数据更容易收集：人判断好坏比写标准答案容易。

## 常见误区

1. **「微调能让模型记住我们的知识库」**：微调主要改变行为和风格，知识灌不牢、更新难、无法引用来源；知识问题交给 RAG。
2. **「数据越多越好，日志全拿去训」**：低质、重复、带偏见的数据会让模型学坏；几百条精选数据往往胜过几万条脏数据，而且必须留出评测集。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：从日志造 SFT / DPO 数据 + DPO 损失**（取自 `car_adapt.py`）

```python
# 定义函数：把一问一答包装成对话消息列表
def msgs(q, a):
    # 返回用户消息和助手消息两条
    return [{"role": "user", "content": q}, {"role": "assistant", "content": a}]

# SFT 数据：只挑点赞的日志，每条包成 messages
sft = [{"messages": msgs(r["q"], r["a"])} for r in logs if r["vote"] == "up"]
# 建一个「问题 → 点赞回答」的字典，方便配对
ups = {r["q"]: r["a"] for r in logs if r["vote"] == "up"}
# DPO 数据：遍历点踩的日志，若同一问题有点赞回答，就配成 chosen/rejected 一对
dpo = [{"prompt": r["q"], "chosen": ups[r["q"]], "rejected": r["a"]}
       for r in logs if r["vote"] == "down" and r["q"] in ups]

# 定义 DPO 损失：输入策略模型和参考模型对好/坏回答的对数概率，beta 默认 0.1
def dpo_loss(pi_w, pi_l, ref_w, ref_l, beta=0.1):
    # 好回答相对参考涨了多少，减去坏回答相对参考涨了多少，再乘 beta
    margin = beta * ((pi_w - ref_w) - (pi_l - ref_l))
    # 返回负的 log sigmoid(margin)
    return -math.log(1 / (1 + math.exp(-margin)))
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `margin` 里是 `(pi_w - ref_w) - (pi_l - ref_l)`，两个括号内都是「策略减参考」，顺序别反。
- sigmoid 是 `1 / (1 + exp(-x))`，负号在 `exp` 里面。
- DPO 推导式里 `ups[r["q"]]` 必须配 `r["q"] in ups` 过滤，否则没有点赞回答的问题会 `KeyError`。

**昨日复习（只做第③步）**：Day 26 的「带重试的节点 + SQLite 检查点 + 断点续跑」。只看你自己写的注释重写，重点核对：续跑传 `None`、`thread_id` 放在 `configurable` 里、`retry_policy=` 挂在 `add_node` 上。

## 动手练习（30 分钟）

在 `car_adapt.py` 上改：

1. 给 `logs` 再加 3 条：「有点热」的一好一坏，和一条只有 👎 的「打开天窗」。确认 SFT 和 DPO 条数各是多少，并解释为什么「打开天窗」两边都用不上。
2. 把 `sft` 和 `dpo` 分别写成 `sft.jsonl`、`dpo.jsonl`（每行一个 JSON，`ensure_ascii=False`）。
3. 写一个 `choose(err_type, has_eval, n_labels)` 函数：返回 `"prompt" / "rag" / "dynamic_few_shot" / "sft" / "dpo"` 之一，规则按核心概念 1、2、5 来定，并为每个分支写一条测试。

## 读后自测

1. Prompt、RAG、微调怎么选？顺序是什么？→ 见核心概念 1、2，面试题 1
2. SFT、RLHF、DPO 分别在教什么？数据长什么样？→ 见先跑起来、核心概念 3、4，面试题 2、4
3. 什么时候值得微调？怎么证明更好？→ 见核心概念 5，面试题 3

---

## AI 历史知识点：直接偏好优化 DPO（2023，斯坦福大学）

**是什么**：一种用偏好数据（同一问题的好回答和坏回答）对齐大模型的方法。它把 RLHF 里「训练奖励模型 + 强化学习」两步，化简成一个直接在偏好对上计算的分类式损失。

**何时由谁提出**：2023 年 5 月 29 日，斯坦福大学的 Rafael Rafailov、Archit Sharma、Eric Mitchell、Stefano Ermon、Christopher D. Manning、Chelsea Finn 在 arXiv 发表《Direct Preference Optimization: Your Language Model is Secretly a Reward Model》。

**为什么重要**：论文证明语言模型本身就隐含一个奖励函数，因此不必单独训练奖励模型，也不需要在训练中从模型采样。实验（最大 6B 参数）显示，它在情感控制上优于 PPO 版 RLHF，在摘要和单轮对话上持平或更好，同时训练更稳定、调参更少。

**和今天的趋势有什么联系**：DPO 让偏好对齐从「大实验室专属」变成普通团队用几张卡就能做的事，开源社区大量模型都用它或它的变体做后训练。对 Agent 开发者来说，把线上 👍👎 日志变成 DPO 数据、训练自己的小模型，已经是企业落地的常见路线。

原始来源：[arXiv:2305.18290](https://arxiv.org/abs/2305.18290v1)、[论文 HTML 版](https://arxiv.org/html/2305.18290v2)

---

## 参考区（用时查，不必通读）

### 细节速查

**DPO 损失**：`L = -E[ log σ( β·log(πθ(yw|x)/πref(yw|x)) - β·log(πθ(yl|x)/πref(yl|x)) ) ]`
- `yw` 好回答（chosen），`yl` 坏回答（rejected），`πref` 冻结的参考模型（通常是 SFT 模型；没有时可先对 chosen 做极大似然得到）。
- `β` 是 KL 约束强度，同时缩放隐式奖励；常见取值 0.1 左右。
- 隐式奖励：`r(x,y) = β·log(πθ(y|x)/πref(y|x))`。
- 损失对每个样本按「隐式奖励排错的程度」加权，论文指出去掉加权会让模型退化。

**数据格式（常见约定）**
- SFT：`{"messages": [{"role": "user", ...}, {"role": "assistant", ...}]}`
- 偏好：`{"prompt": ..., "chosen": ..., "rejected": ...}`
- LangSmith 数据集可导出为 CSV、JSONL 或 OpenAI 微调格式；可把过滤后的实验 trace 加入数据集。

**方法对比**

| 方法 | 需要的数据 | 改什么 | 成本 | 适合 |
|---|---|---|---|---|
| Prompt / few-shot | 规则 + 少量示例 | 输入 | 最低 | 格式、流程、语气 |
| 动态示例 | 标注样本库 + 向量检索 | 输入 | 低 | 类别多、问法多变 |
| RAG | 文档库 | 输入 | 中 | 知识、时效性、可引用 |
| SFT | 好样本 | 权重 | 高 | 稳定格式、领域风格、小模型蒸馏 |
| RLHF（PPO） | 偏好对 + 奖励模型 | 权重 | 最高 | 大规模对齐 |
| DPO | 偏好对 | 权重 | 中高 | 偏好对齐、无奖励模型 |

**Claude 官方相关要点**
- Claude API 目前不提供微调，需要时联系 Anthropic；Claude 本身已经过微调和 RLHF。
- 知识库 < 约 20 万 token（约 500 页）：可直接放进提示词并用 prompt caching。
- 开始提示工程前要有：明确的成功标准、可实测的评测方法、一版初稿提示词；延迟和成本问题有时换模型更快解决。
- 工单分类案例：检索相似示例让准确率 71% → 93%；评测指标示例为准确率（精确匹配）和单次分类成本（来自 `usage`）。

### 来源说明

- DPO 论文为非 LangChain/Claude 来源（arXiv 原文）。
- 示例中的对数概率是手写的演示数字，不来自真实模型；代码已在本地实测。

### 延伸学习

- Claude Platform Docs · Define success criteria and build evaluations（微调前先建评测）
- LangSmith · Manage datasets（数据集版本、拆分、导出微调格式）

### 参考资料

- [Glossary - Claude Platform Docs](https://platform.claude.com/docs/en/about-claude/glossary)
- [Prompt engineering overview - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)
- [Ticket routing - Claude Platform Docs](https://platform.claude.com/docs/en/about-claude/use-case-guides/ticket-routing)
- [Contextual retrieval - Anthropic Engineering](https://www.anthropic.com/engineering/contextual-retrieval)
- [Manage datasets - Docs by LangChain](https://docs.langchain.com/langsmith/index-datasets-for-dynamic-few-shot-example-selection)
- [Direct Preference Optimization (arXiv:2305.18290)](https://arxiv.org/html/2305.18290v2)
