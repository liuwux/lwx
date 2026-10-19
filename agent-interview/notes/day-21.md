# Day 21 · 评测（2026-10-19）

## 读前三问

1. 评测一个 Agent，评「结果」和评「轨迹」有什么区别？各自会漏掉什么？
2. LLM-as-judge 怎么用才可靠？它有哪些坑？
3. 改了一句提示词，怎么在上线前证明 Agent 没有变差？

## 从一个场景开始

你负责的车机语音 Agent 已经能调空调了。产品经理提了个需求：「回复太啰嗦，改简洁点。」你改了 system prompt，自己在车上试了三句话，感觉都挺好，就发版了。

两天后投诉来了：车主只是问「车里几度」，Agent 却顺手把全车空调调成了 24 度；说「主驾太冷」，副驾也被一起调热了。

问题不在提示词改得好不好，而在于**你没法证明它没变差**。手动试三句话，覆盖不了「只问不做」这种反例，也看不出「调对了温度、调错了区域」这种细节。

Android 上你不会不跑单测就发版。Agent 也一样，只不过它的输出不确定，测试要换一套写法：准备一批固定用例，每次改动都跑一遍，用代码或另一个模型给结果打分。这就是今天的主题：评测（eval）。

## 一句话总结

评测就是给 Agent 写测试：从真实失败中整理一批用例，优先用代码检查最终结果，主观质量交给有评分细则的模型评判，每次改动都跑回归，并多跑几次看稳定性。

## 先跑起来

一个两条用例的评测集，对比 v1（原版）和 v2（改了提示词的新版）。每条用例打两种分：**结果**（车辆状态对不对）和**轨迹**（该调的工具有没有调）。无 API Key，模型决策用剧本代替；Agent 循环、工具执行和轨迹评分器都是真实库代码。

```python
# car_eval.py —— 车机 Agent 回归评测：结果评分 + 轨迹评分（langchain 1.4.6 / agentevals 0.0.9）
from langchain.agents import create_agent
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI
from langchain_core.tools import tool
from agentevals.trajectory.match import create_trajectory_match_evaluator

car = {}  # 模拟车辆状态：「结果评分」看的就是它

@tool
def set_ac(zone: str, temp: int) -> str:
    """设置某个区域的空调温度"""
    car[zone] = temp; return "ok"

@tool
def get_cabin_temp() -> str:
    """读取车内温度"""
    return "12度"

def tc(name, **args): return AI("", tool_calls=[{"name": name, "args": args, "id": name}])
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self

# 两个 Agent 版本的「模型决策」（无 API Key，用剧本代替 Claude）；v2 = 改了提示词后的新版本
SCRIPTS = {"v1": {"主驾太冷了": [tc("get_cabin_temp"), tc("set_ac", zone="driver", temp=26), AI("已调到26度")],
                  "车里几度": [tc("get_cabin_temp"), AI("车内12度")]},
           "v2": {"主驾太冷了": [tc("set_ac", zone="all", temp=26), AI("已调到26度")],
                  "车里几度": [tc("get_cabin_temp"), tc("set_ac", zone="all", temp=24), AI("12度，已调到24度")]}}

# 评测集：输入 + 期望的最终车辆状态 + 参考轨迹（只列必须出现的工具调用）
DATASET = [{"input": "主驾太冷了", "expect": {"driver": 26}, "ref": [tc("set_ac", zone="driver", temp=26)]},
           {"input": "车里几度", "expect": {}, "ref": [tc("get_cabin_temp")]}]  # 反例：只问不该动空调

traj_eval = create_trajectory_match_evaluator(trajectory_match_mode="superset")  # 允许多调，不许漏调

for ver, script in SCRIPTS.items():
    for case in DATASET:
        car.clear()
        agent = create_agent(Scripted(messages=iter(script[case["input"]])), tools=[set_ac, get_cabin_temp])
        msgs = agent.invoke({"messages": [{"role": "user", "content": case["input"]}]})["messages"]
        outcome_ok = car == case["expect"]                                # 1) 结果：车的真实状态对不对
        traj_ok = traj_eval(outputs=msgs, reference_outputs=msgs[:1] + case["ref"])["score"]  # 2) 过程
        print(f"{ver} | {case['input']:<6} | 结果={'✔' if outcome_ok else '✘'} 轨迹={'✔' if traj_ok else '✘'} | car={car}")
```

**运行**：`pip install langchain agentevals && python car_eval.py`

