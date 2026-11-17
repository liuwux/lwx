# Day 20 · 结构化输出与校验（第二轮：给模型留退路、半截 JSON 不能执行、修复要有上限）（2026-11-17）

## 今日 AI 动态（2026-11-17）

### 趋势观察

本期的新内容来自阿里和 Google。Qwen4-Max 在官方文档里把函数调用、结构化输出、MCP、并行工具调用都列为标配，「Agent 需要的接口能力」已经是旗舰模型的入门要求，不再是卖点。Google 则把「技能（skill）」做成了可以分享、可以挂 Drive 文件的单位，和 Anthropic 的 Agent Skills 思路相近。推测：「可分享的技能包」会成为各家在 Agent 生态里竞争的新单位。

### 今日要点

- 阿里 Qwen4-Max 于 11 月 16 日正式上线 QwenCloud，100 万 token 上下文，只提供托管 API，不开放权重。
- Gemini 应用里的技能可以分享给同事，还能挂上 Drive、Gemini Notebook、Google Photos 里的文件。
- Apps Script 里的 Gemini 侧边栏正式发布：用自然语言写脚本，执行报错可以一键让 Gemini 解释并修复。

### 分公司动态

#### 1. Google

**[Share skills and add files from Google Drive, Gemini Notebook and Google Photos to your skills in the Gemini app](https://workspaceupdates.googleblog.com/)**（2026-11-16）
在 Gemini 网页版的技能页面里，可以把技能分享给别人，授予查看或编辑权限，对方确认后安装，之后 Gemini 会在合适的时候自动调用，也可以在对话里 @ 它。创建技能时可以挂上 Drive、Gemini Notebook 和 Google Photos 的文件，Drive 文件和 Notebook 会实时同步。面向所有 Workspace 客户和个人账号，11 月 12 日起逐步推送，最长 15 天。
- 为什么重要：技能从「个人提示词」变成了团队可以共享、带数据源的可复用单元，企业里的 Agent 能力开始像文档一样流转。

**[Easily customize and extend Google Workspace with Gemini in Apps Script](https://workspaceupdates.googleblog.com/)**（2026-11-16）
Apps Script 里的 Gemini 侧边栏结束公测、正式发布。用自然语言描述需求，Gemini 读取当前项目文件并写出脚本，包括定时触发这类自动化；改动可以逐块接受或拒绝。执行日志里新增「Fix errors」按钮，让 Gemini 解释运行时错误并给出修复。
- 为什么重要：这是「写代码 → 运行 → 看报错 → 修复」闭环进入办公自动化工具的例子，和今天讲的「校验失败把错误回给模型」是同一个思路。

#### 2. 阿里巴巴 / Qwen

**[qwen4-max 上线（QwenCloud 模型发布记录）](https://docs.qwencloud.com/changelog/models)**（2026-11-16）
Qwen 的新旗舰多模态模型，支持文本、图像、视频理解。11 月 11 日先以预览版在 Token Plan 中提供，16 日正式上线。官方文档列出：100 万 token 上下文，支持函数调用、结构化输出、MCP、流式和并行工具调用、思考模式、显式与隐式缓存；不支持微调、私有部署和批量推理。
- 为什么重要：结构化输出和 MCP 已经是中国旗舰模型的标配，做企业 Agent 选型时，「能不能稳定出 JSON」不再是区分点，区分点在价格、延迟和工具调用的稳定性。

### 动态来源

- [Google Workspace Updates](https://workspaceupdates.googleblog.com/)（两篇均为 2026-11-16 发布）
- [Model releases - QwenCloud](https://docs.qwencloud.com/changelog/models)
- [Latest model - QwenCloud](https://docs.qwencloud.com/developer-guides/getting-started/latest-model)

---

> 第二轮说明：第一轮 Day 20 讲过三种拿 JSON 的方式、「schema 管形状，校验管内容」、错误回传重试、拒答和截断。今天往下挖三件事：**格式被锁死后模型会不会硬编**；**半截 JSON 能不能用**；**修复循环怎么收住**。

## 读前三问

1. 开了约束解码，格式一定对，为什么模型反而可能更容易「编」？schema 上怎么防？
2. 流式输出时拿到的半截 JSON 能做什么、不能做什么？解析失败的工具参数怎么回给模型？
3. 校验失败让模型重试，怎么防止无限循环？错误信息怎么写模型才改得对？

## 从一个场景开始

上一轮你的车机空调指令已经很稳：约束解码锁住格式，Pydantic 卡住 16~30 度。这次产品经理加了两个需求。

第一个，车主说「车里有点闷」。schema 里只有 `ACCommand`，模型**必须**填一个区域和温度，于是它给出 `all, 22`。可车主想要的也许是开窗。格式完全正确，意图完全是猜的。

第二个，为了让车主「感觉快」，卡片要边生成边显示。测试同学发现：流到一半时界面上出现了「主驾 2℃」，下一帧才变成 26℃。如果有人图省事，在流的过程中就把值下发给空调 HAL……

第三个是你自己发现的：某个 prompt 改坏了，模型每次都输出 35 度，校验失败、回传、再失败，一个请求转了几十轮。

这三件事对应今天的三个主题：**逃生口**、**流式结构化**、**修复上限**。

## 一句话总结

约束解码只保证格式，不保证内容为真：schema 要给模型留一个「我不确定」的出口；流式时半截 JSON 只用来显示，完整收到并校验后才执行；修复循环要有具体的错误提示和硬上限。

## 先跑起来

两个文件。`car_union.py`：输出结构是「调空调」和「反问车主」二选一，模型第一次两个都调，被自定义错误提示纠正。`car_stream.py`：模拟流式收到的 JSON 片段，看半截时显示了什么，以及收尾时的四种结果。没有 API Key，模型决策用剧本代替，校验、错误回传、partial JSON 解析都是真实库代码。

```python
# car_union.py —— 多种结构二选一 + 「逃生口」+ 自定义错误回传（langchain 1.5.0）
from typing import Literal, Union
from pydantic import BaseModel, Field
from langchain.agents import create_agent
from langchain.agents.structured_output import (ToolStrategy,
    MultipleStructuredOutputsError, StructuredOutputValidationError)
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

class ACCommand(BaseModel):
    """车主明确要调空调时使用"""
    zone: Literal["driver", "passenger", "all"] = Field(description="作用区域")
    temp: int = Field(ge=16, le=30, description="目标温度，摄氏度")

class AskUser(BaseModel):
    """信息不足、无法确定要做什么时使用：反问车主，不要猜"""
    question: str = Field(description="给车主的一句澄清问题")

def on_error(e: Exception) -> str:                       # 回给模型的话：具体到该怎么改
    if isinstance(e, MultipleStructuredOutputsError):
        return "一次只能输出一种结构。不确定车主意图时只用 AskUser。"
    if isinstance(e, StructuredOutputValidationError):
        return f"字段不合法：{str(e)[-80:]}。温度只能 16~30。"
    return f"Error: {e}"

if __name__ == "__main__":                               # 被 car_stream.py 导入时不运行
    class Scripted(GenericFakeChatModel):                # 剧本模型：代替 Claude（无 API Key）
        def bind_tools(self, tools, **kw): return self
    def ai(*calls):
        return AIMessage("", tool_calls=[{"name": n, "args": a, "id": f"c{i}"}
                                         for i, (n, a) in enumerate(calls)])
    script = [ai(("ACCommand", {"zone": "all", "temp": 22}),   # 第 1 次：两个都调（犹豫）
                 ("AskUser", {"question": "是要开空调还是开窗？"})),
              ai(("AskUser", {"question": "是要开空调还是开窗？"}))]  # 第 2 次：改正

    agent = create_agent(Scripted(messages=iter(script)), tools=[],
                         response_format=ToolStrategy(Union[ACCommand, AskUser], handle_errors=on_error))
    res = agent.invoke({"messages": [{"role": "user", "content": "车里有点闷"}]},
                       {"recursion_limit": 6})            # 修复次数的硬上限
    for m in res["messages"]:
        body = m.text or [(c["name"], c["args"]) for c in m.tool_calls]
        print(f"{m.type:>5}: {str(body)[:90]}")
    out = res["structured_response"]
    print("最终：", type(out).__name__, out)
```

```python
# car_stream.py —— 流式 JSON：半截可以显示，不能执行
import json
from langchain_core.utils.json import parse_partial_json
from pydantic import ValidationError
from car_union import ACCommand

# 模拟 API 一段段推来的 partial_json（真实场景来自 input_json_delta）
chunks = ['{"zone": "dri', 'ver", "te', 'mp": 2', '6}']
buf = ""
for c in chunks:
    buf += c
    view = parse_partial_json(buf)                       # 尽量把半截 JSON 补成对象
    print(f"UI 显示：{view}")                             # 只用来刷新界面

def finish(raw: str, stop_reason: str) -> dict:           # 对应 content_block_stop：只在这里执行
    if stop_reason == "max_tokens":
        return {"is_error": True, "content": "输出被截断，请加大 max_tokens 重试"}
    try:
        cmd = ACCommand.model_validate(json.loads(raw))   # 完整 JSON + 业务校验
    except json.JSONDecodeError:                          # 不是合法 JSON：原样包起来回给模型
        return {"is_error": True, "content": json.dumps({"INVALID_JSON": raw}, ensure_ascii=False)}
    except ValidationError as e:                          # JSON 合法但值不对：告诉模型哪里错
        return {"is_error": True, "content": f"{e.error_count()} 处错误：{e.errors()[0]['msg']}"}
    return {"is_error": False, "content": f"已下发 {cmd.zone} {cmd.temp}℃"}

print(finish(buf, "tool_use"))
print(finish('{"zone": "driver", "temp": 26', "tool_use"))
print(finish('{"zone": "driver", "temp": 2', "max_tokens"))
print(finish('{"zone": "driver", "temp": 2}', "tool_use"))
```

**运行**：`pip install langchain pydantic && python car_union.py && python car_stream.py`

**实测输出**（2026-11-17，Python 3.13，langchain 1.5.0 / langchain-core 1.8.1，各运行 2 次结果一致）：

```
human: 车里有点闷
   ai: [('ACCommand', {'zone': 'all', 'temp': 22}), ('AskUser', {'question': '是要开空调还是开窗？'})]
 tool: 一次只能输出一种结构。不确定车主意图时只用 AskUser。
 tool: 一次只能输出一种结构。不确定车主意图时只用 AskUser。
   ai: [('AskUser', {'question': '是要开空调还是开窗？'})]
 tool: Returning structured response: question='是要开空调还是开窗？'
最终： AskUser question='是要开空调还是开窗？'
```

```
UI 显示：{'zone': 'dri'}
UI 显示：{'zone': 'driver'}
UI 显示：{'zone': 'driver', 'temp': 2}
UI 显示：{'zone': 'driver', 'temp': 26}
{'is_error': False, 'content': '已下发 driver 26℃'}
{'is_error': True, 'content': '{"INVALID_JSON": "{\\"zone\\": \\"driver\\", \\"temp\\": 26"}'}
{'is_error': True, 'content': '输出被截断，请加大 max_tokens 重试'}
{'is_error': True, 'content': '1 处错误：Input should be greater than or equal to 16'}
```

**该看什么**：
- 第一段：模型同时调了两个结构，LangChain 给**每个** tool_call 都回了一条 `tool` 消息（所以有两行一样的），回的是你写的那句具体提示，而不是默认的英文报错。最终拿到的是 `AskUser` 对象，下游据此去语音反问。
- 第二段第 1 行：`'dri'` 不是合法枚举值；第 3 行：`temp: 2` **类型正确、值错误**。这就是「半截能看不能用」。
- 收尾四种结果分别是：成功、JSON 不完整、被截断、JSON 完整但业务不合法。后三种都以 `is_error: True` 回给模型，而不是抛异常。

## 核心概念

### 1. 逃生口：别让 schema 逼模型编

- **是什么**：在输出结构里给模型留一条「信息不够」的合法路径，比如单独的 `AskUser` 结构，或者允许某个字段为 `null`。
- **没有它会怎样**：约束解码会把每个 token 限制在 schema 允许的范围内，schema 里没有「不知道」，模型就只能挑一个看起来合理的值填上。格式 100% 正确，内容是猜的。
- **对应你熟悉的**：Kotlin 里把 `temp: Int` 改成 `temp: Int?`，逼调用方处理「没有值」的情况，而不是塞一个默认的 0。

OpenAI 在推出 Structured Outputs 时就写明：结构合法，值仍可能是错的。Anthropic 减少幻觉的第一条建议是「明确允许 Claude 说不知道」。放到 schema 里，就是给「不知道」一个位置。

### 2. 多种结构二选一（Union）

- **是什么**：`ToolStrategy(Union[A, B])` 把每个结构注册成一个工具，模型选一个「调用」。LangChain 文档原话：模型会根据上下文选最合适的结构。
- **没有它会怎样**：要么把所有字段塞进一个大结构、一堆可选字段（Claude 每个可选参数都会让语法状态空间大约翻倍），要么模型只能硬选唯一那个结构。
- **对应你熟悉的**：Kotlin 的 `sealed class Intent { AC; Ask }`，下游用 `when` 分支处理。

两个细节：模型可能**同时调两个**，抛 `MultipleStructuredOutputsError`，就是今天代码第一轮的情况；`ProviderStrategy`（原生约束解码）的文档没有写 Union 支持，要多选一时用 `ToolStrategy` 更稳。

### 3. 流式结构化：半截能看，不能用

- **是什么**：Claude 的 fine-grained tool streaming（工具定义加 `eager_input_streaming: true`）会把工具参数**边生成边推**给你，服务端不缓冲、**不做 JSON 校验**。
- **没有它会怎样**：参数很长时，界面要等模型全部生成完才有反应。
- **对应你熟悉的**：像 `Flow` 里的中间状态：可以拿来刷 UI，但提交表单只能用最终值。

官方规则三条：只在 `content_block_stop` 时解析；`stop_reason` 是 `max_tokens` 时参数可能被截在半中间，要重试或修复；解析失败时，把原始字符串包在 `{"INVALID_JSON": ...}` 里、以 `is_error: true` 的工具结果回给模型。今天的 `temp: 2` 说明：半截的值**可能是合法类型的错误值**，校验器也拦不住，因为它根本不知道后面还有一个 6。

### 4. 修复循环要收得住

- **是什么**：错误回传让模型自己改，但要有两样东西：**具体的错误提示**和**硬上限**。
- **没有它会怎样**：prompt 有问题时，模型每次犯同一个错，循环一直转，成本和延迟失控。
- **对应你熟悉的**：OkHttp 的重试拦截器一定配最大次数，不会无限重试。

在 LangChain 里，`handle_errors` 传一个函数，按异常类型返回中文、具体到「该改什么」的提示；硬上限用 `recursion_limit`，超了抛 `GraphRecursionError`（实测：模型一直输出 35 度，`recursion_limit=6` 时直接抛出）。抛出后由你决定降级：反问车主、用安全默认值或放弃。

## 高频面试题

**1. 开了约束解码，模型还会编吗？怎么防？**
- 会。约束解码只保证格式，schema 没给「不确定」的位置，模型只能硬填。
- 加逃生口：`AskUser` 结构、可空字段，或者 `confidence` 加阈值。
- 关键字段要能在输入里找到依据，必要时加一个短的依据字段，代码里核对。
- 高风险动作（下发 HAL）前再做一次业务校验。

**2.（追问）为什么不直接加一个 `reasoning` 字段让模型先推理再填结论？**
- 字段顺序就是生成顺序，先依据后结论确实有用。
- 但 Claude 文档提示：要求逐步推理的字段可能触发 `reasoning_extraction` 拒答，改成「简短说明」。
- 需要深度推理就开 extended thinking，约束解码不作用于 thinking。
- Claude 会把必填字段排在可选字段前，在意顺序就全设为必填。

**3.（追问）校验失败重试，怎么防死循环？**
- 错误提示要具体：哪个字段、为什么错、允许范围，按异常类型分别写。
- 硬上限：LangChain 用 `recursion_limit`，超了抛 `GraphRecursionError` 再降级。
- 记录每次失败的原始输出，同一错误反复出现说明是 prompt 或 schema 的问题，不是运气。
- 只重试可修复的错误：`handle_errors` 可以只传特定异常类型，其余直接抛。

**4.（场景题）车机要边说边显示 Agent 生成的空调设置卡片，你怎么设计？**
- 工具开 `eager_input_streaming`，前端用 partial JSON 解析刷新卡片，标记为「生成中」。
- 执行只发生在收到完整参数后：先 `json.loads`，再 Pydantic 校验，最后才调 HAL。
- `max_tokens` 截断、JSON 不完整、业务不合法分别回 `is_error` 结果，让模型修正。
- 枚举值可能是半截（`dri`），显示层要容忍未知值，不能崩。

## 常见误区

1. **「格式 100% 正确，就说明结果可靠」**：约束解码解决的是「解析会不会崩」，不是「内容对不对」。场景里的 `all, 22` 就是格式完美的猜测。
2. **「流式拿到的字段类型对了，就可以先执行」**：`temp: 2` 类型是整数，但真实值是 26。任何有副作用的动作都只能用完整、校验过的参数。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：流式收集 + 收尾校验**（取自 `car_stream.py`）

```python
# 模拟 API 一段段推来的 JSON 片段
chunks = ['{"zone": "dri', 'ver", "te', 'mp": 2', '6}']
# 累积缓冲区，从空字符串开始
buf = ""
# 逐段处理收到的片段
for c in chunks:
    # 新片段拼到缓冲区末尾
    buf += c
    # 把半截 JSON 尽量补成对象
    view = parse_partial_json(buf)
    # 半截结果只用来刷新界面，不执行
    print(f"UI 显示：{view}")

# 收尾函数：拿到完整字符串和停止原因后才决定是否执行
def finish(raw: str, stop_reason: str) -> dict:
    # 被 max_tokens 截断：不解析，直接让模型加大上限重试
    if stop_reason == "max_tokens":
        return {"is_error": True, "content": "输出被截断，请加大 max_tokens 重试"}
    try:
        # 先解析成字典，再按业务模型校验
        cmd = ACCommand.model_validate(json.loads(raw))
    # JSON 本身不合法：把原文包在一个键里回给模型
    except json.JSONDecodeError:
        return {"is_error": True, "content": json.dumps({"INVALID_JSON": raw}, ensure_ascii=False)}
    # JSON 合法但值不对：告诉模型错在哪
    except ValidationError as e:
        return {"is_error": True, "content": f"{e.error_count()} 处错误：{e.errors()[0]['msg']}"}
    # 全部通过才执行
    return {"is_error": False, "content": f"已下发 {cmd.zone} {cmd.temp}℃"}
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 判断 `max_tokens` 要放在 `try` **之前**：截断的内容一旦被别人用 partial 解析器「修补」过，就会变成像 `temp: 2` 那样看似完整的对象，先看停止原因才安全。
- 两个 `except` 的顺序和内容别混：`json.JSONDecodeError` 回原文，`ValidationError` 回错误说明；`e.errors()` 是方法，要加括号。
- `json.dumps(..., ensure_ascii=False)`：不加的话中文会变成 `\uXXXX`；用 `json.dumps` 而不是字符串拼接，引号才会被正确转义。

**昨日复习（只做第③步）**：第二轮 Day 19 的「沙箱里的 `execute` 工具」。只看你自己写的注释重写，重点核对：`env` 是新字典、`subprocess.TimeoutExpired`、截断时先写文件再取 `out[-MAX_OUT:]`。

## 动手练习（30 分钟）

在今天两个文件的基础上改：

1. 在 `car_union.py` 的剧本里，把第 2 次也改成 `ACCommand` 且 `temp: 35`，重复 10 次，确认 `recursion_limit=6` 时抛 `GraphRecursionError`；再用 `try/except` 接住，降级为返回 `AskUser("最高只能 30 度，要调到 30 吗？")`。
2. 给 `ACCommand` 加一个可空字段 `source: str | None`（车主原话里的依据），用 `anthropic.transform_schema(ACCommand)` 看它在 schema 里变成了什么，想一想它会不会被排到 `zone` 前面。
3. 在 `car_stream.py` 里加一个 `safe_view(view)`：半截时 `zone` 不在合法枚举里就显示「…」，`temp` 不在 16~30 就显示「--」。

## 读后自测

1. 开了约束解码，格式一定对，为什么模型反而可能更容易「编」？schema 上怎么防？→ 见第 7 节「核心概念 1」
2. 流式输出时拿到的半截 JSON 能做什么、不能做什么？解析失败的工具参数怎么回给模型？→ 见第 7 节「核心概念 3」
3. 校验失败让模型重试，怎么防止无限循环？错误信息怎么写模型才改得对？→ 见第 7 节「核心概念 4」

## AI 历史知识点：Structured Outputs 结构化输出（2024，OpenAI）

**是什么**：让模型输出严格符合开发者给定 JSON Schema 的 API 能力，有两种用法：函数定义上的 `strict: true`，和 `response_format` 的 `json_schema` 选项。同时加了一个 `refusal` 字段，让开发者能识别模型拒答。

**何时由谁提出**：OpenAI 于 2024 年 8 月 6 日发布。此前一年的 DevDay 推出过 JSON mode，但它只保证「是合法 JSON」，不保证符合某个 schema。

**为什么重要**：它把约束解码做成了通用 API。做法是把 schema 转成上下文无关文法，每一步采样时把会破坏 schema 的 token 屏蔽掉。官方评测：复杂 schema 跟随任务上，`gpt-4o-2024-08-06` 开启后得分 100%，`gpt-4-0613` 不到 40%；模型本身不开约束是 93%，剩下的差距靠约束解码补齐。首次使用新 schema 需要额外编译时间，之后缓存。

**和今天的联系**：这篇发布文自己就列出了今天讲的局限：拒答和 `max_tokens` 截断会破坏 schema、**结构合法时值仍可能是错的**。后来 Claude 的 structured outputs 也采用了同样的思路（语法编译、缓存、拒答例外）。约束解码解决了「能不能解析」，「对不对」仍然要靠逃生口和校验。

**原始来源**：[Introducing Structured Outputs in the API - OpenAI](https://openai.com/index/introducing-structured-outputs-in-the-api/)

## 参考区（用时查，不必通读）

### 细节速查

**LangChain `ToolStrategy`**
- `ToolStrategy(schema, *, tool_message_content=None, handle_errors=True)`；`schema` 可为 Pydantic 类、`Union[...]`、JSON Schema dict。
- `handle_errors`：`True`（默认模板）/ `str`（固定提示）/ 异常类型或元组（只重试这些）/ `Callable[[Exception], str]` / `False`（全部抛出）。
- 默认回传：多结构 `Error: Model incorrectly returned multiple structured responses (A, B) when only one is expected. Please fix your mistakes.`；校验失败 `Error: Failed to parse structured output for tool 'X': ... Please fix your mistakes.`；成功 `Returning structured response: {...}`。
- 拒答：`ProviderStrategy` 抛 `StructuredOutputValidationError`；`ToolStrategy` 结束时没有 `structured_response`；加 `ModelRefusalMiddleware`（`langchain>=1.4.9`）统一抛 `ModelRefusalError`。
- 硬上限：`agent.invoke(..., {"recursion_limit": N})`，超出抛 `GraphRecursionError`。
- 流式辅助：`langchain_core.utils.json.parse_partial_json(s)`，尽量把半截 JSON 补全成对象。

**Claude**
- 复杂度上限：20 个 strict 工具、24 个可选参数、16 个使用 `anyOf` 或类型数组（如 `["string","null"]`）的参数；超出返回 400「Schema is too complex for compilation」；编译超时 180 秒。
- 降复杂度顺序：只把关键工具设为 strict → 减少可选参数（有合理默认值就设为必填，让模型显式填默认值）→ 简化嵌套 → 拆成多次请求或子 Agent。
- 字段顺序：按 schema 定义，但必填在前、可选在后。
- 键不固定的字典：空 `properties` 的 object 只能返回 `{}`，改用 `[{key, value}]` 数组。
- 约束不作用于 thinking、工具调用和工具结果；与 prefill 不兼容；与 citations 同用返回 400。
- fine-grained tool streaming：每个工具加 `eager_input_streaming: true`（替代旧头 `fine-grained-tool-streaming-2025-05-14`，旧头不能与 computer use / browser use 同用）；`content_block_start` 里的 `input: {}` 只是占位，真实值由 `input_json_delta.partial_json` 字符串拼出。

### 来源说明

- 示例代码的模型决策是剧本，未调用真实 Claude（本环境无 Anthropic API Key）；校验、错误回传、partial JSON 解析为真实运行。
- Claude Academy「Structured data」一课正文需登录，仅参考课程大纲。

### 延伸学习

- Claude Academy · Building with the Claude API → Structured data；Tool use with Claude → Tool schemas
- LangChain 文档 · Structured output（Union schema 与自定义错误处理示例）

### 参考资料

- [Structured outputs - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
- [Fine-grained tool streaming - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming)
- [Reduce hallucinations - Claude Platform Docs](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)
- [Structured output - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/structured-output)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
