# Day 16 · 多智能体架构（第二轮：派活要写说明书、结果只回引用、并行要防写冲突）（2026-11-13）

## 今日 AI 动态（2026-11-13）

### 趋势观察

- 两家主要发布方都在把 AI 塞进已有的工作场景：Anthropic 让 Claude 进会议、接入工厂的 ERP 和 CAD 系统；Google 把新模型同时推到搜索、Sheets、Android Studio 和 Antigravity。竞争重点正在从「比模型」转向「比入口和系统集成」。
- Gemini 4 Hydrogen 以中档价格拿到和旗舰 Argon 持平的智能指数（Google 自报），还新增了 xhigh 思考档。推测：用思考档位按任务分配算力的中档模型，会是下一阶段价格战的主战场。
- 两家都在强调可控：Claude Meetings 不存音频、默认不进有外部嘉宾的会议；制造业插件规定结果必须人工审核、不直接操作设备。企业级 Agent 的「人在回路」正在变成产品卖点。

### 今日要点

- Anthropic 推出 Claude Meetings Beta：Claude 作为参会者加入 Google Meet，会中答疑、现场出文档和白板。
- Anthropic 推出 Claude for Manufacturing：17 个技能，首批接入 Autodesk Fusion、Infor、Microsoft 365、Oracle NetSuite。
- Google 发布 Gemini 4 Hydrogen：每百万 token 输入 1.5 / 输出 7.5 美元，1M 上下文，主打编程和 Agent 工作流。
- Gemini 4 Hydrogen 同日接入全球 Google 搜索，Google 称这是搜索迄今最大的一次模型升级。

### 分公司动态

#### 1. Anthropic / Claude