**实测输出**（2026-10-19，Python 3.13）：

```
v1 | 主驾太冷了  | 结果=✔ 轨迹=✔ | car={'driver': 26}
v1 | 车里几度   | 结果=✔ 轨迹=✔ | car={}
v2 | 主驾太冷了  | 结果=✘ 轨迹=✘ | car={'all': 26}
v2 | 车里几度   | 结果=✘ 轨迹=✔ | car={'all': 24}
```

**该看什么**：
- v2 两条全挂结果分：这就是一次被评测集**拦下来的回归**，不用等车主投诉。
- 最后一行：轨迹分 ✔ 但结果分 ✘。`superset` 模式允许「多调工具」，多出来的那次 `set_ac` 没被轨迹评分发现，只有检查车辆状态才抓到。**结果评分是主力，轨迹评分是补充。**
- 第二条用例是「反例」（不该做事的场景），正是它抓住了「只问却动手」的 bug。

## 核心概念

### 1. 评测集：从真实失败里长出来

- **是什么**：一组固定的测试用例（task）。每条包括输入、成功标准，最好还有一份参考答案。同一条用例跑一次叫一次试验（trial）。
- **没有它会怎样**：每次改动只能凭感觉试几句，问题全靠线上用户发现。
- **对应你熟悉的**：就是 JUnit 用例集 + 线上崩溃转化成的回归用例。

怎么起步：Anthropic 建议**先用 20~50 条**，来源是手工测过的场景、bug 单、客服反馈。两条原则：用例要明确到「两个专家判定结果一致」；**正例和反例都要有**（该调空调时调，只问温度时别调）。

### 2. 三类评分器：代码 > 模型 > 人

| 评分器 | 适合 | 缺点 |
|---|---|---|
| 代码 | 状态检查、精确匹配、工具调用检查 | 死板，合理的不同写法也会判错 |
| 模型（LLM-as-judge） | 语气、是否简洁、回答是否有帮助 | 不确定、要花钱、要和人工对齐 |
| 人 | 校准模型评委、抽查 | 慢、贵 |

- **是什么**：LLM-as-judge 的大白话：再请一个模型当「阅卷老师」，按评分细则给答案打分。
- **没有它会怎样**：「回复够不够简洁」这种问题，代码根本判不了。
- 用好它的三点：评分细则写具体（「超过 30 字算不合格」比「要简洁」好）；输出用「通过/不通过」或 1~5 分，并先写理由再给分；留一个「无法判断」的出口，减少瞎判。官方还建议**评委用不同于被测模型的模型**。

### 3. 评结果，还是评轨迹

- **是什么**：结果（outcome）是任务结束后环境的真实状态，比如车里空调实际是多少度，而不是 Agent 说了什么。轨迹（trajectory）是过程中的全部消息和工具调用。
- **没有它会怎样**：只看 Agent 回复「已调好」，它说谎你也不知道；只死抠工具顺序，Agent 换一种合理做法就被判错。
- **对应你熟悉的**：结果评分像 Espresso 断言界面最终状态；轨迹评分像 Mockito `verify()` 某个方法被调用过。

Anthropic 的建议是**评结果，不要死抠路径**。轨迹评分用来补充：检查「必须调用的工具没漏」（`superset`）、「不该调的危险工具没调」（`subset`）。

### 4. 能力评测 vs 回归评测，pass@k vs pass^k

- **能力评测**：难题集，问「它能做到什么」，通过率从低处起步，用来指导改进。
- **回归评测**：已经会做的题，通过率应接近 100%，每次改动都跑，用来防退化。能力题做熟了就「毕业」进回归集。
- **为什么要跑多次**：模型输出有随机性，一次通过不代表稳定。
  - **pass@k**：k 次里至少成功一次的概率，k 越大越高。适合「有一次成功就行」，如写代码。
  - **pass^k**：k 次全部成功的概率，k 越大越低。适合面向用户的场景：单次 75% 的成功率，连跑 3 次全对只有约 42%。

车机控制这种场景看的是 pass^k：车主不会接受「十次里有两次调错区域」。

## 高频面试题

**1. 你怎么评测一个 Agent？**
- 从真实失败和手工测试中整理 20~50 条用例，正例反例都要有。
- 优先代码检查最终状态，主观质量用 LLM-as-judge 加评分细则。
- 必要时检查轨迹：关键工具不漏调、危险工具不乱调。
- 每条跑多次，看 pass^k；接入 CI 做回归。

