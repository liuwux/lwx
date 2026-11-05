# Day 8 · 反思与自我纠错（第二轮：谁来判定做完了、反馈要能照着改、两种重试别混）（2026-11-05）

## 今日 AI 动态（2026-11-05）

### 趋势观察

本期能核实的官方更新只有一条：Claude 的企业连接器支持由组织统一授权，员工第一次登录时自动连上。员工各自登录、各自授权的方式，正在变成管理员按角色统一下发。推测：企业 Agent 落地时，最先卡住的往往不是模型，而是身份和权限，平台方正在把这一层做成现成功能。

### 今日要点

- Claude：Team / Enterprise 组织可以为连接器开启「企业托管认证」，员工首次登录时自动获得访问权限；新增不依赖身份提供商、由 Claude 签发身份的 beta 方式。

### 分公司动态

#### 1. Anthropic / Claude

**[Auto-connect your organization's apps to Claude](https://support.claude.com/en/articles/15537633-auto-connect-your-organization-s-apps-to-claude)**（帮助中心文章，2026-11-05 打开时标注为当天更新）

管理员在组织设置里为某个连接器授权一次，选好开放给哪些角色、Claude 可以申请哪些权限，员工首次登录就能直接使用，不必逐个登录各个应用。身份来源有两种：一种是通过身份提供商的 SSO（已正式可用，目前支持 Okta）；另一种是 beta 的「Claude 签发」，由 Claude 向连接器声明当前用户，按邮箱匹配，首批支持 Amplitude、Atlassian、Datadog、Figma、Linear、Notion、Supabase。用 Claude 签发方式时，把人移出组织后，他已拿到的访问要等过期才失效，所以文档提醒管理员还要去对应服务里删掉这个人。

**为什么重要**：企业 Agent 要调内部系统，「这个工具以谁的身份调用」是面试和落地都绕不开的问题；这里给出了一个现成的统一授权方案，包括撤销权限时要注意的地方。

### 动态来源

- [Auto-connect your organization's apps to Claude - Claude Help Center](https://support.claude.com/en/articles/15537633-auto-connect-your-organization-s-apps-to-claude)

---

> 第二轮说明：第一轮 Day 8 讲过反思的三个层次、什么时候该加反思、验证器从强到弱的排序、模型变强后别过度验证、重试次数与退避。今天不重复，改讲三件进阶的事：「做完了」由谁判定；验证器的反馈怎样写模型才改得对；传输层重试和带反馈的语义重试为什么要分开。

## 读前三问

1. 让模型「检查一下再交付」和用 Stop hook / 中间件强制检查，区别在哪？各适合什么情况？
2. 验证不通过时，回给模型的消息应该写什么？只写「错了，再试一次」会怎样？
3. `ModelRetryMiddleware` 能不能用来做自我纠错？它和「验证失败后带着反馈重写」有什么不同？

## 从一个场景开始

司机说：「从上海开到黄山，给我一个充电方案。」车满电能跑 300km。

第一版 Agent 在提示词里写了「输出前请自查续航」。大多数时候没问题，但偶尔会输出「460km 直达，不用充电」，而且它自查后还会补一句「已确认续航充足」。

第二版加了重试：结果不对就再调一次模型。可是重试时输入和上次一模一样，模型多半再给一个差不多的答案；有一次碰上网络超时，重试反而救回来了。

测试同学指出了两个问题：第一，「自查通过」只是模型自己说的，程序并没有检查；第二，网络超时和答案算错是两种完全不同的失败，不能用同一个重试来处理。

今天讲三件事：验证由谁来做，验证失败后怎么反馈，以及两种重试怎么分开。

## 一句话总结

自我纠错要靠模型之外的验证器来判断是否做完，没通过时把具体错误回传给模型，让它照着改；网络和限流这类偶发错误交给带退避的重试；两种循环都要设上限，验证器本身不能被 Agent 改动。

## 先跑起来

用 LangChain 的 `after_model` 中间件挂一个确定性验证器：模型给出最终方案后，程序逐段检查行驶距离是否超过续航；不通过就把具体哪一段、超了多少写成一条消息，跳回模型重写。用脚本化的假模型，不需要 API Key。

```python
# verify_loop.py —— 外部验证器不通过就把具体错误打回模型重写（脚本化假模型，无需 API Key）
from langchain.agents import AgentState, create_agent
from langchain.agents.middleware import after_model
from langchain.messages import AIMessage, HumanMessage
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel

RANGE_KM = 300  # 满电续航（演示用固定值）

class RetryState(AgentState):
    verify_tries: int

def check_plan(text: str) -> list[str]:
    """确定性验证器：每一段行驶距离都不能超过续航。"""
    legs = [int(x) for x in text.split("|")[1].split(",")]
    return [f"第{i+1}段 {d}km 超过续航 {RANGE_KM}km" for i, d in enumerate(legs) if d > RANGE_KM]

@after_model(state_schema=RetryState, can_jump_to=["model"])
def verify_plan(state: RetryState, runtime) -> dict | None:
    last = state["messages"][-1]
    if last.tool_calls:                      # 还在调工具，不是最终答案，不验
        return None
    errors = check_plan(last.content)
    tries = state.get("verify_tries", 0)
    if not errors or tries >= 2:             # 通过，或已达重试上限：放行结束
        return None
    feedback = "验证未通过：" + "；".join(errors) + "。请只修改出错的路段后重新输出。"
    return {"messages": [HumanMessage(feedback)], "verify_tries": tries + 1, "jump_to": "model"}

script = iter([
    AIMessage("上海→黄山 460km，不充电|460"),
    AIMessage("上海→杭州服务区充电→黄山|180,280"),
])
agent = create_agent(GenericFakeChatModel(messages=script), tools=[], middleware=[verify_plan])
out = agent.invoke({"messages": [{"role": "user", "content": "从上海开到黄山，给我一个充电方案"}]})
for m in out["messages"][1:]:
    print(f"{m.type:>5}: {m.content}")
print("verify_tries =", out.get("verify_tries", 0))
```

**运行**：`pip install -U langchain langgraph && python verify_loop.py`

**实测输出**（2026-11-05，Python 3.13.18，langchain 1.4.9，langgraph 1.2.19）：

```
   ai: 上海→黄山 460km，不充电|460
human: 验证未通过：第1段 460km 超过续航 300km。请只修改出错的路段后重新输出。
   ai: 上海→杭州服务区充电→黄山|180,280
verify_tries = 1
```

**该看什么**：
- 中间那条 `human` 消息不是司机说的，是验证器写的。它点名了哪一段、超了多少，模型照着改就行。
- 「做完了没有」由 `check_plan` 判断，模型自己说「已确认」不算数。`jump_to: "model"` 让流程回到模型，而不是直接结束。
- `verify_tries` 存在 Agent 状态里，到 2 次就放行。真实项目里，放行时还应该标记「未通过验证」，交给人处理，不能悄悄交出一个错误方案。

## 核心概念

### 1. 谁来判定「做完了」：从模型自己说了算到框架强制

- **是什么**：Claude Code 最佳实践把检查从松到严分成四级：写在提示词里，由模型自己判断；设一个 goal，每轮由另一个模型对照检查；用 Stop hook，脚本没通过就不让本轮结束；交给没参与编写的人或子 Agent 复查。
- **没有它会怎样**：模型的「全部通过」只是一句话。官方原话的意思是：要的是证据，即跑了什么命令、输出是什么。
- **对应你熟悉的**：CI 门禁。开发者说「我本地测过了」没用，流水线不绿就合不进去。

Stop hook 的做法：Agent 想结束本轮时运行你的脚本，脚本以退出码 2 退出，stderr 里的内容就作为「为什么还不能结束」回给模型，模型接着干。LangChain 里对应的是 `after_model` 加 `jump_to: "model"`，也就是今天的示例。

### 2. 反馈要能照着改：说清哪错了，而不是「再试一次」

- **是什么**：验证器回传的消息要写清三件事：哪里错了、错多少、改动范围（例如「只改出错的路段」）。
- **没有它会怎样**：只说「错了」，模型只能猜，常常推翻整份方案重写，把原本对的部分也改坏，或者原样再交一次。
- **对应你熟悉的**：表单校验。`TextInputLayout.setError("手机号需 11 位")` 比弹一个「输入有误」有用得多。

LangChain 结构化输出的 `ToolStrategy` 也是这个思路：`handle_errors` 默认为 `True`，校验失败时会把具体的校验错误放进工具消息回给模型，末尾附上「Please fix your mistakes.」。也可以传一个函数，按错误类型生成不同的提示。

### 3. 两种重试别混：传输层重试和语义重试

- **是什么**：`ModelRetryMiddleware` / `ToolRetryMiddleware` 做的是传输层重试：同样的输入再发一次，用指数退避，默认最多重试 2 次。语义重试是把验证器的反馈加进对话，让模型改输出。
- **没有它会怎样**：用传输层重试处理「答案算错」，输入不变，模型多半给出同样的错答案，还多付几次钱；反过来，把网络超时当成答案错误，给模型发一句「请修正」，模型会去改一个本来没问题的答案。
- **对应你熟悉的**：OkHttp 的 `retryOnConnectionFailure` 只管连接失败；接口返回 `code=4001 参数错误`，就该改参数，而不是重发。

工程上分两层：外层用重试中间件兜住网络、限流等偶发错误；内层用验证器加反馈处理内容错误。两层各自有上限。

### 4. 验证器不能被 Agent 改：防止「改考卷」

- **是什么**：Agent 可能为了让检查通过，去改检查本身，例如修改测试、写死期望值。Claude 提示词最佳实践专门有一段提示：不要为了通过测试写死数值；测试有问题就报告，不要绕过去。
- **没有它会怎样**：验证全部通过，但功能是错的，比不验证更危险，因为大家都以为它是对的。
- **对应你熟悉的**：单元测试由 QA 维护，开发者的 MR 不能改断言。

Claude Academy 的做法：修 bug 时先写一个会失败的测试并提交，再要求 Agent 在不改这个测试的前提下修复，可以用 hook 强制执行。另外两条：复查用全新的会话或子 Agent，它没看过原来的推理过程，不会被带偏；让审查者「只报影响正确性的问题，跳过风格」，否则它总能挑出点什么，改到最后反而过度设计。

## 高频面试题

**Q1：「让模型输出前自查」和「程序验证 + 打回」怎么选？**
- 有确定性检查（测试、编译、规则）时，优先用程序验证，并由框架强制执行。
- 只能靠语义判断时，用独立的审查者（新会话或子 Agent）复查，不让写的人自己审。
- 新模型自己就会检查，多加「请自查」可能导致过度验证，增加 token 和延迟。
- 无论哪种，都要求给出证据：命令和输出，而不是一句「已通过」。

**Q2（追问）：验证一直不过，Agent 来回改个没完，怎么办？**
- 设上限：自己在状态里计数；Claude Code 的 Stop hook 连续拦截 8 次后会强制结束本轮（可通过环境变量调整）。
- 看有没有进展：错误数量没减少，或者两次输出一样，就提前停止。
- 到上限后转人工，并标记为「未通过验证」，不能静默交付。
- 回头检查反馈是否具体，以及任务本身是否根本做不到（例如续航不够又没有充电站）。

**Q3（场景题）：车机充电规划 Agent 偶尔给出超续航的方案，怎么加自我纠错？**
- 写一个确定性验证器：每段距离不超过续航减去安全余量，充电站必须真实存在。
- 挂在 `after_model`：不通过就把具体路段和超出的公里数回给模型，跳回重写。
- 超时和限流交给 `ModelRetryMiddleware`，和内容错误分开处理。
- 最多 2 次；还不过，就在屏幕上提示「请手动规划」，并上报日志。

**Q4（追问）：怎么防止 Agent 为了通过验证去改测试或写死结果？**
- 验证器放在 Agent 改不到的地方：hook、中间件，或者只读文件。
- 先提交会失败的测试，再要求修复时不能改测试。
- 提示词里写明：测试有问题就报告，不要绕过去。
- 用一个没参与编写、只有只读权限的审查者复查 diff。

## 常见误区

- **「反思循环越多轮越好」**：没有新信息的反思，大多只是换个说法再写一遍。决定效果的是验证器能不能给出具体、可执行的错误信息。
- **「加了重试中间件就等于有自我纠错」**：重试中间件只是把同样的请求再发一次，解决的是网络和限流问题。答案错了，必须把错误信息带回去，模型才会改。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：确定性验证器，以及验证失败时跳回模型的 `after_model` 中间件**（取自 `verify_loop.py`）

```python
# 在默认的 Agent 状态上加一个字段，记录已经打回重写了几次
class RetryState(AgentState):
    verify_tries: int

# 验证器：取出每段距离，返回所有超续航路段的错误描述
def check_plan(text: str) -> list[str]:
    # 方案格式是「说明|距离1,距离2」，取竖线后面的部分转成整数列表
    legs = [int(x) for x in text.split("|")[1].split(",")]
    # 逐段比较，超过续航的生成一条带序号和公里数的错误
    return [f"第{i+1}段 {d}km 超过续航 {RANGE_KM}km" for i, d in enumerate(legs) if d > RANGE_KM]

# 模型每次返回后运行；声明这个钩子可以跳回模型节点
@after_model(state_schema=RetryState, can_jump_to=["model"])
def verify_plan(state: RetryState, runtime) -> dict | None:
    # 取模型刚返回的那条消息
    last = state["messages"][-1]
    # 还在调用工具，说明不是最终答案，不验证
    if last.tool_calls:
        return None
    # 运行验证器，并读出已经重试的次数
    errors = check_plan(last.content)
    tries = state.get("verify_tries", 0)
    # 验证通过，或者重试已达上限，就不干预，让流程结束
    if not errors or tries >= 2:
        return None
    # 把具体错误拼成反馈，告诉模型只改出错的部分
    feedback = "验证未通过：" + "；".join(errors) + "。请只修改出错的路段后重新输出。"
    # 追加反馈消息、计数加一，并跳回模型重写
    return {"messages": [HumanMessage(feedback)], "verify_tries": tries + 1, "jump_to": "model"}
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 要想跳回模型，装饰器里必须声明 `can_jump_to=["model"]`；跳转写在返回字典的 `"jump_to"` 键里，不是单独返回一个字符串。
- 自定义字段要通过 `state_schema=RetryState` 注册，读的时候用 `state.get("verify_tries", 0)`，第一次读时这个字段还不存在。
- 返回 `None` 表示「不改状态、不跳转」，不是报错。

**昨日复习（只做第③步）**：第二轮 Day 7 的「构造带计划的脚本，创建挂 TodoListMiddleware 的 Agent，并从状态读出计划」（`plan_as_state.py`）。只看你自己写的注释重写，重点核对：`todos()` 返回 `{"todos": [...]}`；每项的键是 `content` 和 `status`；最后从 `out["todos"]` 读计划。

## 动手练习（30 分钟）

在 `verify_loop.py` 的基础上：

1. 把脚本里第二条 AIMessage 也改成超续航（如 `|350,110`），运行后观察：第 2 次打回后第三条还错，会不会停下？如果脚本只准备了两条输出，会发生什么？
2. 把上限放行改成：到达上限时追加一条 `AIMessage("方案未通过续航校验，请手动规划。")` 并 `jump_to: "end"`（记得把 `can_jump_to` 改成 `["model", "end"]`）。
3. 再挂一个 `ModelRetryMiddleware(max_retries=2)`，想一想：它和 `verify_plan` 各自拦住的是哪类失败？

## 读后自测

1. 让模型「检查一下再交付」和用 Stop hook / 中间件强制检查，区别在哪？各适合什么情况？——见「核心概念」第 1 节
2. 验证不通过时，回给模型的消息应该写什么？只写「错了，再试一次」会怎样？——见「核心概念」第 2 节
3. `ModelRetryMiddleware` 能不能用来做自我纠错？它和「验证失败后带着反馈重写」有什么不同？——见「核心概念」第 3 节

## AI 历史知识点：DeepSeekMath 与 GRPO（2024，DeepSeek-AI）

**是什么**：DeepSeekMath 是一个专攻数学推理的 7B 开源模型，在 DeepSeek-Coder-Base-v1.5 7B 的基础上，用从 Common Crawl 筛出的 1200 亿 token 数学数据继续预训练。论文同时提出了 GRPO（组相对策略优化），这是 PPO 的一个变体：对同一道题采样一组答案，用这组答案的平均奖励作为基线，比平均好的答案得到加强，因此不需要再单独训练一个价值模型（critic）。

**何时由谁提出**：DeepSeek-AI 的邵智宏（Zhihong Shao）、王培毅（Peiyi Wang）、朱琪豪（Qihao Zhu）等人，2024 年 2 月 5 日发布于 arXiv。

**为什么重要**：不依赖外部工具和投票时，模型在 MATH 基准上达到 51.7%，64 次采样做自洽投票后达到 60.9%。GRPO 去掉了 PPO 中占用大量显存和算力的价值模型，大大降低了强化学习的成本，后来成为 DeepSeek 推理模型训练中的关键方法。

**和今天的趋势有什么联系**：GRPO 的奖励来自「答案对不对」这类可以自动判定的信号，和今天讲的「用确定性验证器判定做完没有」是同一个思路：能被程序验证的任务，就能让模型反复尝试、反复改进。区别在于，今天是在推理时把错误回传给模型改，GRPO 是在训练时用验证结果更新模型参数。

**原始来源**：
- [DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models - arXiv:2402.03300（2024-02-05）](https://arxiv.org/abs/2402.03300)
- [同上，HTML 全文 v3](https://arxiv.org/html/2402.03300v3)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| `after_model` 跳转 | `@after_model(can_jump_to=[...])` 或类方法上加 `@hook_config(can_jump_to=[...])`；返回 `{"jump_to": "model" / "tools" / "end", ...}`，可同时带状态更新；`"end"` 跳到结束（或第一个 `after_agent`），`"model"` 跳到模型节点（或第一个 `before_model`） |
| 自定义状态 | `state_schema=` 传 `AgentState` 子类；返回的字典按图的 reducer 合并，`messages` 是追加 |
| `ModelRetryMiddleware` / `ToolRetryMiddleware` | `max_retries=2`（共 3 次）、`backoff_factor=2.0`、`initial_delay=1.0`、`max_delay=60.0`、`jitter=True`（±25%）；`on_failure`：`'continue'`（把错误交给模型）/ `'error'`（抛出）/ 函数；`ToolRetryMiddleware` 另有 `tools=` 过滤；`'return_message'`、`'raise'` 已弃用 |
| `retry_on` 默认 | langchain≥1.3.16：重试可重试的模型错误和未分类异常，标为不可重试的模型错误不再重试 |
| `ToolErrorMiddleware` | langchain≥1.3.14；把工具异常转成错误 `ToolMessage`，本身不重试；与重试一起用时，`ToolRetryMiddleware` 放在它后面并设 `on_failure="error"` |
| `ToolStrategy(handle_errors=)` | 默认 `True`（默认模板 + "Please fix your mistakes."）；可传 `str`、异常类型、异常元组、`Callable[[Exception], str]`、`False` |
| Claude Code Stop hook | 退出码 2 阻止结束本轮，stderr 作为原因回给 Claude；或输出 JSON `{"decision": "block", "reason": "..."}`；`hookSpecificOutput.additionalContext` 可给非错误反馈；连续拦截 8 次且中间没有工具调用，就强制结束本轮，用 `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` 调整；`onFailure` 对 Stop 无效；子 Agent 里的 Stop 会被转成 `SubagentStop` |
| 官方示例 | `"Stop": [{"hooks": [{"type": "command", "command": "pnpm tsc --noEmit 1>&2 \|\| exit 2"}]}]` |
| 检查强度四级 | 提示词 → `/goal`（另一个模型每轮检查）→ Stop hook（框架强制）→ `/code-review`（新的子 Agent 复查） |
| Opus 5 提示 | 自身就会验证，旧提示词里的「请自查」可能导致过度验证，官方建议在 Opus 5 上删掉 |

### 来源说明

- 演示代码用脚本化假模型（运行环境没有 API Key），只验证中间件的跳转、反馈注入和计数行为，不代表真实模型的修正效果。
- 今日 AI 动态：OpenAI 官网页面本次无法打开，无法核实，没有收录；Meta 收购 Conductor 只有媒体报道和 Meta 对媒体的确认，没有找到官方发布，没有收录。
- DeepSeekMath / GRPO 来自 arXiv 原始论文（非 LangChain/Claude 来源）。

### 延伸学习

- Claude Academy《Claude Code in action》的 Trust it: Verifying unsupervised runs 一课：验证力度和放权程度成正比，先看 diff 再看总结，用 hook 强制跑测试，请新会话复查。
- Claude Academy《The AI-native SDLC playbook》的 Give Claude a feedback loop 一课：把检查封装成一条命令，修 bug 时先提交会失败的测试，不允许改测试。

### 参考资料

- [Custom middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [Prebuilt middleware - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Structured output - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/structured-output)
- [Hooks reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Best practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Prompting best practices - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Trust it: Verifying unsupervised runs - Claude Code in action - Claude Academy](https://academy.claude.com/courses/claude-code-in-action/trust-it-verifying-unsupervised-runs)
- [Give Claude a feedback loop - The AI-native SDLC playbook - Claude Academy](https://academy.claude.com/courses/ai-native-sdlc-playbook/give-claude-a-feedback-loop)
