# Day 21 · 评测（第二轮：评委先过评、分数差要带误差棒、多轮要有模拟用户）（2026-11-18）

## 今日 AI 动态（2026-11-18）

### 趋势观察

本期官方新内容只来自 Google，两条都和「Agent 交付到哪一步」有关。AI Studio 的 Build Agent 从「生成代码」推进到「生成 Kotlin 应用、配好后端、签名、发布到 Play」，覆盖了 Android 开发的完整链路。Drive 的 Gemini 数据分类正式发布时，强调的是「部署前先在样本上测试指令、上线后看实时指标」，这正是今天讲的「评委先过评」在企业产品里的样子。推测：企业版 AI 功能会越来越多地把「上线前评测 + 上线后监控」做成管理员可见的标配。

### 今日要点

- Google AI Studio 可以从一句话生成带 Firestore 后端、Google 登录和 Gemini 功能的 Kotlin Android 应用，并发布到个人 Play 库。
- Google Drive 的 Gemini 数据分类正式发布：管理员可先测试自然语言分类指令再部署，并查看近实时的标注指标。

### 分公司动态

#### 1. Google

**[Build full-stack Android apps with Gemini in Google AI Studio](https://developers.googleblog.com/build-full-stack-android-apps-with-gemini-in-google-ai-studio/)**（2026-11-17）
AI Studio 的 Build Agent 能把一句需求变成 Kotlin Android 应用：自动配置 Cloud Firestore 和安全规则、Firebase 登录；通过 Firebase AI Logic 调 Gemini，APK 里不放 API Key，用 App Check 校验请求来源。应用可以签名后发布到个人 Google Play 库，不需要 USB 线和完整的开发者账号。浏览器里的模拟器启动更快，可以截一张运行画面附进提示词来改 UI；构建用的是官方 Android CLI 和 Gradle。
- 为什么重要：对 Android 开发者来说，「Agent 能不能做完整个 App」已经有官方答案，面试时谈 Code Agent 可以拿它当例子：生成、构建、在模拟器里看效果、再迭代。

**[Gemini-powered data classification in Google Drive is now generally available](https://workspaceupdates.googleblog.com/2026/11/gemini-powered-data-classification-in-Google-Drive-is-now-generally-available.html)**（2026-11-17）
8 月公测的 Drive AI 分类功能正式发布。管理员用自然语言写分类指令，可以先在后台测试再部署；上线后能看到近实时的指标：Gemini 标了哪些文件、用户接受了多少标签。Gemini 不会覆盖用户手动打的标签和 DLP 规则打的标签。11 月 17 日起逐步推送，预计 12 月 15 日完成。
- 为什么重要：「先测再上、上线看接受率」就是 LLM 评委的校准流程，只是换成了管理员能操作的界面。

### 动态来源

- [Build full-stack Android apps with Gemini in Google AI Studio - Google Developers Blog](https://developers.googleblog.com/build-full-stack-android-apps-with-gemini-in-google-ai-studio/)
- [Google Workspace Updates: November 2026](https://workspaceupdates.googleblog.com/2026/11/)

---

> 第二轮说明：第一轮 Day 21 讲过评测集从失败里长出来、代码 > 模型 > 人三类评分器、评结果 vs 评轨迹、pass@k vs pass^k。今天往下挖三件事：**LLM 评委本身靠不靠谱**、**分数差几个点算不算数**、**多轮对话怎么评**。

## 读前三问

1. 用 LLM 当评委，你怎么证明这个评委是可信的？
2. 新版本评测分高了 5 个点，你怎么判断是真进步还是噪声？
3. 一个要和用户来回说几句的车机 Agent，评测时「用户」从哪来？该评什么？

## 从一个场景开始

上一轮你给车机 Agent 搭好了回归评测：40 道题，代码评分 + LLM 评委，每次改提示词都跑一遍。

这周你改了提示词，让 Agent 在「车里有点闷」这类模糊指令下先反问。跑完评测，**分数掉了 7.5 个点**。产品经理说回滚。

你不甘心，又跑了一次，这次**高了 12 个点**。两次结果方向相反，到底信哪个？

更麻烦的是，测试同学发现：旧版本把空调调到 22 度，车主说「我要开窗」，它又开了窗。LLM 评委只看最后一句「已打开车窗」，判了通过。可车主根本没要空调。

三个问题：**评委自己会不会判错**、**分数波动多大算正常**、**多轮对话里中间的错怎么抓**。

## 一句话总结

评委要先和人工标注对齐才能用；比较两个版本时，每题多跑几次、按题配对，算出差值的置信区间再下结论；多轮对话用模拟用户跑完整段对话，评最终状态，不只看最后一句话。

## 先跑起来

两个文件。`paired_ci.py` 只用标准库，模拟 v1、v2 在 40 道题上各跑 5 次，真实提升设为 6 个百分点，看不同算法下结论怎么变。`sim_eval.py` 用 openevals 的多轮模拟器，让一个「会反应的车主」分别和 v1、v2 对话，再按终态打分。没有 API Key，Agent 和车主都用剧本代替 Claude，模拟器、评分流程是真实库代码。

```python
# paired_ci.py —— v2 比 v1 高几分，是真进步还是噪声？（只用标准库）
import random, statistics as st

rng = random.Random(5)
N, K = 40, 5                                           # 40 道题，每题跑 5 次
difficulty = [rng.uniform(0.1, 1.0) for _ in range(N)]  # 每道题本身的通过概率（题难度不同）

def run(p_boost):                                      # 模拟一次评测：返回 N×K 的 0/1 结果
    return [[int(rng.random() < min(1, p + p_boost)) for _ in range(K)] for p in difficulty]

v1, v2 = run(0.0), run(0.06)                           # v2 真实提升 6 个百分点

def ci(xs):                                            # 均值 ± 1.96 × 标准误
    m, se = st.mean(xs), st.stdev(xs) / len(xs) ** 0.5
    return f"{m:+.3f}  95%CI [{m - 1.96*se:+.3f}, {m + 1.96*se:+.3f}]  SE={se:.3f}"

# ① 只看第 1 次运行（k=1），两组当独立样本
a1, b1 = [r[0] for r in v1], [r[0] for r in v2]
se_unpaired = (st.variance(a1) / N + st.variance(b1) / N) ** 0.5
print(f"k=1 不配对：差 {st.mean(b1)-st.mean(a1):+.3f}  SE={se_unpaired:.3f}")
# ② k=1，但按题配对：同一道题上的差
print("k=1 配对：  差", ci([b - a for a, b in zip(a1, b1)]))
# ③ 每题跑 K 次取平均再配对：随机波动被平均掉
a5, b5 = [st.mean(r) for r in v1], [st.mean(r) for r in v2]
print("k=5 配对：  差", ci([b - a for a, b in zip(a5, b5)]))
print(f"两版本逐题得分相关系数 r={st.correlation(a5, b5):.2f}")
```

```python
# sim_eval.py —— 多轮评测：模拟车主 + 终态评分（openevals 0.2.5）
from openevals.simulators import run_multiturn_simulation

def make_app(version, car):                         # 剧本版车机 Agent（无 API Key，代替 Claude）
    def app(msg, *, thread_id, **kw):
        text = msg["content"]
        if "开窗" in text:
            car["window"] = "open"; return {"role": "assistant", "content": "已打开车窗"}
        if version == "v2" and "闷" in text:          # v2：信息不足先反问
            return {"role": "assistant", "content": "要开空调还是开窗？"}
        car["ac"] = 22; return {"role": "assistant", "content": "已把空调调到 22℃"}
    return app

def owner(trajectory, *, thread_id, turn_counter):   # 模拟车主：会根据 Agent 的回复做反应
    if turn_counter == 0:
        return {"role": "user", "content": "车里有点闷"}
    last = trajectory[-1]["content"]
    if "？" in last: return {"role": "user", "content": "开窗吧"}
    return {"role": "user", "content": "我不要空调，我想开窗！"}   # 被猜错时车主纠正

def run(version):
    car = {"window": "closed", "ac": None}
    def end_state(*, outputs, reference_outputs=None):   # 评终态：窗开了、空调没被乱动
        return {"key": "end_state", "score": car == reference_outputs}
    def turns(*, outputs, reference_outputs=None):       # 评过程：车主说了几句话
        return {"key": "user_turns", "score": sum(m["role"] == "user" for m in outputs)}
    res = run_multiturn_simulation(
        app=make_app(version, car), user=owner, max_turns=4,
        stopping_condition=lambda t, turn_counter: "车窗" in t[-1]["content"],
        trajectory_evaluators=[end_state, turns],
        reference_outputs={"window": "open", "ac": None})
    print(f"== {version}")
    for m in res["trajectory"]: print(f"  {m['role']:>9}: {m['content']}")
    print("  评分:", {r["key"]: r["score"] for r in res["evaluator_results"]})

run("v1"); run("v2")
```

**运行**：`pip install openevals && python paired_ci.py && LANGSMITH_TRACING=false python sim_eval.py`

**实测输出**（2026-11-18，Python 3.13，openevals 0.2.5 / langchain-core 1.8.2，各运行 2 次结果一致）：

```
k=1 不配对：差 -0.075  SE=0.110
k=1 配对：  差 -0.075  95%CI [-0.222, +0.072]  SE=0.075
k=5 配对：  差 +0.095  95%CI [+0.023, +0.167]  SE=0.037
两版本逐题得分相关系数 r=0.78
```

```
== v1
       user: 车里有点闷
  assistant: 已把空调调到 22℃
       user: 我不要空调，我想开窗！
  assistant: 已打开车窗
  评分: {'end_state': False, 'user_turns': 2}
== v2
       user: 车里有点闷
  assistant: 要开空调还是开窗？
       user: 开窗吧
  assistant: 已打开车窗
  评分: {'end_state': True, 'user_turns': 2}
```

**该看什么**：
- 第一段：v2 真实更好，但只跑 1 次时测出来是 **-7.5 分**，置信区间横跨 0，什么也说明不了。每题跑 5 次、按题配对后，标准误从 0.110 降到 0.037，区间整体落在 0 以上。（换一个随机种子再跑，数字会变，这正是要点。）
- 第二段：两个版本最后一句都是「已打开车窗」。只看最后回复的评委会判两个都通过；终态评分发现 v1 多开了空调。
- 两版本的 `user_turns` 一样，说明「轮数」单独看区分不了好坏，要和终态一起看。

## 核心概念

### 1. 评委要先被评

- **是什么**：用 LLM 当评委之前，先拿一批人工标好的样本跑一遍，看评委和人的判断一致率有多高。
- **没有它会怎样**：评委自己的偏差会混进分数。你以为 Agent 变好了，其实是评委变宽松了。
- **对应你熟悉的**：上线一个新的崩溃聚类规则前，先拿历史已人工归类的崩溃对一遍。

LangSmith 的做法：人工标至少 20 条、0 和 1 大致各半，算「一致率」，看不一致的样本归类原因，再改评委提示词。Anthropic 的建议是：评分标准写细、每个维度单独一个评委、允许评委回答「Unknown」，评委尽量用和被测 Agent 不同的模型。

### 2. 分数差要带误差棒

- **是什么**：评测分是从「所有可能的题」里抽样得到的估计值，要报**标准误**和**置信区间**，而不是一个裸数字。
- **没有它会怎样**：场景里一次 -7.5、一次 +12，都可能只是抽样噪声。
- **对应你熟悉的**：启动耗时 A/B 测试，不会只测一次就下结论，要多次取均值、看方差。

Anthropic 给了三个降噪手段：**每题多次采样取平均**（去掉模型自身的随机性）；**按题配对比较**（同一道题上看 v2 − v1，去掉题目难度的差异，两版本得分越相关，收益越大）；**同源题按组算误差**（比如 5 道题出自同一段对话，它们不独立，算标准误时要按组聚类，否则误差会被低估）。

### 3. 多轮评测：模拟用户 + 终态

- **是什么**：让另一个 LLM（或脚本）扮演用户，按人设和 Agent 来回对话，跑完后评**最终状态**和**交互质量**。
- **没有它会怎样**：固定的用户台词不会对 Agent 的回复做反应。Agent 反问了，固定台词却还在说下一句，对话就对不上。
- **对应你熟悉的**：Espresso 测试里的 Fake 服务端，按请求内容返回不同响应，而不是固定回放一份录音。

openevals 的 `run_multiturn_simulation` 接收 `app`、`user`、`max_turns` 或 `stopping_condition`，结束后把整段对话交给评分器。τ-bench 也是这个思路：既查数据库终态，也看对话过程。

### 4. 先怀疑评测，再怀疑 Agent

- **是什么**：分数异常时，先读对话记录（transcript），确认是 Agent 错了还是评测错了。
- **没有它会怎样**：Anthropic 提到，Opus 4.5 在 CORE-Bench 上起初只有 42%，修掉评分过死（要求 `96.124991…`，拒绝了 `96.12`）、题目描述含糊等问题后，约为 95%。
- **对应你熟悉的**：CI 红了先看是不是 flaky test，再去改业务代码。

三个信号：前沿模型多次运行全是 0 分（0% pass@100），多半是题坏了；分数接近满分说明评测**饱和**，该加难题，旧题转入回归集；每次运行要从干净环境开始，上一次留下的文件或缓存会让结果互相影响。

## 高频面试题

**1. LLM 当评委，你怎么保证它可信？**
- 先用人工标注集（至少几十条，正负样本均衡）测一致率，达标才上线。
- 评分标准写成可检查的二元条款，每个维度单独评，允许输出「Unknown」。
- 评委用和被测 Agent 不同的模型，开思考，先推理再给分。
- 上线后定期抽样人工复核，持续把分歧样本加进标注集。

**2.（追问）新版本分数高了 5 个点，你会直接上线吗？**
- 先看置信区间：报均值 ± 1.96 × 标准误，区间跨 0 就不算显著。
- 两版本在同一批题上按题配对比较，比两组独立比较误差小。
- 每题多跑几次取平均，去掉模型的随机性。
- 题量不够就先做功效分析，算出要多少题才能测出 5 个点的差距。

**3.（追问）Agent 评测分数突然很低，你第一步做什么？**
- 先读失败样本的完整记录，判断是 Agent 错还是评分错。
- 检查有没有 0% pass@k 的题：可能题目不可解或标准答案有误。
- 确认每次运行环境是干净的，排除状态残留和基础设施报错。
- 准备一份参考解，先验证它能通过全部评分器。

**4.（场景题）车机语音 Agent 要上线「多轮澄清」功能，你怎么设计评测？**
- 按真实日志写车主人设：模糊型、急躁型、中途改主意型，用模拟用户跑多轮。
- 主指标是终态：车窗、空调、导航状态和预期一致，且没有多余动作。
- 辅助指标：澄清轮数、是否在信息足够时还在追问，用 LLM 评委按细则打分。
- 每个场景跑多次算 pass^k，车机更看重「每次都对」。

## 常见误区

1. **「评委用最强的模型就不用校准了」**：模型再强，细则含糊时判断也会前后不一致（Anthropic 原话：含糊的细则产生不一致的判断）。不和人工标注对一遍，你不知道它在哪类样本上会判错。
2. **「多轮对话只评最后一句就够了」**：今天的 v1 最后一句完全正确，但中间多开了空调。有副作用的 Agent 必须评终态。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：配对比较 + 置信区间**（取自 `paired_ci.py`）

```python
# 两个版本各跑一遍评测，v2 带一点真实提升
v1, v2 = run(0.0), run(0.06)

# 给一组差值算均值和 95% 置信区间
def ci(xs):
    # 均值，和标准误 = 样本标准差 / 根号 n
    m, se = st.mean(xs), st.stdev(xs) / len(xs) ** 0.5
    # 区间 = 均值 ± 1.96 倍标准误
    return f"{m:+.3f}  95%CI [{m - 1.96*se:+.3f}, {m + 1.96*se:+.3f}]  SE={se:.3f}"

# 每道题只取第 1 次运行的结果
a1, b1 = [r[0] for r in v1], [r[0] for r in v2]
# 不配对：两组方差各除以 n 相加再开方
se_unpaired = (st.variance(a1) / N + st.variance(b1) / N) ** 0.5
# 打印不配对时的差值和标准误
print(f"k=1 不配对：差 {st.mean(b1)-st.mean(a1):+.3f}  SE={se_unpaired:.3f}")
# 配对：同一道题上 v2 减 v1，再对差值算区间
print("k=1 配对：  差", ci([b - a for a, b in zip(a1, b1)]))
# 每道题 K 次结果先取平均，降低随机波动
a5, b5 = [st.mean(r) for r in v1], [st.mean(r) for r in v2]
# 多次平均后再配对
print("k=5 配对：  差", ci([b - a for a, b in zip(a5, b5)]))
# 两版本逐题得分越相关，配对的好处越大
print(f"两版本逐题得分相关系数 r={st.correlation(a5, b5):.2f}")
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 标准误除的是 `len(xs) ** 0.5`（根号 n），不是 `len(xs)`；`** 0.5` 外面的括号别漏。
- 配对时是 `zip(a1, b1)` 后逐题相减，`b - a` 的顺序决定正负号：正数表示 v2 更好。
- `st.stdev` 是标准差、`st.variance` 是方差，不配对公式里用的是方差。

**昨日复习（只做第③步）**：第二轮 Day 20 的「流式收集 + 收尾校验」。只看你自己写的注释重写，重点核对：`max_tokens` 判断放在 `try` 之前、两个 `except` 的顺序、`ensure_ascii=False`。

## 动手练习（30 分钟）

在今天两个文件的基础上改：

1. 把 `paired_ci.py` 的随机种子换成 1~10 各跑一次，记下 k=1 和 k=5 时有几次结论是「显著更好」（区间下限 > 0）。再把 `N` 改成 100，看次数怎么变。
2. 在 `sim_eval.py` 里加一个「急躁车主」人设：Agent 一反问就回「随便！」，看 v2 会不会卡住；加一个评分器，Agent 连续反问两次就判失败。
3. 写一个 `agreement(judge, human)` 函数：输入两个 0/1 列表，返回一致率。构造一组 90% 都是 1 的人工标签，让「永远判 1」的评委也拿到 90%，体会为什么标注集要正负均衡。

## 读后自测

1. 用 LLM 当评委，你怎么证明这个评委是可信的？→ 见第 7 节「核心概念 1」
2. 新版本评测分高了 5 个点，你怎么判断是真进步还是噪声？→ 见第 7 节「核心概念 2」
3. 一个要和用户来回说几句的车机 Agent，评测时「用户」从哪来？该评什么？→ 见第 7 节「核心概念 3」

## AI 历史知识点：BIG-bench 超越模仿游戏基准（2022，Google 主导、132 家机构协作）

**是什么**：一个由社区共同出题的大型语言模型基准，论文题为 *Beyond the Imitation Game*。arXiv 版包含 204 个任务，涵盖语言学、数学、常识推理、生物、物理、社会偏见、软件开发等，专挑「当时模型做不到」的题。

**何时由谁提出**：2022 年 6 月 9 日发布在 arXiv，作者 442 人，来自 132 家机构，代码仓库在 google/BIG-bench。评测对象包括 OpenAI 的 GPT 系列、Google 内部的稠密 Transformer 和 Switch 式稀疏 Transformer，并请专家组做了人类基线。

**为什么重要**：它系统记录了模型能力随规模变化的两种形态：靠知识和记忆的任务平滑提升；需要多步推理的任务会在某个规模「突然」跳升，但论文也指出，部分跳升来自评分指标太脆弱。它还在每个任务文件里放了一个**金丝雀字符串（canary GUID）**，方便训练方过滤这些数据，也能通过模型对该字符串的概率检测是否「见过考题」。

**和今天的联系**：今天讲的几个问题它都碰到过：评分指标会制造假象（对应「先怀疑评测」）、考题泄漏会抬高分数（对应「饱和与污染」）、需要人类基线做参照（对应「评委先被评」）。

**原始来源**：[arXiv 2206.04615 论文页（Hugging Face Papers）](https://huggingface.co/papers/2206.04615)、[Google Research 论文页](https://research.google/pubs/beyond-the-imitation-game-quantifying-and-extrapolating-the-capabilities-of-language-models/)、[BIG-bench 金丝雀检测任务说明](https://github.com/google/BIG-bench/blob/main/bigbench/benchmark_tasks/training_on_test_set/README.md)

## 参考区（用时查，不必通读）

### 细节速查

**统计（Anthropic《A statistical approach to model evaluations》）**
- 95% 置信区间 = 均值 ± 1.96 × SEM；SEM = 标准差 / √n。
- 聚类标准误：题目同源（同一段材料出多道题）时按「随机化单位」聚类，流行评测上聚类后的标准误可达朴素值的 3 倍以上。
- 降方差：思维链类评测每题重采样多次取平均；非思维链类可用正确选项的 token 概率当分数。
- 配对比较：前沿模型逐题得分相关系数约 0.3~0.7，相关越高配对收益越大；报告时给出均值差、标准误、置信区间和相关系数。
- 功效分析：给定效应大小、显著性水平、功效，反推需要的题量。

**LLM 评委（Claude 文档 + LangSmith）**
- 评分方式优先级：代码 > LLM > 人工；「量比质重要」：多一些自动评分的题，好过少量人工精评。
- 评委提示要点：详细细则、输出受限（correct / incorrect 或 1~5 分）、开思考先推理后打分、评委模型与被测模型不同。
- LangSmith Align Evals：人工标注 ≥ 20 条、0/1 均衡 → Evaluator Playground → Start Alignment → 一致率 = 评委与人工一致的比例；修改提示词不会自动保存。
- LangSmith 评分类型：Boolean / Categorical / Continuous；输出 schema 的每个顶层键是一条独立反馈。
- Claude Managed Agents 的 outcome：`user.define_outcome` 事件带 `description`、`rubric`（必填）、`max_iterations`（默认 3，最大 20）；评委在独立上下文中运行；结果有 `satisfied` / `needs_revision` / `max_iterations_reached` / `failed` / `interrupted`；需要 `managed-agents-2026-04-01` 头。

**openevals 多轮模拟**
- `run_multiturn_simulation(app, user, max_turns=None, stopping_condition=None, trajectory_evaluators=None, reference_outputs=None, thread_id=None)`，`max_turns` 与 `stopping_condition` 至少给一个。
- `app(msg, *, thread_id)` 返回带 `role`、`content` 的消息；`user` 可以是 `create_llm_simulated_user(system=..., model=..., fixed_responses=...)`、自定义函数 `(trajectory, *, thread_id, turn_counter)`，或固定台词列表。
- 返回 `{"trajectory": [...], "evaluator_results": [...]}`；评分器以 `outputs=`、`reference_outputs=` 关键字参数调用。

**Agent 评测排障（Anthropic《Demystifying evals for AI agents》）**
- 好题标准：两名领域专家独立判定结果一致；有能通过所有评分器的参考解。
- 0% pass@100 多半是题或评分器坏了；评测饱和后转为回归集。
- 每次试验从干净环境开始；共享状态（如能看到之前试验的 git 历史）会虚高分数。
- pass^k 示例：单次 75% 成功率，连续 3 次都成功约 42%。

### 来源说明

- 示例代码中的 Agent 与车主为剧本，未调用真实 Claude（本环境无 API Key）；统计计算与 openevals 模拟器为真实运行，`paired_ci.py` 的数据是带固定种子的模拟数据。
- Claude Academy 课程页本次未能打开；BIG-bench 部分为非 LangChain/Claude 来源。

### 延伸学习

- Claude Academy · Building with the Claude API → Prompt evaluation 部分
- LangSmith 文档 · Improve LLM-as-judge evaluators using human feedback；How to simulate multi-turn interactions

### 参考资料

- [Demystifying evals for AI agents - Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [A statistical approach to model evaluations - Anthropic](https://www.anthropic.com/research/statistical-approach-to-model-evals)
- [Define success criteria and build evaluations - Claude Platform Docs](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- [Define outcomes - Claude Platform Docs](https://platform.claude.com/docs/en/managed-agents/define-outcomes)
- [Improve LLM-as-judge evaluators using human feedback - Docs by LangChain](https://docs.langchain.com/langsmith/improve-judge-evaluator-feedback)
- [How to define an LLM-as-a-judge evaluator - Docs by LangChain](https://docs.langchain.com/langsmith/llm-as-judge)
- [How to simulate multi-turn interactions - Docs by LangChain](https://docs.langchain.com/langsmith/multi-turn-simulation)