**2.（追问）LLM 当评委不可靠怎么办？**
- 细则写具体，输出离散分数，先给理由再给分。
- 每个维度单独一个评委，留「无法判断」选项。
- 用一批人工标注样本校准，看评委与人的一致率。
- 评委用与被测不同的模型，定期人工抽查记录。

**3.（追问）pass@k 和 pass^k 分别什么时候用？**
- pass@k：至少一次成功，适合可以重试、有人挑结果的场景。
- pass^k：每次都成功，适合面向用户、要求稳定的场景。
- 同一 Agent，k 增大时前者上升、后者下降。
- 车机、客服这种场景应看 pass^k。

**4.（场景题）车机 Agent 改了提示词，如何保证上线不出事？**
- 先跑回归集，按版本对比通过率，出现任何回归就阻止发版。
- 每条多跑几次，比较 pass^k，排除偶然通过。
- 用例必须包含反例（只问不做、模糊指令、越界温度）。
- 上线后把线上失败的对话补进评测集，形成闭环。

## 常见误区

1. **「轨迹完全一致才算对」**：严格比对工具顺序很脆弱，Agent 换一种合理做法就被判错。先评结果，轨迹只查关键约束。
2. **「通过率 100% 说明 Agent 很好」**：可能是题太简单（饱和），也可能是评分器有漏洞。要定期读失败和通过的完整记录，并补充更难的题。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：评测主循环 —— 逐版本、逐用例，打结果分和轨迹分**（取自 `car_eval.py`）

