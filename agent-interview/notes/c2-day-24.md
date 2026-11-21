# Day 24 · 护栏与人在回路（第二轮：审批绑定参数、会过期、执行前重查，以及恢复与补偿）（2026-11-21）

## 今日 AI 动态（2026-11-21）

### 趋势观察

过去 24 小时没有新模型发布。最大的消息是资本层面的：Anthropic 公布 IPO 最终募资规模，几家主要的芯片和云伙伴同时入股。OpenAI 这边继续用专门的评测来推进「AI for Science」，评测结果显示模型「读懂化学」已经很强，「决定下一步该做哪个实验」仍然偏弱。推测：这种「理解强、决策弱」的差距，正是 Agent 里还需要人在回路的原因之一，和今天的主题直接相关。

### 今日要点

- Anthropic 完成 IPO 交割，含同步私募在内总募资约 1195 亿美元。
- OpenAI 发布药物化学评测 MedChemBench：GPT-6.2 Astra 得分 84.5%，但在「挑选化合物」和数值预测这两类题上只拿到约一半分数。
- Qoder 发布 0.5.17 维护版本，修复插件和 Windows 启动相关问题。

### 分公司动态

#### 1. Anthropic

- **[Anthropic announces closing of initial public offering…](https://www.anthropic.com/news/ipo-close)**（2026-11-20）
  Anthropic 以每股 83 美元发行约 11.27 亿股 A 类股，承销商的超额配售权已全部行使，IPO 募资总额约 935 亿美元。同时以私募方式向 AMD、Broadcom、Google、Micron、Microsoft、NVIDIA、TSMC Partners 出售 N 类股，募资约 260 亿美元，两项合计约 1195 亿美元。股票代码 ANT，在纳斯达克全球精选市场交易，11 月 19 日开始上市交易。
  **为什么重要**：这是公司官方确认的最终募资规模。主要的算力和云伙伴同时成为股东，意味着它们和 Anthropic 的绑定更深了。

#### 2. OpenAI

- **[MedChemBench: Benchmarking AI for medicinal chemistry](https://openai.com/index/introducing-medchembench/)**（2026-11-20，日期以 [OpenAI Science 栏目](https://openai.com/news/science/)列表为准）
  这是一个新评测：99 道题，取材自 117 篇论文和专利，涉及 761 个分子结构。题型分四类：结构还原、科学推理、化合物排序与挑选、性质预测。在 xhigh 推理档位下，GPT-6.2 Astra 得分 84.5%，GPT-5.6 Sol 只有 58.2%。Astra 的结构类题目得分 95.4%，挑选和数值预测两类只拿到约一半分数。
  **为什么重要**：评测把「理解」和「决策」分开打分，能更清楚地看出模型离独立做科研决策还差多远。

#### 3. 阿里巴巴 / Qoder

- **[Qoder 0.5.17「Everyday Polish」](https://changelog.qoder.com/app)**（2026-11-20）
  这是一个维护版本：插件固定引导会跟随界面语言和主题；修复了几个问题，包括市场插件安装后找不到入口、插件介绍重复弹出、Windows 单用户安装快速更新后无法启动，以及部分模型请求失败时只显示笼统报错。
  **为什么重要**：时间范围内阿里方面唯一的官方产品更新，说明 Qoder 的插件生态正在打磨使用体验。

### 动态来源

- https://www.anthropic.com/news/ipo-close
- https://openai.com/index/introducing-medchembench/
- https://openai.com/news/science/
- https://changelog.qoder.com/app

---

## 读前三问

1. 车主已经点了「同意」，为什么执行前还要再检查一次？要检查哪几样东西？
2. 在 LangGraph 里，把审批过期时间写在调用 `interrupt()` 的那个节点里，为什么会出问题？
3. 一个任务里同时有可撤回和不可撤回的步骤，怎么排顺序？中途失败了怎么办？

## 从一个场景开始

第一轮我们给「开车窗」加了条件审批：车速低于 60 km/h 时，Agent 想把车窗全开，就弹窗问车主。

上线一周后出了一个工单。车主在小区里说「把车窗全打开」，弹窗出来时他正在倒车，没顾上点。十分钟后他上了高速，看到屏幕上还挂着那个弹窗，顺手点了「同意」，车速 100 km/h 时四个窗全开了。

审批逻辑本身没错：发起请求时车速只有 20 km/h，满足条件，车主也确实点了同意。问题出在**中间隔的十分钟**。LangGraph 的中断会无限期等下去，等回来时，人同意的还是十分钟前那个世界里的请求。

第二轮要补的就是这段空档：**审批不是终点，从同意到执行之间也要设防。**

## 一句话总结

审批只证明「人在某个时刻同意了某个具体动作」。所以审批要绑定参数、设有效期，执行前按当时的状况再查一遍；恢复时别让节点重跑出副作用；不可逆的步骤放最后，前面的步骤要备好补偿。

## 先跑起来

一个三节点的图：`propose` 生成请求，`approve` 中断等人，`execute` 放行前做三道复核。用三种「车主何时点同意」来对比。不需要 API Key，图、中断、checkpointer 都是真实的 LangGraph 代码。

```python
# approval_guard.py —— 审批之后再校验一次：审批绑定参数、会过期、执行前重查车况（langgraph 1.2）
import hashlib, json
from typing import TypedDict
from langgraph.graph import StateGraph, START
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command

CAR = {"kmh": 20, "now": 0}       # 模拟车况和时钟（秒）
TTL = 60                          # 审批有效期

def digest(action): return hashlib.sha256(json.dumps(action, sort_keys=True).encode()).hexdigest()[:8]

class S(TypedDict, total=False):
    request: dict; approval: dict; result: str

def propose(s):   # 时间戳、摘要都在中断之前、另一个节点里算好，恢复时不会被重算
    action = {"tool": "open_window", "args": {"percent": 100}}
    return {"request": {"action": action, "digest": digest(action), "expires_at": CAR["now"] + TTL}}

def approve(s):   # 这个节点只做一件事：把请求原样交给人
    return {"approval": interrupt(s["request"])}

def execute(s):   # 放行前三道复核：人同意的是这份参数吗？过期没？车况还允许吗？
    req, ok = s["request"], s["approval"]
    if not ok.get("approved"): return {"result": "车主拒绝"}
    if ok.get("digest") != digest(req["action"]): return {"result": "拒绝：参数与审批时不一致"}
    if CAR["now"] > req["expires_at"]: return {"result": "拒绝：审批已过期，请重新确认"}
    if req["action"]["args"]["percent"] > 30 and CAR["kmh"] > 60:
        return {"result": f"拒绝：当前车速 {CAR['kmh']} km/h，全开车窗不安全"}
    return {"result": f"已执行 open_window {req['action']['args']}"}

g = StateGraph(S).add_sequence([propose, approve, execute])     # 三个节点顺序相连
app = g.add_edge(START, "propose").compile(checkpointer=InMemorySaver())

def run(tid, wait, kmh_after):
    CAR.update(kmh=20, now=0)
    cfg = {"configurable": {"thread_id": tid}}
    req = app.invoke({}, cfg)["__interrupt__"][0].value
    CAR.update(now=wait, kmh=kmh_after)             # 车主看弹窗的这段时间里，世界变了
    out = app.invoke(Command(resume={"approved": True, "digest": req["digest"]}), cfg)
    print(f"{tid}: 弹窗={req['digest']} 等了{wait}s 车速{kmh_after} -> {out['result']}")

run("A-马上点", 5, 20)
run("B-上了高速才点", 20, 90)
run("C-两分钟后才点", 120, 20)
```

**运行**：`pip install langgraph && python approval_guard.py`

**实测输出**（2026-11-21，langgraph 1.2.24）：

```
A-马上点: 弹窗=30605f62 等了5s 车速20 -> 已执行 open_window {'percent': 100}
B-上了高速才点: 弹窗=30605f62 等了20s 车速90 -> 拒绝：当前车速 90 km/h，全开车窗不安全
C-两分钟后才点: 弹窗=30605f62 等了120s 车速20 -> 拒绝：审批已过期，请重新确认
```

**该看什么**：
- 三次的弹窗摘要都是 `30605f62`，车主点的都是「同意」，结果却不一样。决定放不放行的是**执行那一刻**的条件。
- B 没过期，但车况变了；C 车况没变，但审批过期了。两道检查各拦住一种情况，缺一不可。
- 恢复时传回的 `digest` 是车主在弹窗上看到的那个。执行前比对它，保证「执行的就是人看到的」。

## 核心概念

### 1. 审批要绑定参数：执行的必须是人看到的

- **是什么**：审批记录里要带上「被批准的是哪份参数」，执行前比对。今天用参数的哈希摘要做比对。
- **没有它会怎样**：从弹窗到执行之间，参数可能被改：模型重新规划、别的中间件改写、并发请求串号。人批的是 30%，执行的是 100%。
- **对应你熟悉的**：支付 SDK 的签名：订单金额参与签名，中途被改，验签就过不去。

官方的做法不完全一样，但思路一致。Claude Agent SDK 里，`canUseTool` 返回什么参数，工具就**只按这份参数**执行；如果你改了参数，Claude 能看到结果，但不会被告知参数被改过。Claude Code 的 auto mode 也规定，用户的一次授权「只覆盖一个动作」，而且授权消息要说出具体是哪个动作、危险在哪（比如 force push 推到哪个分支）。

### 2. 审批会过期，执行前要重查（TOCTOU）

- **是什么**：TOCTOU 是 time-of-check to time-of-use，意思是「检查时」和「使用时」之间状态变了。审批是一次检查，执行是一次使用。
- **没有它会怎样**：LangGraph 文档写明，中断后图会「无限期等待，直到你恢复它」，框架本身不提供过期机制。不自己加有效期，十分钟前的弹窗照样能放行。
- **对应你熟悉的**：`checkSelfPermission` 不能只在启动时查一次。用户可能在设置里撤销了权限，每次真正用之前都要再查。

两条落地规则：
- **超时默认拒绝**。Claude Agent SDK 的 PreToolUse hook 默认 600 秒超时，超时后这次工具调用直接被阻止，不会因为没人回复就放行。
- **执行前重查实时条件**（车速、余额、文件是否还在），不要信任审批时快照下来的值。

### 3. 恢复语义：节点会重跑，中断前算的东西都会变

- **是什么**：恢复时，调用 `interrupt()` 的节点会从第一行重新执行。
- **没有它会怎样**：如果把 `expires_at = now() + 60` 写在 `approve` 节点里，恢复时它会按**恢复那一刻**的时间重新算一遍，审批永远不会过期。今天的代码把时间戳和摘要放在前一个节点 `propose` 里，它的结果已经存进 checkpoint，恢复时不会重跑。
- **对应你熟悉的**：Activity 重建时 `onCreate` 会再跑一遍。在里面生成的随机 ID 每次都不一样，需要存进 `savedInstanceState`。

第一轮讲过「中断前的副作用会重复执行」。第二轮补三点：
- **不确定的值也算**：时间、随机数、一次 LLM 调用，都要放在中断之前的独立节点里，或者放进 Functional API 的 `@task`（已完成的 task 结果从 checkpoint 读出，不会重算）。
- **多个中断按顺序匹配**：同一节点里有多个 `interrupt()` 时，靠调用顺序和恢复值对应，所以不能用条件跳过某个中断。并行分支同时中断时，用 `Command(resume={中断id: 值})` 一次性回复。
- **持久化有档位**：`durability="exit"` 只在退出时保存，最快；但进程中途崩溃会丢掉中间状态。有真实副作用的流程用 `"sync"`。

### 4. 可逆性进阶：不可逆的放最后，可逆的备好补偿

- **是什么**：一个多步任务里，先做可撤回的步骤，最后做不可撤回的那一步；每个已完成的写操作都准备一个「补偿动作」，后面失败了就倒序撤回。这种做法叫 Saga。
- **没有它会怎样**：先付了定金再去订座，订座失败时，钱已经付出去了。
- **对应你熟悉的**：数据库事务的回滚；Gmail 的「撤销发送」，其实是把真正不可逆的发送往后推了几秒。

Anthropic 的官方做法也是按可逆性分级。Claude Code auto mode 默认拦截 force push、生产部署、`git reset --hard`、`terraform destroy`，以及「不可逆地删除会话开始前就存在的文件」。Computer use 文档建议，凡是有现实后果、或者需要用户明确同意的操作（完成付款、同意服务条款），都要先请人确认。

## 高频面试题

**1. 用户已经批准了，为什么执行前还要再校验？校验什么？**
- 审批和执行之间有时间差，状态可能已经变了（TOCTOU）。
- 校验参数：执行的必须是审批时展示的那份，用摘要比对。
- 校验时效：审批设有效期，过期就重新发起，超时默认拒绝。
- 校验实时条件：车速、余额等在执行前重新读取，不用审批时的快照。

**2.（追问）你在 `approve` 节点里算了 `expires_at`，测试时发现审批永远不会过期，为什么？**
- 恢复时中断所在节点从头重跑，`expires_at` 按恢复时刻重算了。
- 修法：把时间戳放在中断之前的另一个节点，或者放进 `@task`，让它的结果从 checkpoint 读出。
- 同理，随机数、LLM 调用、对外请求都不要写在中断之前的同一节点里。
- 判断方法：这段代码跑两遍结果一样吗？不一样就要挪出去。

**3.（场景题）车主说「订明晚那家餐厅，付 200 元定金，再发消息约老王」，你怎么设计？**
- 排顺序：先查空位并锁座（可撤回）→ 再发消息（难撤回，审批时允许改正文）→ 最后付定金（不可逆，单独审批，展示金额和商户）。
- 每步留补偿：付款失败就取消锁座，并告诉车主消息已发出、需要的话可以再发一条更正。
- 付款审批绑定金额和商户，设较短有效期，执行前重查锁座是否还有效。
- 行驶中只用语音简单确认；涉及付款的确认改成停车后在屏幕上完成。

**4.（追问）多个护栏或 hook 给出的结论冲突，怎么合并？**
- 原则是最严格的结论生效。Claude Agent SDK 的优先级是：deny > defer > ask > allow。
- 匹配到的多个 hook 并行运行，取最严格的结果。
- 即使 hook 返回 allow，也不会跳过 deny 和 ask 规则；deny 规则在 `bypassPermissions` 模式下依然生效。
- 必须对每一次调用都生效的检查，放在 PreToolUse hook 里，不要放在 `canUseTool` 里，因为被规则自动放行的调用根本不会走到它。

## 常见误区

1. **「人不回复，审批自然就失效了」**：恰好相反。LangGraph 的中断会无限期等待，几小时后点「同意」照样能放行。有效期和默认拒绝要自己加。
2. **「输出护栏放在 `after_agent` 里，流式输出也能拦住」**：LangChain 文档没有说明 `after_agent` 检查会在流式 token 送达用户之前执行，已经流出去的内容收不回来。官方明确支持流式的是 PII 中间件：设置 `apply_to_output=True` 后，流式输出也会被打码。其他输出检查要么先缓冲再放出，要么接受「事后撤回」。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：审批三节点**（取自 `approval_guard.py`）

```python
# 发起请求：在中断之前算好摘要和过期时间，让它们存进 checkpoint
def propose(s):
    # 要执行的动作：工具名加参数
    action = {"tool": "open_window", "args": {"percent": 100}}
    # 返回请求：动作本身、参数摘要、过期时刻
    return {"request": {"action": action, "digest": digest(action), "expires_at": CAR["now"] + TTL}}

# 审批节点：只调用中断，把请求原样交给人，人的回复就是返回值
def approve(s):
    return {"approval": interrupt(s["request"])}

# 执行节点：放行前逐项复核
def execute(s):
    # 取出请求和人的回复
    req, ok = s["request"], s["approval"]
    # 人没同意：直接结束
    if not ok.get("approved"): return {"result": "车主拒绝"}
    # 人看到的摘要和现在的参数对不上：拒绝
    if ok.get("digest") != digest(req["action"]): return {"result": "拒绝：参数与审批时不一致"}
    # 已经过了有效期：拒绝，要求重新确认
    if CAR["now"] > req["expires_at"]: return {"result": "拒绝：审批已过期，请重新确认"}
    # 重查实时车况：开度大且车速高就不执行
    if req["action"]["args"]["percent"] > 30 and CAR["kmh"] > 60:
        return {"result": f"拒绝：当前车速 {CAR['kmh']} km/h，全开车窗不安全"}
    # 全部通过才真正执行
    return {"result": f"已执行 open_window {req['action']['args']}"}
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `expires_at` 必须写在 `propose` 里，写进 `approve` 就会在恢复时被重算。
- `interrupt()` 的返回值就是恢复时 `Command(resume=...)` 传进来的值，直接放进 state，不用再包一层。
- 复核的顺序：先看是否同意，再比摘要、看时效，最后查实时车况。前面几项都是便宜的本地判断，放前面。

**昨日复习（只做第③步）**：第二轮 Day 23 的「污点追踪中间件 TaintGuard」。只看你自己写的注释重写，重点核对：`state_schema = TaintState`、`request.tool_call["id"]`、`Command(update={"tainted": True, "messages": [result]})`。

## 动手练习（30 分钟）

在 `approval_guard.py` 上改：

1. 支持「改参数后批准」：车主回复 `{"approved": True, "edited_args": {"percent": 30}, ...}`。`execute` 要用改后的参数执行，并且对改后的参数重新做车速检查。
2. 想一想：改了参数后，摘要比对应该比哪一份？让车主回传「改后参数的摘要」，再用改后参数重新计算摘要来比对。
3. 过期后不要直接结束：加一条条件边，从 `execute` 回到 `propose`，重新发起审批（最多重试 1 次）。

提示：条件边用 `g.add_conditional_edges("execute", 路由函数)`。路由函数返回 `"propose"` 或 `END`，需要从 `langgraph.graph` 导入 `END`。

## 读后自测

1. 车主已经点了「同意」，为什么执行前还要再检查一次？要检查哪几样东西？见「核心概念 1、2」节和「先跑起来」节
2. 在 LangGraph 里，把审批过期时间写在调用 `interrupt()` 的那个节点里，为什么会出问题？见「核心概念 3」节
3. 一个任务里同时有可撤回和不可撤回的步骤，怎么排顺序？中途失败了怎么办？见「核心概念 4」节

## AI 历史知识点：Concrete Problems in AI Safety（2016，Dario Amodei、Chris Olah 等）

**是什么**：一篇把「AI 安全」从哲学讨论拉回工程问题的论文。作者把机器学习系统可能出现的「意外事故」归成五个可以动手研究的具体问题：避免副作用（avoiding side effects）、避免奖励投机（avoiding reward hacking）、可扩展的监督（scalable supervision）、安全探索（safe exploration）、分布偏移（distributional shift）。

**何时由谁提出**：2016 年以 arXiv 预印本发表（arXiv:1606.06565），作者是 Dario Amodei、Chris Olah、Jacob Steinhardt、Paul Christiano、John Schulman、Dan Mané。论文收录在 Google Research 的出版物列表中。

**为什么重要**：它第一次用工程语言定义了「系统按目标做事，却产生了没人想要的后果」这类问题，后来很多安全研究都沿用了这套分类。

**和今天的趋势有什么联系**：今天讲的几样东西，几乎都能在这五个问题里找到原型。不可逆动作要审批、先做可撤回的步骤，对应「避免副作用」和「安全探索」；靠人来审批 Agent 的动作，而且要解决审批疲劳，对应「可扩展的监督」。十年前论文里的强化学习智能体，换成了今天能调工具的 LLM Agent，问题本身没变。

来源：[Concrete Problems in AI Safety - Google Research](https://research.google/pubs/concrete-problems-in-ai-safety/)

## 参考区（用时查，不必通读）

### 细节速查

**LangGraph 中断与恢复（langgraph 1.2.x）**
- 导入：`from langgraph.types import interrupt, Command`；`from langgraph.checkpoint.memory import InMemorySaver`。中断需要 checkpointer + `thread_id`，payload 必须能序列化成 JSON。
- 读待处理中断：`invoke()` 结果里的 `result["__interrupt__"]`（每项有 `.value`、`.id`）；`get_state(cfg).tasks[i].interrupts`；`get_state(cfg).next` 为空表示已经跑完。
- 恢复：`Command(resume=值)`；多个中断一起恢复用 `Command(resume={i.id: 值 for i in interrupts})`；同一节点里的多个中断严格按调用顺序匹配。
- 校验人的输入：不要在节点里写 `while True` 循环调用 `interrupt()`；每次运行只中断一次，把追问写进 state，再用条件边回到这个节点。
- 等待时长：没有内置过期，会一直等下去。
- 持久化档位：`durability="exit"`（只在退出时保存，最快）/ `"async"`（下一步执行时异步保存，有很小的丢失风险）/ `"sync"`（下一步开始前保存）。
- Functional API：恢复时从 entrypoint 开头重跑，已完成的 `@task` 结果直接从 checkpoint 读取；开始了但没跑完的 task 可能再跑一次，副作用要用幂等键保护。
- `StateGraph.add_sequence([...])`：按顺序添加节点并连边。

**Claude Agent SDK 权限**
- 评估顺序：hooks → deny 规则 → ask 规则 → 权限模式 → allow 规则 → `canUseTool`。
- 权限模式：`default`（CLI 里也叫 manual）、`dontAsk`、`acceptEdits`、`bypassPermissions`、`plan`、`auto`。
- PreToolUse hook 返回 `hookSpecificOutput.permissionDecision`：`allow` / `deny` / `ask` / `defer`，可附带 `permissionDecisionReason`、`updatedInput`。冲突时 deny > defer > ask > allow。
- `defer` 会结束本轮，`stop_reason` 为 `"tool_deferred"`，此时 `updatedInput` 被忽略。`updatedInput` 配 `allow` 表示自动放行改后的参数，配 `ask` 表示把改后的参数展示给用户。
- hook 默认超时 600 秒，超时后这次调用被阻止。`canUseTool` 可以一直挂起；等待时间可能很长时改用 hook `defer`。
- `canUseTool` 返回 `{behavior: "allow", updatedInput}` 或 `{behavior: "deny", message}`；想让批准长期有效，就把 `suggestions` 回传到 `updatedPermissions`。

**Claude Code auto mode 默认拦截**：force push、生产部署和迁移、不可逆地删除会话开始前就存在的文件、`git reset --hard`、`terraform destroy`。关键路径上的 `rm` 不能被任何 allow 规则或 hook 的 allow 放行。沙箱只约束 shell 命令，和权限规则、权限模式是互补的几层。

**LangChain PII 中间件**：策略有 redact（默认）/ mask / hash / block；开关有 `apply_to_input`、`apply_to_output`、`apply_to_tool_results`。`apply_to_output=True` 时流式输出也会被打码（需要 `langchain>=1.3.2`）。

### 来源说明

- 示例代码里的车况和时钟是模拟的，图、中断、checkpointer 都是真实的 LangGraph 代码。审批摘要和过期机制是本篇自己的示意，不是框架内置功能。
- LangGraph Durable execution 页面这次没能打开，相关内容来自 Checkpointers 和 Functional API 页面。
- 历史知识点：arXiv 原页没能打开，作者和五个问题以 Google Research 页面为准，属于非 LangChain/Claude 来源。

### 延伸学习

- Claude Academy · Claude Code in Action：「Permission modes」「Hooks」两课
- Claude Academy · The AI-Native SDLC Playbook：「Hooks as approval gates」
- LangChain 文档 · Interrupts、Functional API

### 参考资料

- [Interrupts - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [Checkpointers - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/checkpointers)
- [Functional API - Docs by LangChain](https://docs.langchain.com/oss/python/langgraph/functional-api)
- [Guardrails - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/guardrails)
- [Handle approvals and user input - Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/user-input)
- [Permissions - Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/permissions)
- [Hooks - Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/hooks)
- [Permission modes - Claude Code Docs](https://code.claude.com/docs/en/permission-modes)
- [Sandboxing - Claude Code Docs](https://code.claude.com/docs/en/sandboxing)
- [Computer use tool - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)
- [Our framework for developing safe and trustworthy agents - Anthropic](https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents)
- [Trustworthy agents in practice - Anthropic](https://www.anthropic.com/research/trustworthy-agents)
- [How we built Claude Code auto mode - Anthropic](https://www.anthropic.com/engineering/claude-code-auto-mode)
- [Claude Code in Action - Claude Academy](https://academy.claude.com/courses/claude-code-in-action)
- [Hooks as approval gates - Claude Academy](https://academy.claude.com/courses/ai-native-sdlc-playbook/hooks-as-approval-gates)
