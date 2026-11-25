# Day 28 · 端侧与移动端 Agent（第二轮：驾驶状态裁剪工具、上云前脱敏、工具检索、语音延迟预算）（2026-11-25）

## 今日 AI 动态（2026-11-25）

### 趋势观察

本期只有 OpenAI 有新的官方发布，两篇都是「通用模型 + Agent 做科研」：GPT-6 不调用专门的生物模型就能设计能结合靶点的蛋白质，业余天文爱好者用 Codex 和 Astra 在公开望远镜数据里找系外行星候选。共同点是通用模型开始替代一部分专用工具链，但两篇都写明了局限：蛋白质设计仍要用专用模型筛选，行星候选还需要多年确认。

### 今日要点

- OpenAI：GPT-6 系列在实验室测试中设计出能结合靶点的蛋白质，322 个小型结合蛋白里 35 个结合成功（10.9%）。
- OpenAI：两位业余天文爱好者用 Codex 和 Astra 检索 Kepler、TESS 数据，找到一批系外行星候选，其中两个信号已被 SETI 研究所的独立搜索复现。

### 分公司动态

#### 1. OpenAI

**[GPT-6 models demonstrate intrinsic protein-design capabilities](https://openai.com/index/gpt-6-models-demonstrate-intrinsic-protein-design-capabilities/)**（2026-11-24）

- OpenAI 称 GPT-6 系列只靠推理和通用代码就能设计蛋白质的结构与序列，设计这一步不调用专门的生物学模型；配套发布了预印本。
- 与合作方 Onava 的湿实验：5 个靶点上 322 个小型结合蛋白有 35 个结合成功；另一组 GPT-6.2 生成的新序列 107 个中 14 个结合；182 个重新设计的 VHH 抗体有 128 个结合 IL-4Rα。
- 局限写得很清楚：候选仍要用 Boltz-2、ESMFold2 等专用模型过滤；完整蛋白质结构预测很弱（平均 TM-score 0.26，ESMFold2 是 0.93）。
- **为什么重要**：这是「通用模型 + 专用工具做验证」的分工样板，和 Agent 设计里「模型生成、外部校验」是同一个思路。

**[Searching for distant planets with Astra](https://openai.com/index/searching-for-distant-planets-with-astra/)**（2026-11-24）

- 两位业余天文爱好者用 Codex 和 Astra 模型检索公开望远镜数据。一人检索约 10 万颗 Kepler 恒星，标出 200 多个信号（其中 10 个属于已知行星），两个主要候选也被 SETI 研究所的独立搜索发现。
- 另一人检索覆盖约 1900 万颗恒星的 TESS 数据，第一轮得到 49 个候选，最强的一个已交给 TESS 团队进一步分析。
- 文中强调确认可能需要数月到数年。
- **为什么重要**：展示了非专业用户靠 Code Agent 完成「写数据管线—筛信号」的长任务，门槛从「会写天文软件」降到「会描述需求」。

### 动态来源

- https://openai.com/index/gpt-6-models-demonstrate-intrinsic-protein-design-capabilities/
- https://openai.com/index/searching-for-distant-planets-with-astra/

---

## 读前三问

1. 车在行驶中，怎么让模型「根本看不到」播放视频这类工具？为什么不只靠 prompt 里写「行驶中不要播视频」？
2. 车载 Agent 的模型在云端，用户说了手机号，上云前怎么处理？脱敏之后端侧拨号工具还拿得到真号码吗？
3. 一台车上有上百个 App 能力，全部当工具塞给模型会出什么问题？有哪两种「按需加载工具」的做法？

## 从一个场景开始

第一轮 Day 28，你的车载助手已经能把空调、导航、音乐包装成工具，端侧执行，云端模型做决策。上线评审时被问了三个问题：

- 安全组：「车速 80 时用户说『放个电影』，模型调用了 `play_video`，谁负责？」你说 prompt 里写了不许，对方回答：prompt 不是权限。
- 法务：「用户说『给 138xxxx 回电话』，这个号码原样发到了云端模型厂商？」
- 架构师：「后面要接入车控、充电、维保、社交几十个 App，每个 App 十几个能力，工具描述加起来几万 token，首字延迟和选错工具都会变差。」

三个问题有一个共同答案：**模型每次能看到什么（工具、文字），要在发给模型之前由端侧代码决定**。今天就讲这一层。

## 一句话总结

端侧 Agent 要在请求上云之前做三件事：按车辆状态裁剪可见工具，把隐私数据脱敏，工具太多时按需检索；权限靠代码而不是 prompt，同时给语音交互留出延迟预算。

## 先跑起来

```python
# car_cloud_guard.py —— 车载 Agent：按驾驶状态裁剪工具 + 上云前脱敏手机号（langchain 1.5，假模型离线可跑）
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware, wrap_model_call
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

@tool
def set_ac(temp: int) -> str:
    """设置空调温度（摄氏度）"""
    return f"空调已设为 {temp}°C"
@tool
def play_video(title: str) -> str:
    """在中控屏播放视频"""
    return f"正在播放 {title}"

@dataclass
class Car:
    driving: bool  # 车辆是否在行驶

SEEN = []  # 记录云端模型每次收到的最后一条用户消息
class CloudModel(GenericFakeChatModel):
    def bind_tools(self, tools, **kw): return self  # 假模型不真绑定
    def _generate(self, messages, stop=None, run_manager=None, **kw):
        SEEN.append(messages[-1].content)
        return super()._generate(messages, stop, run_manager, **kw)

@wrap_model_call
def driving_tools(request, handler):
    if request.runtime.context.driving:  # 行驶中：视频工具对模型不可见
        tools = [t for t in request.tools if t.name != "play_video"]
        request = request.override(tools=tools)
    print("模型可见工具:", [t.name for t in request.tools])
    return handler(request)

phone = PIIMiddleware("phone", detector=r"1[3-9]\d{9}", strategy="redact")
model = CloudModel(messages=iter([AIMessage(content="好的，已为你处理。")] * 4))
agent = create_agent(model, tools=[set_ac, play_video],
                     middleware=[phone, driving_tools], context_schema=Car)

for driving in (True, False):
    agent.invoke({"messages": [{"role": "user", "content": "给13812345678回电话，再放个电影"}]},
                 context=Car(driving=driving))
print("云端收到的原话:", SEEN)
```

运行：`pip install langchain langgraph && python car_cloud_guard.py`

真实输出（langchain 1.5.2 / langgraph 1.2.24）：

```text
模型可见工具: ['set_ac']
模型可见工具: ['set_ac', 'play_video']
云端收到的原话: ['给[REDACTED_PHONE]回电话，再放个电影', '给[REDACTED_PHONE]回电话，再放个电影']
```

该看的地方：

- 第一行是行驶中：`play_video` 不在工具列表里，模型想调也调不了。第二行驻车，两个工具都在。
- 第三行：不管行驶与否，云端模型收到的都是 `[REDACTED_PHONE]`，真号码没离开车机。
- 另外实测：`agent.invoke` 返回的 `messages[0]` 也已被改成 `[REDACTED_PHONE]`。PII 中间件改的是**状态本身**，不只是发出去的副本，这一点决定了核心概念 2 的设计。

## 核心概念

### 1. 按车辆状态裁剪工具（动态工具集）

- **是什么**：每次调用模型前，端侧代码根据车速、档位、账号等运行时信息，从全部工具里挑出这一刻允许的子集，模型只看得到子集。
- **没有它会怎样**：只在 prompt 里写「行驶中不许播视频」，模型偶尔会违反，用户也能用「忽略前面的规定」绕过去。工具不在列表里，模型就无从调用。
- **对应你熟悉的**：Android 的运行时权限 + 车规里的「行驶中禁用视频」(UX Restrictions)，限制由系统强制，不由 App 自觉。

代码里 `request.runtime.context.driving` 来自 `invoke(..., context=Car(...))`，是本次调用的只读上下文，不进对话历史。`request.override(tools=...)` 只影响这一次模型调用。注意它是**第一道**闸：执行工具时仍要在端侧再查一次车速（模型可能在驻车时决定、执行时已在行驶）。

### 2. 上云前脱敏：数据最小化

- **是什么**：把发给云端模型的文本里的手机号、车牌、精确位置等替换成占位符，真实值只留在车机上。
- **没有它会怎样**：隐私数据进了模型厂商的日志和你自己的 trace 系统，合规审计过不去。
- **对应你熟悉的**：Android 里「只申请粗略位置」的思路，云端只拿完成任务所需的最小信息。

难点在「脱敏后怎么还能拨号」。实测已证明 `PIIMiddleware` 的 `redact` 会改写状态里的原消息，端侧也拿不到原值了。常见做法：端侧自己维护一个映射表（如 `CONTACT_1 → 13812345678`），发给云端的是 `CONTACT_1`，模型调用 `call(contact="CONTACT_1")`，端侧工具执行时再查表还原。真号码从不出车。

### 3. 工具太多：按需检索工具

- **是什么**：不把上百个工具一次性发给模型，而是先给一小组常用工具 + 一个「找工具」的能力，需要时再加载具体定义。
- **没有它会怎样**：Claude 文档给的数据是，多个 MCP server 的工具定义能占约 55k token；可选工具超过 30–50 个后，选对工具的准确率会下降。工具检索通常能把这部分减少 85% 以上。
- **对应你熟悉的**：Android 的按需加载模块（Dynamic Feature Module），用到时才下载。

两条路线：

| 做法 | 在哪里选 | 代表 |
|---|---|---|
| 服务端工具检索 | 模型自己搜索被标记为延迟加载的工具 | Claude 的 tool search tool + `defer_loading` |
| 先用小模型挑工具 | 你的代码里，主模型调用前 | LangChain `LLMToolSelectorMiddleware` |

和概念 1 的区别：概念 1 是**权限**（不许用的绝对不给），概念 3 是**相关性**（可以用但这次用不上的先不给）。两者叠加时，先按权限裁剪，再在剩下的里面检索。

### 4. 语音场景的延迟预算

- **是什么**：把「用户说完话 → 听到第一个字」的总时长（例如目标 1 秒内）拆给各环节：端侧识别、上云网络、模型首 token、工具执行、语音合成。
- **没有它会怎样**：每个环节单看都不慢，加起来 3 秒，用户以为没听见，又说一遍，导致重复执行。
- **对应你熟悉的**：Android 的 16ms 帧预算，超了就掉帧。

常用手段：简单指令（「空调调到 22 度」）走端侧小模型或规则，不上云；上云的请求用流式输出，攒够一句就送 TTS 播报；先播「好的，正在为你导航」，后台再执行慢工具。今天的工具裁剪和检索本身也在帮忙：工具定义越少，首 token 越快。

## 高频面试题

**1. 行驶中禁用某些功能，你会怎么实现？**

- 第一道：每次模型调用前按车辆状态过滤工具列表，模型看不到被禁的工具。
- 第二道：工具执行时在端侧再校验一次车辆状态，防止「决策时驻车、执行时行驶」。
- 不靠 prompt 约束，prompt 只用来解释为什么做不了，让回复更友好。
- 被拒的调用要记日志，用于评测和合规审计。

**2. 追问：脱敏后模型看不到真实手机号，怎么完成「给张三打电话」？**

- 端侧维护占位符映射表，上云文本只含 `CONTACT_1` 这类占位符。
- 模型输出的工具参数也用占位符，端侧工具执行前查表还原。
- 注意框架的脱敏中间件可能直接改写状态，原值要在脱敏前自己留存。
- 映射表只在本次会话有效，会话结束即清，不写入云端记忆。

**3. 场景题：车机要接入 200 个 App 能力，你怎么设计工具层？**

- 先按权限和车辆状态裁剪，再按相关性检索，常用的十个左右常驻。
- 用 Claude tool search（`defer_loading`）或小模型选工具，避免工具定义占满上下文。
- 工具名和描述按 App 分组、写清楚适用场景，检索才准。
- 用评测集统计「选错工具率」和首 token 延迟，对比引入检索前后。

**4. 端云协同下，哪些请求不该上云？**

- 高频、确定的车控指令（温度、车窗）：端侧规则或小模型，延迟低、断网可用。
- 含敏感信息且无法脱敏的请求：只在端侧处理或明确拒绝。
- 涉及安全的动作（行驶中的车控）：最终决定权在端侧代码，云端只给建议。
- 复杂多步任务（规划行程、比价充电站）才上云。

## 常见误区

- **「prompt 里写了禁止，就等于做了权限控制。」** prompt 会被违反、被注入绕过。权限必须在工具列表和工具执行两处由代码强制。
- **「用了 PII 中间件就解决隐私问题了。」** 它只管匹配得上的格式，人名、地址常漏；而且实测会改写状态，端侧业务若要原值必须自己先留存。脱敏规则要配评测集持续检查漏报。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：驾驶状态裁剪工具 + 上云前脱敏**（取自 `car_cloud_guard.py`）

```python
# 定义运行时上下文：记录车辆是否在行驶
@dataclass
class Car:
    driving: bool
# 用 wrap_model_call 包住每一次模型调用
@wrap_model_call
def driving_tools(request, handler):
    # 从运行时上下文读取行驶状态
    if request.runtime.context.driving:
        # 行驶中：从工具列表里去掉播放视频
        tools = [t for t in request.tools if t.name != "play_video"]
        # 用 override 生成只含剩余工具的新请求
        request = request.override(tools=tools)
    # 交给下一层，真正调用模型
    return handler(request)
# 用正则识别手机号，命中就替换成占位符
phone = PIIMiddleware("phone", detector=r"1[3-9]\d{9}", strategy="redact")
# 组装 Agent：先脱敏再裁剪工具，并声明上下文类型
agent = create_agent(model, tools=[set_ac, play_video],
                     middleware=[phone, driving_tools], context_schema=Car)
# 调用时通过 context 传入本次的车辆状态
agent.invoke({"messages": [{"role": "user", "content": "给13812345678回电话"}]},
             context=Car(driving=True))
```

**三步走**

1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**

- 上下文从 `request.runtime.context` 读，不是 `request.state`；`create_agent` 必须传 `context_schema=Car`，`invoke` 时用关键字参数 `context=`。
- `request.override(tools=tools)` 返回新请求，要赋值回 `request`，否则过滤不生效。
- 自定义检测器在本版本（langchain 1.5.2）要传正则**字符串**；传 `re.compile(...)` 实测报 `'re.Pattern' object is not callable`。

**昨日复习（只做第③步）**：第二轮 Day 27 的「可失败的假模型 + 路由中间件 + 回退」。只看你自己写的注释重写，重点核对：中间件顺序 `[route, ModelFallbackMiddleware(backup)]`、`request.override(model=...)`、`messages=iter([...])`。

## 动手练习（30 分钟）

在 `car_cloud_guard.py` 上改：

1. 给 `Car` 加一个字段 `speed: int`，把过滤条件改成 `speed > 0`（行驶判断更贴近真实车机）。
2. 在 `play_video` 工具函数内部再加一次检查：如果一个全局变量 `CURRENT_SPEED > 0`，返回「行驶中无法播放视频」。这是第二道闸。
3. 不用 `PIIMiddleware`，自己写一个 `@before_model` 或 `@wrap_model_call` 中间件：把手机号替换成 `CONTACT_1`，原号码存进字典 `VAULT`。打印 `VAULT` 和 `SEEN`，确认云端只看到占位符、端侧还留着真号码。

## 读后自测

1. 车在行驶中，怎么让模型「根本看不到」播放视频这类工具？为什么不只靠 prompt？——见「核心概念 1」
2. 上云前怎么处理手机号？脱敏后端侧拨号工具还拿得到真号码吗？——见「先跑起来」和「核心概念 2」
3. 上百个 App 能力全部当工具会出什么问题？有哪两种按需加载工具的做法？——见「核心概念 3」

## AI 历史知识点：联邦学习 Federated Learning（2016，Google）

**是什么**：一种让数据留在手机上、只把本地算出的模型更新上传汇总的训练方法。论文提出的核心算法是反复做「设备本地训练 → 服务器对模型参数取平均」，后来被称为 FedAvg。

**何时由谁提出**：H. Brendan McMahan、Eider Moore、Daniel Ramage、Seth Hampson、Blaise Agüera y Arcas（Google），论文《Communication-Efficient Learning of Deep Networks from Decentralized Data》，2016 年 2 月首次发布于 arXiv，发表于 AISTATS 2017。

**为什么重要**：它第一次系统地处理了手机数据的两个现实问题：每台设备的数据量差异大、分布也不一样（非独立同分布），以及网络通信是最大成本。论文报告，与同步随机梯度下降相比，所需通信轮数减少 10–100 倍。

**和今天的联系**：今天讲的「真号码不出车机、云端只拿最小信息」和联邦学习是同一种思想：把计算推到数据所在的设备上，而不是把数据搬到计算那里。端侧 Agent 解决的是推理阶段的隐私，联邦学习解决的是训练阶段的隐私。

来源：[arXiv:1602.05629](https://arxiv.org/abs/1602.05629)

## 参考区（用时查，不必通读）

### 细节速查

**PIIMiddleware（`from langchain.agents.middleware import PIIMiddleware`）**

| 参数 | 说明 |
|---|---|
| 第一个参数 pii_type | 内置：`email`、`credit_card`、`ip`、`mac_address`、`url`；也可写自定义名（如 `phone`） |
| `strategy` | `redact`（默认，替换为 `[REDACTED_类型]`）、`mask`（部分遮盖）、`hash`（确定性哈希）、`block`（抛异常） |
| `detector` | 正则字符串或返回 `list[PIIMatch]` 的函数；文档还写了可传编译后的正则，但 1.5.2 实测不支持 |
| `apply_to_input` | 默认 True，检查用户消息 |
| `apply_to_output` | 默认 False，检查模型输出（1.3.2 起含流式输出） |
| `apply_to_tool_results` | 默认 False，检查工具返回 |

**动态工具（context-engineering 文档的写法）**

- 读状态：`request.state` / `request.messages`；读运行时上下文：`request.runtime.context.<字段>`；读长期存储：`request.runtime.store`
- 修改只对本次模型调用生效，不写回状态：`request.override(tools=...)`

**LLMToolSelectorMiddleware**：参数 `model`（选工具用的模型，默认主模型）、`max_tools`、`always_include`（常驻工具名列表，不计入 `max_tools`）。文档建议 10 个以上工具时使用。

**Claude tool search tool**

| 项 | 值 |
|---|---|
| 工具类型 | `tool_search_tool_regex_20251119`（正则，模式 ≤200 字符）、`tool_search_tool_bm25_20251119`（自然语言，≤500 字符） |
| 延迟加载 | 工具定义上加 `"defer_loading": true`；至少保留一个非延迟工具；搜索工具本身不能延迟 |
| 每次返回 | 默认 5 个，`limit` 可调 |
| 上限 | 每个请求最多 10,000 个延迟工具 |
| 注意 | 每次请求仍要发送全部工具定义；延迟工具不能带 `cache_control`；Managed Agents 不支持 |
| 何时用 | 10 个以上工具、定义超过 10k token、聚合多个 MCP server 时 |

### 来源说明

- 「联邦学习」为非 LangChain/Claude 来源（原始论文）。
- 本期 AI 动态：Anthropic、Google 博客、智谱、DeepSeek、Qoder 等多个官方页面本次运行未能打开（抓取授权超时），这些厂商可能有遗漏；能打开的 Qwen 更新日志、Meta AI 博客在时间窗口内没有新内容。
- LangChain streaming 文档页未能打开，概念 4 的流式做法未附代码。

### 延伸学习

- Claude 官方文档：Tool search tool、Manage tool context（见下方链接）
- LangChain 文档：Context engineering 中的「Selecting tools」一节

### 参考资料

- https://docs.langchain.com/oss/python/langchain/middleware/built-in
- https://docs.langchain.com/oss/python/langchain/context-engineering
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool
