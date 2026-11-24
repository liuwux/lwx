# Day 27 · 模型适配（第二轮：换模型迁移、评测选型、路由与回退、拒答处理）（2026-11-24）

## 今日 AI 动态（2026-11-24）

### 趋势观察

本期最集中的方向是 **Agent 安全**：Google 用「工具调用前由外部裁判核对用户意图」把提示注入攻击成功率压到 1% 以下，OpenAI 与 Cisco 则开源了让 AI 帮忙生成「临时漏洞护盾」的运行时，两者都要求人或外部策略做最终把关。另一条线是**多模型分工**进入生产：OpenAI 的客户案例里，一个大模型负责给前线 Agent 造工具和流程，前线用更小的模型。Anthropic 的访谈研究则提醒，企业里 AI 的价值主要体现在「产出变多」，瓶颈在审核和维护。

### 今日要点

- Google 公布语义治理策略的实测：AgentDojo 上攻击成功率从 48.7% 降到 0.9%，正常任务效用只降 2.6 个点。
- OpenAI 与 Cisco 开源 Shirudo：基于 eBPF 的临时「护盾」，在补丁上线前挡住漏洞的利用路径。
- Anthropic 发布对 58 家企业客户的访谈研究：AI 价值主要来自产出增加，而不是裁员降本。
- OpenAI 客户案例：Richpanel 用「大模型造工具 + 小模型跑前线」的分工，30 天内自动化 50% 客服。

### 分公司动态

#### 1. Anthropic