**[Introducing Claude Meetings, now in beta](https://claude.com/resources/articles/claude-meetings)**（2026-11-12）
Claude 以可见参会者的身份加入视频会议，记笔记、回答口头或聊天区的提问，并在会上直接生成文档和白板。目前只支持 Google Meet，面向 Team 和 Enterprise 计划；只保存转录、摘要和屏幕描述，不保存音频。Beta 期间纪要和会中问答免费，文档和白板免费到 2027 年 2 月 10 日；启用了 ZDR、HIPAA 或 CMEK 的组织暂不可用。
**为什么重要**：Claude 从「会后总结」走到「会中干活」，开始和会议纪要类产品正面竞争。

**[Introducing Claude for Manufacturing](https://claude.com/resources/articles/claude-for-manufacturing)**（2026-11-12）
面向工厂团队的插件，含 17 个技能，覆盖报价、质量调查、设备故障诊断、替代供应商寻源和生产报告等环节。首批连接器为 Autodesk Fusion、Infor、Microsoft 365 和 Oracle NetSuite。所有结果发给客户或供应商前都要人工审核，插件不直接操作设备。
**为什么重要**：Claude 的行业插件从金融、生命科学延伸到制造业，开始接入 ERP、CAD 这些核心生产系统。

#### 2. Google

**[Introducing Gemini 4 Hydrogen](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-hydrogen/)**（2026-11-12）
Gemini 4 家族中更快、更便宜的型号，主打日常编程和 Agent 工作流。每百万 token 输入 1.5 美元、输出 7.5 美元，整个 1M 上下文同价，最多输出 1M token；思考档位分 low / medium / high（默认）/ xhigh。Google 自报：Artificial Analysis 智能指数 high 档 51（Argon 为 50），xhigh 档 54；Terminal-Bench 4.1 得分 57%（Gemini 3.8 Flash 为 18.2%）。已上线 Gemini API、AI Studio、Android Studio、Antigravity、Gemini 应用和 Sheets。
**为什么重要**：Android Studio 当天就能用，对 Android 开发者来说，这是一个价格低、主打 Agent 编程的新选项。

**[Google Search gets a major upgrade with Gemini 4 Hydrogen](https://blog.google/products-and-platforms/products/search/gemini-4-hydrogen-search-upgrade/)**（2026-11-12）
Hydrogen 面向全球所有用户接入 Google 搜索，能理解更长、更复杂的查询，并行检索相关子话题，答案里加入更多图片、表格和自定义工具。
**为什么重要**：「拆成子话题并行检索」正是多智能体研究系统的做法，现在直接落到了全球最大的搜索入口。

### 动态来源

- https://claude.com/resources/articles/claude-meetings
- https://claude.com/resources/articles/claude-for-manufacturing
- https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-hydrogen/
- https://blog.google/products-and-platforms/products/search/gemini-4-hydrogen-search-upgrade/

---

> 第二轮说明：第一轮 Day 16 讲过「先问要不要多 Agent」、子 Agent 当工具、Handoff、并行的边界。今天只讲进阶：主管给子 Agent 的「派活说明书」怎么写、结果怎么交回才不丢信息也不撑爆上下文、并行写共享状态的冲突、一个子 Agent 失败时整轮怎么办，以及多 Agent 到底贵多少。

## 读前三问

1. 主管派活时只写一句「查一下充电站」会出什么问题？一份合格的派活说明至少要有哪几项？
2. 子 Agent 查到 23 条充电站详情，应该原样交回主管吗？如果不交回，主管怎么用到这些详情？
3. 三个子 Agent 并行，其中一个超时了，整轮任务该怎么处理？并行写同一个状态字段又会出什么错？

## 从一个场景开始

第一轮做的车机主管 Agent 上线了，用户说：「明天开车去杭州，帮我规划充电、天气和住宿。」

主管并行派出三个子 Agent，问题接连出现：

- 主管给充电子 Agent 只写了「查充电站」。它查了上海市区的站，还顺手**预约**了一个桩。
- 充电子 Agent 把 23 条原始结果全部交回主管，主管的上下文一下子多了几千 token，后面的推理开始走神。
- 天气服务超时，抛出的异常让整轮失败，已经查好的充电站和酒店也白查了。
- 改成把结果写进共享状态后，两个子 Agent 同一步写入，图直接报错。

单 Agent 时这些都不是问题。多 Agent 的难点不在「拆」，而在**交接**：派出去说什么、交回来交什么、出了错怎么收场。

## 一句话总结

多 Agent 的质量取决于交接：派活写清目标、格式和边界，结果把详情放在外部、只回摘要和引用，并行写共享状态要定义合并规则，单个子任务失败要降级而不是让整轮失败。

## 先跑起来

一个文件：主管用一个 `task` 工具并行派出三个子 Agent。为了不依赖 API Key、结果可复现，主管用「剧本模型」代替 Claude 做决策，子 Agent 用函数桩代替；工具执行、并行调度、状态合并都是 LangChain / LangGraph 的真实代码。

```python
# car_trip.py —— 主管用一个 task 工具并行派活：说明书要写全、详情落盘只回引用、单个失败不拖垮整轮（langchain 1.5.0）
import operator; from typing import Annotated
from langchain.agents import create_agent, AgentState
from langchain.tools import tool, ToolRuntime
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI, ToolMessage
from langgraph.types import Command

class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self   # 剧本模型，代替 Claude
def model(*replies): return Scripted(messages=iter(replies))
class TripState(AgentState):
    files: Annotated[dict, operator.or_]          # 并行子 Agent 同时写 files，用合并 reducer 防冲突

WORKERS = {  # 子 Agent 注册表：真实项目里每个都是 create_agent(...)
    "charging": lambda brief: "\n".join(f"{c}站|{(i+1)*7}km|空{i%4}桩" for i, c in enumerate("ABCDEFGHIJKLMNOPQRSTUVW")),
    "hotel":    lambda brief: "\n".join(f"杭州酒店{i}|带充电桩|{300+i*20}元" for i in range(12)),
    "weather":  lambda brief: (_ for _ in ()).throw(TimeoutError("天气服务 8s 超时")),
}
@tool
def task(agent_name: str, objective: str, output_format: str, boundaries: str,
         runtime: ToolRuntime) -> Command:
    """把子任务派给子Agent。必须写清目标、返回格式、不许做什么。"""
    brief = f"目标:{objective}\n返回格式:{output_format}\n边界:{boundaries}"   # 子 Agent 只看得到这份说明书
    try:
        detail, path = WORKERS[agent_name](brief), f"/trip/{agent_name}.md"
        msg = f"完成，首行：{detail.splitlines()[0]}；全文见 {path}"           # 主管只拿摘要+路径
        update = {"files": {path: detail}}
    except Exception as e:                                          # 失败也要回一条 ToolMessage
        msg, update = f"失败（{type(e).__name__}: {e}），其他子任务不受影响，可重试或降级", {}
    return Command(update={**update, "messages": [ToolMessage(msg, tool_call_id=runtime.tool_call_id)]})

def call(i, **a): return {"name": "task", "args": a, "id": f"c{i}"}
boss = create_agent(model(
    AI("", tool_calls=[
        call(1, agent_name="charging", objective="沿途每150km内找有空桩的快充站",
             output_format="站名|距离|空桩数，最多5行", boundaries="只查询，不预约不支付"),
        call(2, agent_name="weather", objective="明天沪杭高速沿途天气",
             output_format="一句话，含是否下雨", boundaries="只查询"),
        call(3, agent_name="hotel", objective="杭州带充电桩的酒店",
             output_format="名称|价格，最多3行", boundaries="只查询，不下单")]),
    AI("充电站和酒店已排好（见 /trip/）；天气查询超时，出发前我再查一次。")),
    tools=[task], state_schema=TripState)
res = boss.invoke({"messages": [{"role": "user", "content": "明天开车去杭州，帮我规划充电、天气和住宿"}], "files": {}})
for m in res["messages"][2:]: print(f"{m.type:>4}: {m.text}")
print("主管上下文字数:", sum(len(m.text) for m in res["messages"]), "| files:", {k: len(v) for k, v in res["files"].items()})
```

**运行**：`pip install langchain`（实测 langchain 1.5.0 / langgraph 1.2.21），然后 `python car_trip.py`。

**实测输出**（2026-11-13）：

```
tool: 完成，首行：A站|7km|空0桩；全文见 /trip/charging.md
tool: 失败（TimeoutError: 天气服务 8s 超时），其他子任务不受影响，可重试或降级
tool: 完成，首行：杭州酒店0|带充电桩|300元；全文见 /trip/hotel.md
  ai: 充电站和酒店已排好（见 /trip/）；天气查询超时，出发前我再查一次。
主管上下文字数: 179 | files: {'/trip/charging.md': 283, '/trip/hotel.md': 193}
```

把 `files` 的类型改成普通的 `dict`（去掉 reducer）再跑，实测报错：

```
langgraph.errors.InvalidUpdateError: At key 'files': Can receive only one value per step. Use an Annotated key to handle multiple values.
```

**该看什么**：
- 第 2 行：天气子任务失败了，但它仍然返回一条 ToolMessage，另外两个子任务的结果照常保留，主管最后给出了降级回复。
- 最后一行：主管上下文只有 179 字，充电站全文 283 字留在 `files` 里。规模越大，这个差距越明显。
- 报错那段：两个子 Agent 在同一步写 `files`，没有 reducer，LangGraph 不知道该保留哪个，直接拒绝。

## 核心概念

### 1. 派活说明书：子 Agent 只看得到你写的那几行

- **是什么**：子 Agent 从空白上下文开始，不知道用户原话。主管写给它的任务描述就是它的全部世界。Anthropic 的经验是每份说明都要有：目标、输出格式、工具和信息源的指引、明确的任务边界。
- **没有它会怎样**：Anthropic 在研究系统里发现，说明写得含糊，子 Agent 会互相重复劳动、留下空白，或者把任务理解错。开头场景里「顺手预约」就是缺了边界。
- **对应你熟悉的**：像给后台 Service 发 Intent。extras 里没带的信息，Service 就不知道。

代码里把这几项做成 `task` 工具的**必填参数**，相当于用 schema 逼主管写全。Claude Academy 的子 Agent 课程还强调：子 Agent 的描述会影响主管写出的派活内容，描述里写「必须告诉它具体看哪些文件」，主管就会写具体。

### 2. 结果交回：只回摘要和引用，详情放外面

- **是什么**：子 Agent 把完整产出写到外部（文件系统、共享状态、存储），只给主管返回摘要和一个引用（路径或 ID）。主管需要细节时再去读。
- **没有它会怎样**：要么原样交回，撑爆主管上下文；要么只交一句话，细节在转述中丢失。Anthropic 把后者叫做「传话游戏」。
- **对应你熟悉的**：Activity 之间传大图不放 Intent，而是传一个 `Uri`，对方按需用 ContentResolver 读取。

输出格式最好固定。课程里的建议是在格式里加一栏「遇到的障碍」，把子 Agent 踩过的坑一起交回，主管就不用再踩一次。

### 3. 并行写共享状态：要先定义合并规则

- **是什么**：LangGraph 里，同一步有多个节点写同一个状态字段时，必须用 `Annotated[类型, reducer]` 声明怎么合并，否则报 `InvalidUpdateError`。
- **没有它会怎样**：要么直接报错（像上面那样），要么在别的框架里静默「后写覆盖先写」，结果丢了也没人知道。
- **对应你熟悉的**：多个协程同时改一个 `MutableStateFlow`，要用 `update { }` 原子合并，不能各自 `value =` 覆盖。

注意 `operator.or_` 只是浅合并：两个子 Agent 写同一个路径时，后到的仍然会覆盖。所以要么给每个子 Agent 分配独立的命名空间（示例里用 `/trip/<agent_name>.md`），要么写自定义 reducer。

### 4. 失败隔离与成本：多 Agent 很贵，错误还会累积

- **是什么**：每个子任务的异常都要在它自己的边界里捕获，转成一条「失败 + 原因 + 建议」的结果交给主管决定重试还是降级。长任务要能从检查点恢复，而不是从头重来。
- **没有它会怎样**：一个子任务失败，整轮都失败，已完成的工作全部浪费。Anthropic 指出，Agent 是有状态的，错误会累积，从头重启代价很高，用户体验也差。
- **成本**：Anthropic 的数据是，Agent 的 token 用量约为普通对话的 4 倍，多 Agent 系统约为 15 倍。所以子 Agent 的数量要按任务难度来定：简单查询 1 个子 Agent，对比类问题 2–4 个，复杂研究才用 10 个以上。

## 高频面试题

**1. 主管 Agent 的派活说明应该包含什么？怎么保证主管真的写全？**
- 目标、输出格式、可用工具和信息源、边界（不许做什么、什么时候停）。
- 把这几项做成派活工具的必填参数，用 schema 约束，而不是靠 prompt 提醒。
- 子 Agent 的描述写具体，主管写出的说明也会更具体。
- 子 Agent 只给需要的工具，比如只读调研的子 Agent 不给写入和下单工具。

**2.（追问）子 Agent 的结果怎么交回主管？各有什么取舍？**
- 只回最终文本：最简单，但细节会丢。
- 原样回全部消息：信息全，但主管上下文膨胀，也会混入噪音。
- 详情写入外部存储，只回摘要和引用：兼顾两者，代价是主管要多一步读取，存储要有命名规则。
- LangChain 里可以让工具返回 `Command`，同时更新自定义状态和回一条 ToolMessage。

**3.（场景题）车机主管并行派出充电、天气、酒店三个子 Agent，天气超时了，怎么设计？**
- 每个子任务各自设超时，异常在工具内部捕获，转成带原因的失败结果，不向上抛。
- 主管根据结果决定：重试一次、换备用服务，或者降级回复「天气稍后再查」。
- 已完成的结果写入状态并做检查点，重试时只重跑失败的那一个。
- 并行写同一字段要用 reducer，并给每个子 Agent 独立的命名空间。

**4. 面试官问：「多 Agent 效果更好，为什么不都用多 Agent？」**
- 多 Agent 约用普通对话 15 倍的 token，任务价值要撑得起这个成本。
- 需要共享大量上下文、子任务之间依赖多的场景不适合，比如大多数编码任务。
- LangChain 的测试显示，子 Agent 模式是无状态的，同一个请求再来一次要重走完整流程；有状态的模式（Handoff、Skills）在重复请求时调用次数少 40–50%。
- 先用单 Agent 加好的工具和 prompt，确实遇到工具太多选不准、需要并行、需要隔离上下文时再拆。

## 常见误区

1. **「子 Agent 越多越好，反正是并行的」**：并行省的是时间，不省 token。子 Agent 越多，成本越高，主管汇总的负担也越重。数量应该按任务难度定。
2. **「子 Agent 的异常让它抛出来，框架会处理」**：并行调用里一个工具抛异常，往往整轮失败，其他子任务的成果也一起丢了。异常要在子任务边界内转成结果。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带失败隔离的派活工具**（取自 `car_trip.py`）

```python
# 自定义状态：在消息之外加一个 files 字段
class TripState(AgentState):
    # 并行写入时按字典合并，避免同一步写冲突
    files: Annotated[dict, operator.or_]

# 声明为工具，主管靠函数名和文档决定怎么调用
@tool
# 四个业务参数都必填，runtime 由框架注入
def task(agent_name: str, objective: str, output_format: str, boundaries: str,
         runtime: ToolRuntime) -> Command:
    """把子任务派给子Agent。必须写清目标、返回格式、不许做什么。"""
    # 拼出派活说明书，这是子 Agent 唯一能看到的内容
    brief = f"目标:{objective}\n返回格式:{output_format}\n边界:{boundaries}"
    try:
        # 调用子 Agent 拿到完整结果，并为它分配独立路径
        detail, path = WORKERS[agent_name](brief), f"/trip/{agent_name}.md"
        # 给主管的只有首行摘要和路径
        msg = f"完成，首行：{detail.splitlines()[0]}；全文见 {path}"
        # 完整结果写进共享状态
        update = {"files": {path: detail}}
    # 捕获子 Agent 的任何异常，不让它拖垮整轮
    except Exception as e:
        # 失败也给出原因和建议，状态不更新
        msg, update = f"失败（{type(e).__name__}: {e}），其他子任务不受影响，可重试或降级", {}
    # 同时更新状态并回一条对应这次调用的工具消息
    return Command(update={**update, "messages": [ToolMessage(msg, tool_call_id=runtime.tool_call_id)]})
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `Annotated[dict, operator.or_]`：第二个参数是函数本身，不是 `operator.or_()`。
- `ToolMessage` 必须带 `tool_call_id=runtime.tool_call_id`，否则主管无法把结果对应到哪次调用。
- 失败分支的 `update` 是空字典 `{}`，不是 `None`，否则 `**update` 会报错。

**昨日复习（只做第③步）**：第二轮 Day 15 的「推送接收端的三道检查」（`webhook_verify.py`）。只看你自己写的注释重写，重点核对：用 `hmac.compare_digest` 比较签名；先验签再看时间戳；重复通知回 200。

## 动手练习（30 分钟）

1. 给 `task` 加一个可选参数 `timeout_s: int = 5`，在 `WORKERS` 里加一个「慢」子 Agent（`time.sleep`），用 `concurrent.futures` 实现超时，超时走失败分支。
2. 让 `charging` 和 `hotel` 都写同一个路径 `/trip/result.md`，跑一下，看 `operator.or_` 下谁的结果被保留；再写一个自定义 reducer：同路径时把两份内容拼接起来。
3. 把主管的剧本改成：第一轮收到天气失败后，再发一次 `task(agent_name="weather", ...)`。思考：在真实场景里，主管怎么知道这次该重试而不是降级？

## 读后自测

1. 主管派活时只写一句「查一下充电站」会出什么问题？一份合格的派活说明至少要有哪几项？——见第 1 节和 Q1
2. 子 Agent 查到 23 条充电站详情，应该原样交回主管吗？如果不交回，主管怎么用到这些详情？——见第 2 节和 Q2
3. 三个子 Agent 并行，其中一个超时了，整轮任务该怎么处理？并行写同一个状态字段又会出什么错？——见第 3、4 节和 Q3

## AI 历史知识点：CICERO 外交博弈 AI（2022，Meta AI）

**是什么**：一个会玩桌游《外交》（Diplomacy）的 AI。这个游戏 7 人对局，胜负主要靠私下谈判、结盟和背盟，CICERO 要用自然语言和人类玩家协商，再据此决定行动。

**何时由谁提出**：Meta AI 于 2022 年 11 月 22 日发布，论文由 Meta FAIR Diplomacy 团队发表在《Science》上。

**为什么重要**：Meta 称它是第一个在《外交》中达到人类水平的 AI。在 webDiplomacy.net 上，它的平均得分是人类玩家的两倍多，在玩过不止一局的参与者中排名前 10%。它由两部分组成：一个 27 亿参数的可控对话模型，和一个预测其他玩家行动、再规划自己行动的策略推理引擎；生成消息前先定好「意图」，再过滤掉与计划不一致的消息。

**和今天的趋势有什么联系**：CICERO 是「语言模型 + 规划」在多方协作场景里的早期成功案例。今天的多 Agent 系统里，主管理解各方意图、协调分工、让说的和做的保持一致，面对的是同一类问题。

**原始来源**：
- [CICERO: An AI agent that negotiates, persuades, and cooperates with people - Meta AI](https://ai.meta.com/blog/cicero-ai-negotiates-persuades-and-cooperates-with-people/)
- [Human-level play in the game of Diplomacy by combining language models with strategic reasoning（论文索引页）](https://www.mit.edu/~gfarina/2022/cicero)

---

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| LangChain 五种模式 | Subagents（主管把子 Agent 当工具调用）、Handoffs（工具调用改状态，切换当前 Agent）、Skills（单 Agent 按需加载专业提示和知识）、Router（先分类再分发，最后汇总）、Custom workflow（LangGraph 自定义流程） |
| LangChain 模式对比 | 单次请求调用次数：Subagents 4，Handoffs / Skills / Router 3；重复请求（第二轮/合计）：Subagents 4/8、Handoffs 2/5、Skills 2/5、Router 3/6；多领域任务调用/token：Subagents 5/~9K、Handoffs 7+/~14K+、Skills 3/~15K、Router 5/~9K |
| 子 Agent 输入 | 只传查询 = 隔离；传父级消息历史 = 带上下文。Deep Agents 叫 isolated（默认）和 fork，fork 会带上父级完整历史，含工具调用和 system prompt |
| 子 Agent 输出 | 返回最终文本；或返回 `Command(update={...自定义状态..., "messages": [ToolMessage(content, tool_call_id=...)]})` |
| 派发方式 | 每个子 Agent 一个工具；或一个 `task(agent_name, description)` 工具查注册表派发 |
| 同步/异步 | 默认同步，主管等子 Agent 跑完；异步是三工具模式：启动拿 job ID、查状态、取结果 |
| 并行 | 主管一轮可调用多个子 Agent；工具内部调用的子 Agent 无法被静态发现，`get_state(subgraphs=True)` 拿不到其状态 |
| 并行写冲突 | 同一步多值写同一字段报 `InvalidUpdateError`（INVALID_CONCURRENT_GRAPH_UPDATE），用 `Annotated[类型, reducer]` 解决 |
| Anthropic 研究系统数据 | Agent 约 4 倍对话 token，多 Agent 约 15 倍；BrowseComp 中 token 用量单独解释 80% 的表现差异（加上工具调用次数和模型选择共 95%）；Opus 4 主管 + Sonnet 4 子 Agent 比单 Agent Opus 4 高 90.2%（内部评测）；主管并行起 3–5 个子 Agent、子 Agent 并行 3+ 工具，复杂查询耗时最多减少 90%；工具测试 Agent 改写工具描述后任务耗时减少 40% |
| 投入分级 | 简单事实查询 1 个 Agent、3–10 次工具调用；对比类 2–4 个子 Agent、各 10–15 次；复杂研究 10 个以上 |
| 生产化 | 错误会累积，要能从出错处恢复（检查点 + 重试）；全量追踪但不读对话内容；彩虹部署避免打断运行中的 Agent；同步执行是瓶颈，异步会带来结果协调、状态一致性和错误传播问题 |
| 评测 | 先用约 20 条代表性查询；LLM 评审按事实准确、引用准确、完整性、来源质量、工具效率打 0–1 分和通过/不通过；人工评测能发现偏好 SEO 内容农场这类问题 |
| 不适合多 Agent | 需要所有 Agent 共享上下文、Agent 之间依赖多；大多数编码任务；实时协调；价值撑不起 token 成本 |
| Claude Managed Agents | `multiagent: {type: "multiagent_20261001"}`（取代旧的 `coordinator`），beta 头 `managed-agents-2026-04-01`；三种机制：subagents（只能一层，最多 25 个子线程）、workflows（后台分阶段跑多个 Agent，结果程序化传递）、advisor（主线程中途咨询另一个模型，默认关闭）；所有 Agent 共享沙箱、文件系统和凭证，但各自上下文隔离；预定义和内联 Agent 列表各最多 20 个；建议设置会话预算 |
| 子 Agent 设计（Academy） | 定义输出格式是最重要的改进，给子 Agent 天然的停止点；输出模板加「遇到的障碍」；描述会影响主管写出的输入；按职责限制工具（只读调研只给 Glob / Grep / Read） |

### 来源说明

- 示例里主管用剧本模型、子 Agent 用函数桩，没有调用真实模型；工具执行、并行调度和状态合并是 LangChain / LangGraph 的真实行为。
- 历史知识点引用 Meta AI 官方博客（非 LangChain/Claude 来源）；《Science》论文原文页未打开，作者信息以 Meta 博客和论文索引页为准。

### 延伸学习

- Claude Academy：Introduction to subagents 课程，重点看「[Designing effective subagents](https://academy.claude.com/courses/introduction-to-subagents/designing-effective-subagents)」一节。

### 参考资料

- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Multiagent orchestration - Claude Platform Docs](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)
- [Designing effective subagents - Claude Academy](https://academy.claude.com/courses/introduction-to-subagents/designing-effective-subagents)
- [Multi-agent - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent)
- [Subagents - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents)