```python
# 评测集：每条有输入、期望的最终车辆状态、参考轨迹
DATASET = [{"input": "主驾太冷了", "expect": {"driver": 26}, "ref": [tc("set_ac", zone="driver", temp=26)]},
           # 反例：只问温度，不该动空调
           {"input": "车里几度", "expect": {}, "ref": [tc("get_cabin_temp")]}]

# 轨迹评分器：superset 表示可以多调，但参考里的调用不能少
traj_eval = create_trajectory_match_evaluator(trajectory_match_mode="superset")

# 外层遍历每个 Agent 版本
for ver, script in SCRIPTS.items():
    # 内层遍历每条用例
    for case in DATASET:
        # 每条用例开始前清空车辆状态，避免用例之间互相影响
        car.clear()
        # 用该版本、该用例的剧本建一个 Agent，挂上两个工具
        agent = create_agent(Scripted(messages=iter(script[case["input"]])), tools=[set_ac, get_cabin_temp])
        # 跑一次，拿到完整消息列表（即轨迹）
        msgs = agent.invoke({"messages": [{"role": "user", "content": case["input"]}]})["messages"]
        # 结果分：车辆最终状态是否等于期望
        outcome_ok = car == case["expect"]
        # 轨迹分：参考轨迹前面补上用户消息，再与实际轨迹比对
        traj_ok = traj_eval(outputs=msgs, reference_outputs=msgs[:1] + case["ref"])["score"]
        # 打印一行：版本 | 输入 | 两个分数 | 车辆状态
        print(f"{ver} | {case['input']} | 结果={outcome_ok} 轨迹={traj_ok} | car={car}")
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 参数名是 `trajectory_match_mode`，评分器调用用关键字 `outputs=` 和 `reference_outputs=`，结果取 `["score"]`。
- 每条用例前要 `car.clear()`，否则上一条的状态会让下一条误判（官方说的「试验隔离」）。
- 剧本要包一层 `iter(...)` 再传给 `messages`。

**昨日复习（只做第③步）**：Day 20 的「定义 schema → 挂到 Agent → 取出结构化结果」。只看你自己写的注释重写 `ACCommand`、`create_agent(..., response_format=ToolStrategy(...))` 和取结果那几行，重点核对：范围参数是 `ge`/`le`，结果 key 是 `structured_response`。

## 动手练习（30 分钟）

在 `car_eval.py` 上改：

1. 加一条用例「副驾有点热」，期望 `{"passenger": 22}`，给 v1、v2 各写一份剧本（v2 写成调错区域）。
2. 把轨迹模式改成 `"strict"` 再跑，看哪些原来通过的用例变成 ✘，用一句话写下原因。
3. 在循环外统计每个版本的「结果通过率」，并加一条规则：v2 通过率低于 v1 时打印「阻止发版」。

## 读后自测

1. 评结果和评轨迹有什么区别？各自会漏掉什么？→ 见「先跑起来」的「该看什么」、核心概念 3
2. LLM-as-judge 怎么用才可靠？有哪些坑？→ 见核心概念 2、面试题 2
3. 改了一句提示词，怎么在上线前证明 Agent 没有变差？→ 见核心概念 4、面试题 4

---

## 参考区（用时查，不必通读）

### 细节速查

**Anthropic 术语**（Demystifying evals for AI agents）
- task（用例）、trial（一次试验）、grader（评分器）、transcript / trace / trajectory（完整记录）、outcome（环境最终状态）、eval harness（跑评测的基础设施）、agent harness（让模型成为 Agent 的脚手架）。
- 评分器类型：代码（字符串匹配、单测、静态分析、状态检查、工具调用检查）、模型（评分细则、自然语言断言、两两对比、多评委共识）、人（专家、众包、抽查、A/B）。
- 技巧：部分得分；每个维度一个评委；给评委「Unknown」出口；试验之间环境隔离；评分器要难以钻空子；关注饱和；多跑 0% 通过率的题通常说明题目坏了。

**Claude 文档 · 成功标准与评测**
- 成功标准：Specific、Measurable、Achievable、Relevant；常见维度：任务准确度、一致性、相关性、语气、隐私、上下文利用、延迟、价格。
- 方法示例：精确匹配、余弦相似度（一致性）、ROUGE-L（摘要）、LLM Likert 1~5 分、LLM 二分类、LLM 序数打分。
- 原则：贴近真实分布、覆盖边界情况、尽量自动化、**量大优于精而少**、可用 Claude 扩充用例。

**Claude Academy · Model based grading**：评委返回 JSON：`strengths`、`weaknesses`、`reasoning`、`score`(1~10)；要求先列优缺点是为了避免评委总给中间分。

**agentevals（Python）**
- `create_trajectory_match_evaluator(trajectory_match_mode=...)`：`strict`（同顺序同调用）/ `unordered`（任意顺序）/ `subset`（只能调参考里有的）/ `superset`（至少调参考里的）。
- `tool_args_match_mode`：`exact`（默认）/ `ignore` / `subset` / `superset`；`tool_args_match_overrides`：按工具名设模式、字段列表或比较函数。
- `create_trajectory_llm_as_judge(model=..., prompt=TRAJECTORY_ACCURACY_PROMPT | TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE)`，返回 `{key, score, comment}`。
- 异步版：`create_async_trajectory_match_evaluator`、`create_async_trajectory_llm_as_judge`；LangGraph：`create_graph_trajectory_llm_as_judge`、`graph_trajectory_strict_match`、`extract_langgraph_trajectory_from_thread`。
- 输入可以是 LangChain 消息对象，也可以是 OpenAI 格式 dict。

**LangSmith**
- 离线评测（开发期、固定数据集、对比版本）vs 在线评测（生产流量、采样、告警）。
- 概念：dataset → example；experiment = 某版本在某数据集上的一次运行；评分器：人工 / 代码 / LLM-as-judge / pairwise。
- pytest：`@pytest.mark.langsmith`，`from langsmith import testing as t`，`t.log_inputs / log_outputs / log_reference_outputs`，`pytest file.py --langsmith-output`；或 `Client().evaluate(target, data="数据集名", evaluators=[...])`。
- 环境变量：`LANGSMITH_API_KEY`、`LANGSMITH_TRACING=true`。

### 来源说明

- 示例代码的模型决策是剧本，未调用真实 Claude（本环境无 API Key）；LLM-as-judge 未实测。
- Claude Academy 课程正文需登录，参考的是课程页面公开摘要。

### 延伸学习

- Claude Academy · Building with the Claude API → Prompt evaluation（A typical eval workflow、Generating test datasets、Model based grading、Code based grading）
- LangChain 文档 · Agent Evals；LangSmith · Trajectory evals

### 参考资料

- [Demystifying evals for AI agents - Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Define success criteria and build evaluations - Claude Platform Docs](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- [Model based grading - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api/model-based-grading)
- [Agent Evals - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/test/evals)
- [How to evaluate your agent with trajectory evaluations - Docs by LangChain](https://docs.langchain.com/langsmith/trajectory-evals)
- [LangSmith Evaluation - Docs by LangChain](https://docs.langchain.com/langsmith/evaluation)
- [langchain-ai/agentevals - GitHub](https://github.com/langchain-ai/agentevals)
