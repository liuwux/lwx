# Day 29 · 系统设计题（一）：企业客服 Agent（2026-10-27）

## 今日 AI 动态（2026-10-27）

### 趋势观察

过去 24 小时只有一条官方发布，但分量很重：Anthropic 正式提交 IPO 申请。头部模型公司开始进入公开市场，意味着收入、亏损和治理结构都要定期公开披露。推测：后续模型定价和企业产品节奏，会更多受到季度财务指标的影响。

### 今日要点

- Anthropic 向美国 SEC 提交 S-1 招股书，计划在纳斯达克上市，代码「ANT」。

### 分公司动态

#### 1. Anthropic

**[Anthropic files S-1 with the SEC for proposed initial public offering](https://www.anthropic.com/news/s1-sec)**（2026-10-26）

Anthropic 宣布已向美国 SEC 提交 S-1 注册声明，拟首次公开发行 A 类普通股，计划在纳斯达克全球精选市场上市，股票代码「ANT」，发行股数和价格区间尚未确定，主承销商为高盛和摩根士丹利。据 TechCrunch 对招股书的报道，2026 年上半年收入 163 亿美元（去年同期 12 亿美元），年消费超过 100 万美元的企业客户约 1900 家；七位联合创始人组成的「Founder LLC」将掌握 50.1% 的投票权。

**为什么重要**：这是第一家以公益公司（PBC）身份申请上市的头部大模型厂商，招股书让外界第一次看到企业级 AI 业务的真实收入规模，也会影响其他 AI 公司的融资和上市节奏。

### 动态来源

- [Anthropic files S-1 with the SEC for proposed initial public offering](https://www.anthropic.com/news/s1-sec)
- [Anthropic officially files for its public debut - TechCrunch](https://techcrunch.com/2026/10/26/anthropic-officially-files-for-its-public-debut/)（媒体报道，用于补充招股书数字）

---

## 读前三问

1. 面试官说「设计一个企业客服 Agent」，你按什么顺序答？
2. 客服 Agent 的工具怎么分级？退款这类操作怎么防止模型「自作主张」？
3. 怎么评测一个客服 Agent，又怎么把它安全地上线？

## 从一个场景开始

你在一家车企做车主 App。客服中心每天收到上万条咨询：「首保免费吗」「我的配件订单到哪了」「充电桩装不上要退款」。

老板说：「做个 AI 客服，把人工量降一半。」

第一版你直接把大模型接进聊天框，很快出了问题：

- 车主问保养政策，模型编了一个「首保 1 万公里内免费」，实际是 5000 公里。
- 车主说「给我退款」，模型回复「已为您退款」，可后台什么也没发生。
- 车主情绪激动要找人工，模型还在一遍遍道歉，转不出去。

你缺的不是一个更强的模型，而是一套**系统设计**：答案有出处、操作有审批、搞不定能转人工、上线前有评测。这就是今天的面试题。

## 一句话总结

客服 Agent = 带引用的知识库问答 + 分级的业务工具 + 高风险操作人工审批 + 可交接的转人工，再用解决率等指标评测、灰度上线。

## 先跑起来

用 LangChain 的 `create_agent` 搭一个最小客服 Agent：一个只读的知识库工具，一个需要人工审批的退款工具。为了不依赖 API Key，模型换成按剧本出牌的假模型，只看流程。

```python
# support_agent.py —— 车企客服 Agent：知识库问答 + 退款需人工审批（剧本模型代替真实 LLM）
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage, ToolMessage
from langchain_core.outputs import ChatGeneration, ChatResult
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command

KB = {"KB-07": "首保在 5000 公里或 6 个月内免费，到任一授权门店均可。"}
@tool
def search_kb(query: str) -> str:
    """查官方知识库（保养、质保、充电等政策），返回带编号的条款。"""
    return "[KB-07] " + KB["KB-07"]
@tool
def refund(order_id: str, amount: int) -> str:
    """为订单 order_id 退款 amount 元。高风险操作，执行前需人工审批。"""
    return f"订单 {order_id} 已退款 {amount} 元"

class Scripted(GenericFakeChatModel):  # 剧本模型：按关键词调工具，拿到结果后回复
    def bind_tools(self, tools, **kw): return self
    def _generate(self, messages, stop=None, run_manager=None, **kw):
        last = messages[-1]
        if isinstance(last, ToolMessage): msg = AIMessage(f"客服：{last.content}")
        elif "保养" in last.content: msg = AIMessage("", tool_calls=[{"name": "search_kb", "args": {"query": "首保"}, "id": "k1"}])
        else: msg = AIMessage("", tool_calls=[{"name": "refund", "args": {"order_id": "A1001", "amount": 299}, "id": "r1"}])
        return ChatResult(generations=[ChatGeneration(message=msg)])

agent = create_agent(
    model=Scripted(messages=iter([])), tools=[search_kb, refund], checkpointer=InMemorySaver(),
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"refund": {"allowed_decisions": ["approve", "reject"]}, "search_kb": False})],
)
cfg = {"configurable": {"thread_id": "user-42"}}  # 一个会话一个 thread_id
r = agent.invoke({"messages": [("user", "首次保养免费吗？")]}, cfg, version="v2")
print("问1 ->", r.value["messages"][-1].content, "| 中断:", r.interrupts)
r = agent.invoke({"messages": [("user", "配件订单 A1001 我要退款")]}, cfg, version="v2")
req = r.interrupts[0].value["action_requests"][0]
print("问2 -> 暂停等审批:", req["name"], req["args"])
r = agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), cfg, version="v2")  # 坐席点了「同意」
print("审批后 ->", r.value["messages"][-1].content)
```

**运行**：`pip install -U langchain langgraph && python support_agent.py`

**实测输出**（2026-10-27，Python 3.13，langchain 1.4.7）：

```
问1 -> 客服：[KB-07] 首保在 5000 公里或 6 个月内免费，到任一授权门店均可。 | 中断: ()
问2 -> 暂停等审批: refund {'order_id': 'A1001', 'amount': 299}
审批后 -> 客服：订单 A1001 已退款 299 元
```

**该看什么**：
- 问 1 的回答带着 `[KB-07]` 编号：答案来自知识库，而且可以追溯。中断列表是空的，只读工具不打扰人。
- 问 2 里模型已经决定退款，但 Agent **停在了执行之前**，把工具名和参数交给人审。
- 审批用的是同一个 `thread_id`，靠 checkpointer 把暂停时的状态存住，恢复后才真正执行。

## 核心概念

### 1. 系统设计的答题顺序

- **是什么**：按「需求 → 架构 → 工具 → 评测 → 上线」五步讲，每一步先说结论再说取舍。
- **没有它会怎样**：一上来就讲用哪个模型、哪个框架，面试官听不出你有没有想清楚「要解决什么、怎么算成功」。
- **对应你熟悉的**：Android 面试里设计一个图片加载库，也是先问需求（缓存、并发、生命周期），再画模块图。

需求阶段要问清：渠道（App、电话、企业微信）、问题类型占比、能执行哪些操作、高峰并发、合规要求。Anthropic 的客服指南建议**先画出一段理想对话，再拆成几个任务**，比如问候、产品咨询、话题管理、报价。

### 2. 知识问答走检索，实时数据走工具

- **是什么**：政策、FAQ 这类相对稳定的内容放进知识库，用检索取最相关的几段并要求回答带出处；订单、余额、物流这类实时数据，只能通过工具查业务系统。
- **没有它会怎样**：模型凭记忆回答政策就会编数字；把订单数据塞进知识库，查到的永远是旧状态。
- **对应你熟悉的**：本地 Room 缓存放不常变的配置，实时状态必须请求接口。

Claude 官方客服指南明确区分了这两点：长上下文用 RAG 只取相关内容，降成本、降延迟；账户、订单这类实时信息要用工具，光靠嵌入检索不够。

### 3. 工具分级与人工审批

- **是什么**：只读工具（查知识库、查订单）放行；写操作（退款、改地址）按金额和风险分级，高风险的执行前必须暂停等人确认。
- **没有它会怎样**：模型理解错一句话，钱就退出去了；或者模型嘴上说「已退款」，后台根本没调用。
- **对应你熟悉的**：运行时权限：读相册和打电话的授权级别不一样。

在 LangChain 里，用 `HumanInTheLoopMiddleware` 的 `interrupt_on` 给每个工具配置是否审批。暂停依赖 checkpointer 保存状态，所以**审批流程必须配持久化存储**。官方客服指南的护栏里也写了：不做承诺、不达成协议，Agent 只提供信息和指引。

### 4. 转人工是设计的一部分

- **是什么**：明确触发条件（用户要求人工、连续两轮没解决、情绪激烈、超出权限），转人工时把对话摘要、已查到的订单、已尝试的方案一起交给坐席。
- **没有它会怎样**：用户被困在机器人里反复兜圈；坐席接手后还要让用户从头再说一遍。
- **对应你熟悉的**：页面跳转时用 `Bundle` 把状态带过去，而不是让下一个页面重新加载。

LangChain 的客服教程把流程做成状态机：每一步（查质保 → 分类问题 → 解决）有自己的提示词和工具，`escalate_to_human` 是其中一个正式工具，而不是异常分支。

### 5. 用指标评测，用灰度上线

- **是什么**：先定成功标准，再建测试集离线评测，最后小流量上线、对比人工。
- **没有它会怎样**：只能靠「看几条对话感觉不错」决定上线，出问题时也说不清是变好还是变差。
- **对应你熟悉的**：App 发版的灰度发布 + 崩溃率监控。

Claude 官方指南把指标分两类：**质量**（理解准确率、回答准确率、引用相关性、话题遵守、转人工效率）和**业务**（情绪、分流率、满意度、平均处理时长）。Anthropic 在《Building effective agents》里也说，客服适合 Agent 的原因之一，就是成功可以用「问题是否被解决」清楚衡量。

## 高频面试题

**1.（场景题）设计一个车企 App 的客服 Agent。**
- 需求：渠道、问题分布、可执行操作、并发和合规要求，定好解决率等目标。
- 架构：接入层 → 意图路由 → Agent（知识库检索 + 业务工具）→ 会话状态存储 → 人工坐席台。
- 工具分级：只读放行，写操作按风险审批，转人工是正式工具。
- 评测与上线：离线测试集 → 影子模式 → 灰度 → 全量，全链路日志可追溯。

**2.（追问）退款金额阈值怎么定？审批由谁来做？**
- 按金额和风险分档：小额自动、中额坐席审批、大额或异常主管审批。
- 阈值来自历史数据和业务方，不是工程师拍脑袋。
- 工具层再做一次硬校验（订单归属、是否已退过），不依赖模型。
- 审批记录和模型原始请求一起留存，便于审计。

**3. 如何减少客服 Agent 的幻觉？**
- 政策类问题必须先检索，回答带条款编号，没检索到就说不知道并转人工。
- 实时数据只来自工具返回，不让模型猜。
- 护栏：不承诺、不提竞品、不猜未来产品。
- 用带标准答案的测试集定期回归，监控引用准确率。

**4. 意图很多（咨询、投诉、退款、技术故障）时怎么组织？**
- 先做一个意图分类，把请求路由到专门的提示词和工具集。
- 代价是多一次模型调用的延迟，可用小模型做分类。
- 或者用状态机：每一步只暴露当前需要的工具。
- 先把单 Agent 的提示词和工具优化到位，再考虑拆分。

## 常见误区

1. **「模型够强就不用审批」**：审批防的不只是模型犯错，还有提示注入和用户误操作。写操作的最后一道关必须在代码里，而不在提示词里。
2. **「把转人工率降到 0 才算成功」**：一味压低转人工率，会把搞不定的问题硬留在机器人里，满意度反而下降。目标是「该转的及时转，转过去的信息完整」。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带人工审批的客服 Agent**（取自 `support_agent.py`）

```python
# 把只读的查知识库函数声明成工具
@tool
# 参数是查询词，返回带编号的条款文本
def search_kb(query: str) -> str:
    # docstring 写清用途：查政策类知识，返回带编号条款
    """查官方知识库（保养、质保、充电等政策），返回带编号的条款。"""
    # 返回条款时带上编号，方便回答引用出处
    return "[KB-07] " + KB["KB-07"]

# 用 create_agent 组装客服 Agent
agent = create_agent(
    # 传入模型、两个工具和内存版 checkpointer（审批暂停要靠它保存状态）
    model=Scripted(messages=iter([])), tools=[search_kb, refund], checkpointer=InMemorySaver(),
    # 中间件：退款只允许同意或拒绝，查知识库不需要审批
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"refund": {"allowed_decisions": ["approve", "reject"]}, "search_kb": False})],
)
# 一个会话用一个 thread_id，审批前后必须相同
cfg = {"configurable": {"thread_id": "user-42"}}
# 用户发起退款，用 v2 格式调用以拿到 interrupts
r = agent.invoke({"messages": [("user", "配件订单 A1001 我要退款")]}, cfg, version="v2")
# 取出第一个待审批动作：工具名和参数
req = r.interrupts[0].value["action_requests"][0]
# 坐席同意后，用 Command 带着决定恢复执行
r = agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), cfg, version="v2")
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 恢复时传的是 `Command(resume={"decisions": [...]})`，不是新的 `{"messages": ...}`；`decisions` 是列表，顺序对应待审批动作。
- 没有 `checkpointer` 就无法暂停后恢复；恢复时的 `cfg` 必须是同一个 `thread_id`。
- `interrupt_on` 的值可以是 `True`、`False` 或配置字典，字典里的键是 `allowed_decisions`。

**昨日复习（只做第③步）**：Day 28 的「带校验的工具 + 端云路由中间件」。只看你自己写的注释重写，重点核对：`request.runtime.context.online`、最后的 `return handler(...)`、`request.override(model=...)` 返回新请求。

## 动手练习（30 分钟）

在 `support_agent.py` 上改：

1. 把审批决定换成 `{"type": "reject", "message": "订单已超过 7 天退款期"}`，看最后的回复变成什么。
2. 加一个工具 `escalate_to_human(summary: str)`，返回「已转人工，工单号 T-001」。在剧本模型里：用户原话含「人工」时调用它。
3. 在 `refund` 工具内部加硬校验：`amount > 500` 时直接返回错误信息，不执行退款。

## 读后自测

1. 「设计一个企业客服 Agent」按什么顺序答？→ 见核心概念 1，面试题 1
2. 工具怎么分级？退款怎么防止模型自作主张？→ 见先跑起来、核心概念 3，面试题 2
3. 怎么评测、怎么上线？→ 见核心概念 5，面试题 1、3

---

## AI 历史知识点：WebGPT（2021，OpenAI）

**是什么**：OpenAI 在 GPT-3 上微调出的一个「会上网查资料」的问答模型。它在一个纯文本浏览环境里自己搜索、点开网页、摘录段落，最后写出长答案，并**附上引用来源**。

**何时由谁提出**：2021 年 12 月 17 日，OpenAI 的 Reiichiro Nakano、Jacob Hilton、Suchir Balaji 等人在 arXiv 发布论文《WebGPT: Browser-assisted question-answering with human feedback》。

**为什么重要**：它是最早一批「模型调用外部工具完成任务」的代表作之一。训练方法也有代表性：先模仿人类示范（行为克隆），再用人类偏好训练的奖励模型做拒绝采样。在 Reddit ELI5 问题上，最佳模型的答案有 56% 的情况比人类示范者写的更受偏好，69% 的情况胜过 Reddit 最高赞回答。

**和今天的趋势有什么联系**：WebGPT 让模型「边查边答、答案带出处」，方便人核对事实。今天客服 Agent 要求「先检索、再回答、带条款编号」，搜索类 Agent 普遍附引用，都是这个思路的延续。

原始来源：[WebGPT: Browser-assisted question-answering with human feedback (arXiv:2112.09332)](https://arxiv.org/abs/2112.09332v3)

---

## 参考区（用时查，不必通读）

### 细节速查

**LangChain 人工审批（HumanInTheLoopMiddleware）**
- 导入：`from langchain.agents.middleware import HumanInTheLoopMiddleware`
- `interrupt_on={工具名: True | False | {"allowed_decisions": [...], "description": ...}}`；`True` 表示允许全部决定
- 决定类型：`{"type": "approve"}`、`{"type": "edit", "edited_action": {"name": ..., "args": {...}}}`、`{"type": "reject", "message": ...}`（message 可选）、`{"type": "respond", "message": ...}`
- 必须配 checkpointer：测试用 `InMemorySaver`，生产用 `AsyncPostgresSaver` 等
- `invoke(..., version="v2")` 返回 `GraphOutput`：`.value` 是状态，`.interrupts` 是中断元组
- 待审批动作：`r.interrupts[0].value["action_requests"]`，每项含 `name`、`args`

**LangChain 客服状态机（handoffs-customer-support 教程）**
- 状态：继承 `AgentState`，字段 `current_step`、`warranty_status`、`issue_type`
- 工具通过 `Command(update={...})` 更新状态并切换步骤，`ToolMessage` 用 `runtime.tool_call_id`
- `@wrap_model_call` 中间件按 `current_step` 调 `request.override(system_prompt=..., tools=...)`
- 长对话：`SummarizationMiddleware(model=..., trigger=("tokens", 4000), keep=("messages", 10))`

**Claude 官方客服指南要点**
- 提示词分段：`IDENTITY`（放 system）、静态知识（`<static_context>`）、4–5 个以上示例、护栏、任务说明；大部分指令放首个 user 轮
- 护栏：只谈自家产品、婉拒不提供的服务、不猜未来产品、不做承诺、不提竞品、去除 PII
- 模型：复杂多步客服用 Opus 5.5；提示词大、延迟敏感用 Haiku 5.5；`effort: low` 最快
- 生产化：Streamlit 只做原型，生产用 FastAPI/Flask，支持 SSE 流式、缓存、会话保持
- 扩展：任务种类多时加意图分类器路由到专门对话，多一次调用延迟

**评测指标（Claude 官方指南）**

| 类别 | 指标 |
|---|---|
| 质量 | 理解准确率、回答相关性、回答准确率、引用相关性、话题遵守、内容生成效果、转人工效率 |
| 业务 | 情绪维持、分流率（deflection）、客户满意度（CSAT）、平均处理时长（AHT） |

### 来源说明

- 代码用剧本模型代替真实 LLM，只演示工具调用与审批流程；已在本地实测。
- 「今日 AI 动态」中的财务数字来自 TechCrunch 对 S-1 的报道（媒体来源）；Anthropic 官方公告未列这些数字。历史知识点来源为 arXiv（非 LangChain/Claude 来源）。

### 延伸学习

- Claude Docs · Use case guides → Customer support agent
- Docs by LangChain · Build customer support with handoffs

### 参考资料

- [Customer support agent - Claude Platform Docs](https://platform.claude.com/docs/en/about-claude/use-case-guides/customer-support-chat)
- [Building effective AI agents - Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- [Build customer support with handoffs - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs-customer-support)
- [Human-in-the-loop - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