**[Working with AI](https://www.anthropic.com/research/working-with-ai)**（2026-11-23，Anthropic Economics 团队）

- 基于对 58 家美国企业客户的 66 次访谈。多数企业说 AI 的价值来自做得更多，而不是削减成本。
- 100 个用 AI 搭建的软件项目里，80% 是把手工流程自动化，20% 替换了原有软件（常是外采的）。
- 归因于 AI 的裁员很少：2 家完全取消了某个岗位，10 家反而多招了人，约三分之二表示招得更少。一个文档审核 Agent 让一家公司计划招聘的审核员减少了 90% 以上。
- **为什么重要**：它点名了 Agent 落地的实际瓶颈：大量产出要人审核、数据和上下文不足、Agent 搭的工具需要长期维护。

#### 2. OpenAI

**[OpenAI and Cisco introduce Shirudo](https://openai.com/index/introducing-shirudo/)**（2026-11-23）

- 一个基于 eBPF 的开源 Linux 运行时。发现漏洞后，先部署一个临时「护盾」，只拦截漏洞利用的那条代码路径，正常请求照常通过，为测试和上线真补丁争取时间。
- 护盾需要运维人员批准，加载时不用重启服务，带加密签名并经过 eBPF 校验器检查，由 TPM 记录审计日志。
- 受控测试中，Codex 生成的护盾在不重启的情况下挡住了运行中 Elasticsearch 上的一次漏洞利用。代码以 BSD-2-Clause 开源。
- **为什么重要**：安全类 Agent 有了「调查—生成防护—测试」的可编程落点，同时保留人工批准这一步。

**[Richpanel helps companies automate 50% of support in 30 days](https://openai.com/business/customer-stories/richpanel-ai-customer-support)**（2026-11-23，客户案例）

- Richpanel 的「Boss AI」运行在 GPT-6 Astra 和 GPT-6.1 Sol 上，负责为前线客服 Agent 构建工具和处理流程；前线 Agent 运行在 GPT-6 Luna 上。
- 六个月完成约 300 个部署，承诺 30 天内自动化 50% 工单，做不到就退款。
- **为什么重要**：这是今天主题「多模型分工」的生产实例：贵模型做低频的设计工作，便宜模型做高频执行。

#### 3. Google

**[Did the user ask for this? Securing AI agents against intent hijacking with Semantic governance policies](https://developers.googleblog.com/did-the-user-ask-for-this-securing-ai-agents-against-intent-hijacking-with-semantic-governance-policies/)**（2026-11-23，Google Developers Blog）

- 语义治理策略（Gemini 平台预览版，经 Agent Gateway 执行）：每个工具调用在执行前，由一个 LLM 裁判对照用户原始请求和策略检查。
- 在 AgentDojo 基准上用 gemini-2.5-flash Agent 测试，攻击成功率从 48.7% 降到 0.9%，正常任务效用下降 2.6 个点；929 次需要调用工具的攻击无一成功。
- 作者说明没有测试自适应攻击，纯文本层面的操纵仍需要其他防护。
- **为什么重要**：用实测数据证明了「在 Agent 外部校验工具调用」对间接提示注入很有效，可以直接借鉴到自己的护栏设计。

#### 4. Meta

**[Researchers Are Using Meta's Open-Weight AI Models to Transform Medicine](https://about.fb.com/news/2026/11/researchers-using-meta-open-weight-ai-models-medical-breakthroughs/)**（2026-11-23）

- Meta 扩大学术合作，向研究者提供资金和 Muse Glimmer、SAM 3、DINOv3、Brain2Qwerty v2 等模型。
- 介绍了三个早期项目：LSU 在 Quest 3 上做医疗操作的 VR 训练教练；一家医院基金会模拟癫痫发作起源；哥伦比亚大学把 DINO 变体改造用于细胞显微图像分析。文中没有给出量化结果。
- **为什么重要**：它展示了开源权重模型被「适配」到专业领域的常见路径，和今天的主题直接相关。

### 动态来源

- [Anthropic · Working with AI](https://www.anthropic.com/research/working-with-ai)
- [OpenAI · Introducing Shirudo](https://openai.com/index/introducing-shirudo/)
- [OpenAI · Richpanel customer story](https://openai.com/business/customer-stories/richpanel-ai-customer-support)
- [Google Developers Blog · Semantic governance policies](https://developers.googleblog.com/did-the-user-ask-for-this-securing-ai-agents-against-intent-hijacking-with-semantic-governance-policies/)
- [Meta Newsroom · Open-weight models in medicine](https://about.fb.com/news/2026/11/researchers-using-meta-open-weight-ai-models-medical-breakthroughs/)

---

## 读前三问

1. 把车载助手的底座从老模型换成新模型，为什么不能只改一个模型名？上线前要检查哪几类东西？
2. 什么时候该「换更大的模型」，什么时候该「调高 effort」或「拆成大小模型路由」？用什么数据来做这个决定？
3. 主模型过载时自动回退到备用模型，这套回退能不能也兜住模型拒答？为什么？

## 从一个场景开始

第一轮我们讲了「什么时候值得微调」，结论是先把 Prompt 和 RAG 用到头。车载助手团队照做了，没微调，一直用同一个老模型。

这周产品要求升级到新模型。工程师改了一行模型名，测试环境立刻报 400：请求里写死了 `temperature=0`，还有一段「助手预填」。改掉之后能跑了，但又发现：同样的提示词，新模型回答变长了，因为老提示里那句「务必详尽、反复核对」被新模型当真照做；token 用量涨了一截，因为分词器变了。

紧接着是成本问题：所有请求都走大模型太贵，「打开空调」这种指令根本不需要。于是加了大小模型路由，又加了过载回退。结果某天用户问到改装电池，大模型拒答了，回退却没有触发，前端直接显示一句「抱歉」。

第二轮要讲的就是：**换模型是一次迁移，选模型要靠评测，多模型要有路由和回退，而拒答需要单独处理。**

## 一句话总结

换模型要当成一次迁移来做：先跑评测集再上线。选型看「每完成一个任务的成本」，先调强度再换模型。多模型靠路由加回退，但拒答是正常返回，回退兜不住，要单独判断。

## 先跑起来

车载助手：短指令走小模型，复杂规划走大模型；大模型过载时回退到备用模型。最后用一个对照实验看「拒答」会不会触发回退。用 LangChain 的假模型离线运行，不需要 API key。

```python
# car_route.py —— 车载助手：按难度路由大小模型 + 主模型过载时回退（langchain 1.5，假模型离线可跑）
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware, wrap_model_call
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage
CALLS = []                                        # 记录每次真正被调用的模型
class Fake(GenericFakeChatModel):
    tag: str = ""
    fail: bool = False
    def _generate(self, messages, stop=None, run_manager=None, **kw):
        CALLS.append(self.tag)
        if self.fail:
            raise RuntimeError(f"{self.tag}: 529 overloaded")
        return super()._generate(messages, stop, run_manager, **kw)
def fake(tag, text, fail=False, meta=None):
    msg = AIMessage(content=text, response_metadata=meta or {})
    return Fake(tag=tag, fail=fail, messages=iter([msg] * 5))
small = fake("small", "已为你打开空调，24 度")
big = fake("big", "", fail=True)                  # 大模型此刻过载
backup = fake("backup", "路线：先去 A 站充电 20 分钟，再走 G4")
refuser = fake("big2", "抱歉，这个我不能帮忙", meta={"stop_reason": "refusal"})
@wrap_model_call
def route(request, handler):                      # 短指令走小模型，长任务走大模型
    hard = len(request.messages[-1].content) > 12
    return handler(request.override(model=big if hard else small))
agent = create_agent(small, tools=[], middleware=[route, ModelFallbackMiddleware(backup)])  # 顺序：路由在外层
for q in ["打开空调", "帮我规划去杭州的路线，电量 30%，中途要充一次电"]:
    CALLS.clear()
    out = agent.invoke({"messages": [{"role": "user", "content": q}]})
    print(f"{q[:8]:<10} 调用链 {CALLS} → {out['messages'][-1].content}")
# 对照：拒答不是异常，回退中间件不会被触发
agent2 = create_agent(refuser, tools=[], middleware=[ModelFallbackMiddleware(backup)])
CALLS.clear()
m = agent2.invoke({"messages": [{"role": "user", "content": "教我改装电池"}]})["messages"][-1]
print("拒答样例  调用链", CALLS, "| stop_reason =", m.response_metadata.get("stop_reason"))
```

**运行**：`pip install langchain && python car_route.py`

**实测输出**（2026-11-24，Python 3.13，langchain 1.5.2 / langchain-core 1.8.4）：

```
打开空调       调用链 ['small'] → 已为你打开空调，24 度
帮我规划去杭州的   调用链 ['big', 'backup'] → 路线：先去 A 站充电 20 分钟，再走 G4
拒答样例  调用链 ['big2'] | stop_reason = refusal
```

**该看什么**：

- 第二行调用链是 `['big', 'backup']`：路由先选了大模型，大模型抛出过载异常，回退中间件接住后换成备用模型。
- 第三行只调用了一次：拒答是一次「成功」的返回，没有抛异常，所以回退不会发生。必须自己检查 `stop_reason`。
- 中间件顺序很关键。最初把 `ModelFallbackMiddleware` 放在 `route` 前面，运行直接报 `RuntimeError: big: 529 overloaded`：外层的回退把模型换成 backup 之后，内层的 `route` 又把它改回了 big。列表里靠前的中间件在外层。

## 核心概念

### 1. 换模型是一次迁移，不是改个字符串

- **是什么**：新模型的请求规则、默认行为和计费方式都可能变化。以 Claude Opus 5.5 迁移指南为例：`temperature`/`top_p`/`top_k` 要去掉，预填助手消息会返回 400，强制工具调用只剩 `auto`/`none`，默认 effort 从 `high` 降为 `medium`，新分词器会多用约 0～35% 的 token。
- **没有它会怎样**：上线当天出现 400；或者能跑但行为悄悄变了，比如回答变长、思考变浅、成本上涨，没人发现。
- **对应你熟悉的**：升级 `targetSdkVersion`。代码照样编译，但运行时权限、后台限制等行为变了，必须对照行为变更清单逐条检查。

迁移检查分三类：**请求**（会不会 400）、**响应**（拒答、思考块按 `type` 读取）、**提示词**（删掉为老模型写的「反复核对」这类补丁，官方原话是新模型会「逐字照做」）。

### 2. 评测驱动选型：比「每个任务的成本」，不比单价

- **是什么**：用同一个评测集，对比不同模型和不同 effort 的通过率与成本。官方建议算「每完成一个任务的成本」，并且要看长尾请求，不能只看中位数。
- **没有它会怎样**：只看每百万 token 的单价，小模型看起来便宜，但它要多轮重试、多调工具，最终每个任务反而更贵。
- **对应你熟悉的**：性能优化看 P99 耗时，不只看平均耗时。

两条官方建议：调 effort 往往比换模型更有效；只有可以自动验证结果的任务，才适合「先低 effort 跑，失败再用高 effort 重跑」，官方给的例子能在通过率不变的情况下便宜约 40%。

### 3. 路由与回退：两件事，位置不同

- **是什么**：**路由**在调用前决定用哪个模型（按难度、按任务类型）；**回退**在调用失败后换一个模型重试。LangChain 里分别对应 `@wrap_model_call` 自定义中间件（或 `ModelRouterMiddleware`）和 `ModelFallbackMiddleware`。
- **没有它会怎样**：所有请求都走大模型，又贵又慢；或者主模型一过载，整个车载助手就不可用。
- **对应你熟悉的**：OkHttp 的拦截器链。`addInterceptor` 的顺序决定谁先看到请求，谁最后改写它。

官方说「多模型组合必须在整条成本-质量曲线上胜过单个模型」才值得做，否则只是增加复杂度。

### 4. 拒答不是错误：回退之外要单独处理

- **是什么**：Claude 拒答时返回 HTTP 200，`stop_reason` 为 `"refusal"`，还会给出类别（如 `cyber`、`bio`）。它不是异常，所以基于异常的回退和基于错误率的监控都发现不了。
- **没有它会怎样**：用户看到一句「抱歉」，而后台监控一切正常。
- **对应你熟悉的**：接口返回 HTTP 200 但 body 里 `code != 0`。只判断 HTTP 状态码就会漏掉业务失败。

处理方法：按 `stop_reason` 分支（不要依赖可能为空的类别字段），再决定是换模型、改写提示还是给用户友好提示。Claude API 还提供服务端回退（beta）：只在安全拒答时触发，最多 3 个备选模型；限流和 5xx 错误仍原样返回，要自己处理。

## 高频面试题

**1. 你负责把线上 Agent 从老模型迁移到新模型，说说你的步骤。**

- 读官方迁移指南，按「请求 / 响应 / 提示词」三类列出需要改的地方。
- 先在评测集上跑新旧两个版本，比较通过率、token 用量、延迟和每任务成本。
- 删掉为老模型写的补丁指令，重新调 effort 和 `max_tokens`。
- 灰度上线，按模型维度监控拒答率和成本，保留一键回滚到旧模型的开关。

**2.（追问）新模型评测通过率更高，但上线后成本涨了 30%，可能是什么原因？**

- 分词器变化，同样的文本 token 更多。
- 默认思考更多或输出更长；老提示里「务必详尽」被新模型照做。
- 提示缓存失效：换模型后缓存要重新建立，最小可缓存长度也可能变。
- 应对：按每任务成本重新评估，调低 effort，精简提示词，检查缓存命中率。

**3.（场景题）车载助手每天 100 万次请求，九成是简单指令。怎么设计模型方案？**

- 规则或小分类器做路由：简单指令走小模型，规划和多轮推理走大模型。
- 路由放在外层，回退放在内层，回退目标是同级别的备用模型或另一家供应商。
- 拒答单独判断，必要时换模型或给友好兜底话术。
- 用评测证明组合方案在整条成本-质量曲线上胜过「全用一个模型调 effort」。

**4.（追问）为什么不直接微调一个小模型来代替大模型？**

- Claude API 没有自助微调，要走 Amazon Bedrock 等渠道，还要用预置吞吐量来部署，成本和维护都重。
- 微调后的模型绑定某个底座版本，底座一升级就要重新训练、重新评测。
- 先用路由、effort 和提示词把成本降下来；只有任务稳定、数据充足、评测证明有收益时才考虑微调或蒸馏。

## 常见误区

1. **「模型名是配置项，换一下就行」**：采样参数、预填、工具选择方式、默认 effort、分词器都可能变化。迁移要按清单检查，并用评测集验证。
2. **「有了回退就高可用了」**：回退只接住异常。拒答是正常返回，必须按 `stop_reason` 单独处理；中间件顺序写反，回退甚至会被路由覆盖掉。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：可失败的假模型 + 路由中间件 + 回退**（取自 `car_route.py`）

```python
# 定义一个可以模拟失败的假模型：带名字标签和是否失败的开关
class Fake(GenericFakeChatModel):
    tag: str = ""
    fail: bool = False
    # 每次生成前先记录自己被调用了
    def _generate(self, messages, stop=None, run_manager=None, **kw):
        CALLS.append(self.tag)
        # 设为失败时，抛出过载异常
        if self.fail:
            raise RuntimeError(f"{self.tag}: 529 overloaded")
        # 否则交给父类，按预设消息返回
        return super()._generate(messages, stop, run_manager, **kw)
# 工厂函数：用一句预设回复造一个假模型，可附带响应元数据
def fake(tag, text, fail=False, meta=None):
    msg = AIMessage(content=text, response_metadata=meta or {})
    return Fake(tag=tag, fail=fail, messages=iter([msg] * 5))
# 路由中间件：看最后一条用户消息的长度，决定用大模型还是小模型
@wrap_model_call
def route(request, handler):
    hard = len(request.messages[-1].content) > 12
    # 用 override 换掉本次请求的模型，再交给下一层
    return handler(request.override(model=big if hard else small))
# 组装 Agent：路由在外层，回退在内层
agent = create_agent(small, tools=[], middleware=[route, ModelFallbackMiddleware(backup)])
```

**三步走**

1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**

- 中间件顺序是 `[route, ModelFallbackMiddleware(backup)]`。写反后，回退换上的 backup 会被内层的 `route` 改回 big，直接报错。
- 用 `request.override(model=...)` 生成新请求再传给 `handler`，不要直接修改 `request` 的属性。
- `messages=iter([...])` 要传迭代器，不是列表；每调用一次消耗一条，所以这里放了 5 条。

**昨日复习（只做第③步）**：第二轮 Day 26 的「幂等支付节点 + 超时重试 + 排空续跑」。只看你自己写的注释重写，重点核对：幂等键来自 `s['order_id']`、`add_node` 里的 `timeout=1.0` 和 `RetryPolicy`、续跑时传 `None`。

## 动手练习（30 分钟）

在 `car_route.py` 上改：

1. 写一个 `@wrap_model_call` 中间件 `refusal_guard`：调用 `handler` 后，如果最后一条消息的 `response_metadata["stop_reason"] == "refusal"`，就改用 `backup` 重新调用一次。让「教我改装电池」的调用链变成 `['big2', 'backup']`。
2. 想一想 `refusal_guard` 应该放在 `route` 的外层还是内层，两种都试一下，对比调用链。
3. 给每个假模型加一个「单价」，在 `CALLS` 之外累计成本，算出两条问题各花多少钱。

## 读后自测

1. 把车载助手的底座从老模型换成新模型，为什么不能只改一个模型名？上线前要检查哪几类东西？见第 7 节「核心概念 1」
2. 什么时候该「换更大的模型」，什么时候该「调高 effort」或「拆成大小模型路由」？用什么数据来做这个决定？见第 7 节「核心概念 2」「核心概念 3」
3. 主模型过载时自动回退到备用模型，这套回退能不能也兜住模型拒答？为什么？见第 6 节「先跑起来」和第 7 节「核心概念 4」

## AI 历史知识点：神经语言模型的缩放定律 Scaling Laws（2020，OpenAI）

**是什么**：一篇论文，系统测量了语言模型的损失（loss）如何随**模型参数量、数据量、训练算力**三者变化。结论是三者都呈幂律关系，有的趋势跨越了七个数量级以上；而网络的宽度、深度等结构细节，在很大范围内影响很小。

**何时由谁提出**：2020 年 1 月 23 日发布，作者是 Jared Kaplan、Sam McCandlish、Tom Henighan、Tom Brown、Dario Amodei 等 10 人，其中 Kaplan 同时署名约翰斯·霍普金斯大学，其余均为 OpenAI。

**为什么重要**：它让「模型做多大、喂多少数据」从凭经验变成可以预测的工程问题。论文还发现大模型更省样本，并提出在固定算力下，最有效的做法是训练很大的模型、用相对适中的数据、在收敛前就停下来。这直接支撑了后来 GPT-3 的规模化路线。论文作者中的多人后来参与创立了 Anthropic。

**和今天的趋势有什么联系**：今天讲的「选大模型还是小模型」，背后就是缩放定律：更大的模型更强，但更贵。前天 OpenAI 那篇用困惑度划「共同上限」的文章，用的也是同一套「能力可以用损失预测」的思路。而对应用开发者来说，结论反过来用：既然大模型的能力边际是可预测的，就该用评测去找「够用的最小模型」。

来源：[Scaling laws for neural language models - OpenAI](https://openai.com/index/scaling-laws-for-neural-language-models/) · [arXiv:2001.08361（ar5iv）](https://ar5iv.labs.arxiv.org/html/2001.08361)

## 参考区（用时查，不必通读）

### 细节速查

**Claude 当前模型（models overview，均为 1M 上下文、128K 最大输出）**

| 模型 | ID | 输入 / 输出（每百万 token） |
|---|---|---|
| Fable 5.2 | `claude-fable-5-2` | $8 / $40 |
| Opus 5.5 | `claude-opus-5-5` | $4 / $20 |
| Sonnet 5.5 | `claude-sonnet-5-5` | $2 / $10 |
| Haiku 5.5 | `claude-haiku-5-5` | 起价 $0.10 / $0.50 |

- 官方建议大多数场景从 Opus 5.5 起步；`xhigh`/`max` 仍不过评测时再换 Fable 5.2。`GET /v1/models` 可以列出可用模型。

**Opus 5.5 迁移要点**

- thinking：省略或 `{"type":"adaptive"}`；`disabled` 和 `{"type":"enabled","budget_tokens":N}` 返回 400。
- effort：`low` / `medium` / `high` / `xhigh` / `max`，默认 `medium`（Opus 5 默认 `high`），建议重做 effort 扫描。
- 采样参数：去掉 `temperature`、`top_p`、`top_k`（自 Opus 4.7 起移除）。
- 预填：以 assistant 消息结尾自 Opus 4.6 起返回 400，改用结构化输出或系统提示。
- `tool_choice`：只接受 `auto`、`none`；强制调用改为 `auto` + strict 工具（每层 schema 需 `"additionalProperties": false`）或结构化输出。
- 思考输出：`max_tokens` 包含思考和正文，`xhigh`/`max` 建议从 64k 起；显示默认 `"omitted"`，需要时设为 `"summarized"`；按 `type` 读取内容块；思考块原样回传，改动会 400。
- 拒答：`stop_reason: "refusal"` + `stop_details.category`（`cyber`、`bio`、`frontier_llm`、`reasoning_extraction`、`general_harms`）。
- 分词器（自 Opus 4.7）：token 数约为原来的 1～1.35 倍；提示缓存最小长度 512 token。
- 思考块绑定模型：回退到读不了它的模型时会被静默丢弃。

**服务端回退（beta）**

```python
client.beta.messages.create(model=..., messages=..., max_tokens=1024,
    fallbacks=[{"model": "claude-opus-4-8"}],   # 或 fallbacks="default"
    betas=["server-side-fallback-2026-07-01"])
```

- 最多 3 个备选，依次尝试；只有安全拒答触发，限流、过载、5xx 原样返回；响应带 `fallback` 内容块，`usage.iterations` 记录每次尝试；路由约 1 小时内保持粘性；Batches API 和 Haiku 5.5 不支持。
- 客户端版本：`Anthropic(middleware=[BetaRefusalFallbackMiddleware([{"model": ...}])])`。子 Agent 和工具内部的调用需要各自配置回退；`reasoning_extraction` 类没有回退，要改写提示。

**LangChain 1.x 模型相关中间件**

- `ModelFallbackMiddleware(first_model, *additional_models)`：参数可以是字符串或 `BaseChatModel`，只在异常时触发。
- `ModelRetryMiddleware(max_retries=2, retry_on=..., on_failure="continue"|"error"|callable, backoff_factor=2.0, initial_delay=1.0, max_delay=60.0, jitter=True)`。
- `ModelRouterMiddleware`（beta，`langchain>=1.5.0`）：`classifier`、`choices={"fast": ModelChoice(model=..., criteria=...)}`、`instructions`。
- `ModelRefusalMiddleware`（`langchain>=1.4.9`）：把拒答转成 `ModelRefusalError` 异常；放在回退中间件之后（内层），回退就能接住它。
- `init_chat_model(..., configurable_fields=("model","model_provider","temperature","max_tokens"), config_prefix="first")`，运行时用 `config={"configurable": {"first_model": ...}}` 切换。
- 测试用假模型：`GenericFakeChatModel(messages=iter([...]))`，同文件还有 `FakeListChatModel`、`FakeMessagesListChatModel`、`ParrotFakeChatModel` 等。

**评测与微调**

- 成功标准要具体、可测、可达、相关；评分优先代码评分，其次 LLM 评分（详细评分细则、先验证可靠性、用不同模型评分），人工评分尽量避免。
- Bedrock 微调（cookbook）：JSONL 格式 `{"system": ..., "messages": [...]}`；超参数 `epochCount`、`batchSize`、`learningRateMultiplier`；`create_model_customization_job(customizationType="FINE_TUNING")`；部署需预置吞吐量。

### 来源说明

- 示例代码已在 langchain 1.5.2 上实测；中间件顺序的结论来自实际报错和调换顺序后的结果。
- Anthropic 官方的「在 Bedrock 上微调 Claude」「模型蒸馏」两篇新闻跳转到 claude.com 后没能打开，相关说法只引用 cookbook 页面；目前哪些模型可在 Bedrock 微调未核实。
- 今日动态里，blog.google AI 板块、docs.bigmodel.cn、Qoder 更新日志、阿里云新闻页没能打开。OpenAI 的 OpenWorks 官方页面没有标注日期，没有收录。

### 延伸学习

- Claude Academy · Building with the Claude API：Module 2「Prompt evaluation」（评测流程、生成测试集、模型评分与代码评分），Module 9「Agents and workflows」中的「Routing workflows」。

### 参考资料

- [Models overview - Claude Docs](https://platform.claude.com/docs/en/models/overview)
- [Migration guide（索引）- Claude Docs](https://platform.claude.com/docs/en/about-claude/models/migration-guide)
- [Opus 5.5 migration guide - Claude Docs](https://platform.claude.com/docs/en/models/opus-5-5/migration-guide)
- [Choosing a model - Claude Docs](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
- [Optimizing for cost and intelligence - Claude Docs](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)
- [Refusals and fallback - Claude Docs](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
- [Define success criteria and build evaluations - Claude Docs](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- [Fine-tuning on Bedrock - Claude Cookbook](https://platform.claude.com/cookbook/finetuning-finetuning-on-bedrock)
- [Models - LangChain Docs](https://docs.langchain.com/oss/python/langchain/models)
- [Built-in middleware - LangChain Docs](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Custom middleware - LangChain Docs](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [Unit testing - LangChain Docs](https://docs.langchain.com/oss/python/langchain/test/unit-testing)
- [fake_chat_models.py - langchain-ai/langchain](https://github.com/langchain-ai/langchain/blob/master/libs/core/langchain_core/language_models/fake_chat_models.py)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
