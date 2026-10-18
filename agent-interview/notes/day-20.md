# Day 20 · 结构化输出与校验（2026-10-18）

## 读前三问

1. 让模型输出 JSON 有哪几种做法？可靠性分别差在哪里？
2. 用了 Claude 原生结构化输出（约束解码），为什么还要自己做 Pydantic 校验？
3. 校验失败时，Agent 怎样把错误「喂回」模型让它自己改？什么情况下改也没用？

## 从一个场景开始

车机语音助手的链路是：车主说话 → 模型理解 → 输出一条指令 → 你的 Kotlin 代码去调空调 HAL。

一开始你在提示词里写「请只输出 JSON」，测试时一切正常。上线一周，崩溃日志来了：

- 有时模型在 JSON 前面加一句「好的，以下是指令：」，`JSONObject` 直接抛异常；
- 有时字段名写成 `temperature`，而你的代码读的是 `temp`；
- 车主说「调到 35 度」，模型老老实实输出 35，可空调只支持 16~30 度。

前两个是**格式**问题：输出根本不是你要的结构。第三个是**业务**问题：结构对了，值不合法。今天就是把这两层都堵上：先让模型「只能」按结构输出，再在代码里校验，不合格就把错误告诉模型让它重来。

## 一句话总结

结构化输出分两层：用工具调用或约束解码保证格式一定符合 schema，用 Pydantic 等校验保证值符合业务规则；校验失败就把错误回给模型重试，拒答和截断要单独处理。

## 先跑起来

一个「空调指令」schema，温度限定 16~30。代码先看 Claude SDK 实际发给 API 的 schema 长什么样，再用 LangChain 的 `ToolStrategy` 跑一次「输出越界 → 报错回给模型 → 模型改正」。没有 API Key，模型用剧本代替，校验与重试是 LangChain 的真实代码。

```python
# car_ac_struct.py —— 结构化输出 + 校验失败自动修复（langchain 1.4.6 / anthropic 1.15.0）
from typing import Literal
from pydantic import BaseModel, Field
from anthropic import transform_schema
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage as AI

class ACCommand(BaseModel):
    """车机空调指令"""
    zone: Literal["driver", "passenger", "all"] = Field(description="作用区域")
    temp: int = Field(ge=16, le=30, description="目标温度，摄氏度")

# 1) Claude 原生结构化输出能约束到什么程度？看 SDK 实际发给 API 的 schema
print("发给 API 的 temp：", transform_schema(ACCommand)["properties"]["temp"])

# 2) 没有 API Key：用剧本模型代替 Claude；第一次故意给出越界温度
class Scripted(GenericFakeChatModel): bind_tools = lambda self, tools, **kw: self
call = lambda i, t: AI("", tool_calls=[{"name": "ACCommand", "id": f"c{i}",
                                        "args": {"zone": "driver", "temp": t}}])
model = Scripted(messages=iter([call(1, 35), call(2, 30)]))

agent = create_agent(model, tools=[],
                     response_format=ToolStrategy(ACCommand))  # handle_errors 默认 True
res = agent.invoke({"messages": [{"role": "user", "content": "主驾太冷了，调到35度"}]})

for m in res["messages"]:
    body = m.text or m.tool_calls[0]["args"]
    print(f"{m.type:>5}: {str(body)[:120]}")
cmd = res["structured_response"]
print("最终对象：", type(cmd).__name__, cmd)
```

**运行**：`pip install langchain anthropic && python car_ac_struct.py`

**实测输出**（2026-10-18，Python 3.13）：

```
发给 API 的 temp： {'type': 'integer', 'description': '目标温度，摄氏度\n\n{maximum: 30, minimum: 16}', 'title': 'Temp'}
human: 主驾太冷了，调到35度
   ai: {'zone': 'driver', 'temp': 35}
 tool: Error: Failed to parse structured output for tool 'ACCommand': Failed to parse data to ACCommand: 1 validation error for
   ai: {'zone': 'driver', 'temp': 30}
 tool: Returning structured response: zone='driver' temp=30
最终对象： ACCommand zone='driver' temp=30
```

**该看什么**：
- 第一行：`ge=16, le=30` 没有变成 schema 约束，而是被**挪进了 description**。Claude 的约束解码不支持数值范围，范围只能靠你的代码校验。
- 第一次输出 35 被 Pydantic 拒绝，错误以一条 `tool` 消息回给模型，模型第二次给出 30。这就是「失败修复」循环，你一行重试代码都没写。
- 最终拿到的是 `ACCommand` 对象（在 `structured_response` 里），不是字符串，下游直接用。

> 说明：模型决策是剧本（本环境无 Anthropic API Key），校验、错误回传和 schema 转换均为真实库代码，已实测。

## 核心概念

### 1. 三种拿 JSON 的方式

| 方式 | 怎么做 | 格式可靠性 |
|---|---|---|
| 提示词要求 | 「只输出 JSON」+ 示例 | 大多数时候对，偶尔多一句话、缺字段 |
| 工具调用 | 把 schema 当成一个工具，让模型「调用」它 | 结构基本对，但不保证 100% |
| 原生约束解码 | API 按 schema 限制每个 token 只能怎么生成 | 格式保证符合 schema |

- **是什么**：「约束解码」的大白话：模型每生成一个字，API 都检查「按 schema 这里还能写什么」，不合规的字直接不让选。Claude 里叫 structured outputs，用 `output_config.format` 开启；给工具加 `strict: true` 则约束工具参数。
- **没有它会怎样**：就是场景里的前两种崩溃，你得写一堆正则去「抠」JSON。
- **对应你熟悉的**：提示词要求像口头约定接口格式；约束解码像 Retrofit + 编译期生成的数据类，格式错了根本过不了。

### 2. Schema 约束 ≠ 业务校验

- **是什么**：约束解码只管「结构」：类型、必填字段、枚举。数值范围、字符串长度这类约束 Claude 不支持，SDK 会把它们挪进描述里当提示，然后在拿到结果后**由 SDK 在本地校验**。
- **没有它会怎样**：以为开了 strict 就万事大吉，35 度照样下发到空调。
- **对应你熟悉的**：Gson 能把 JSON 解析成 `data class`，但「温度是否在 16~30」还得在 ViewModel 里 `require()`。

一句话记：**schema 管形状，校验管内容。** 跨字段规则（比如「副驾无人时不能设副驾温度」）更是只能写在代码里。

### 3. 失败修复循环

- **是什么**：校验失败时，不直接抛给用户，而是把错误信息作为一条消息发回模型：「temp 必须 ≤30，请修正」，让它重新输出。LangChain 的 `ToolStrategy` 默认就这么做（`handle_errors=True`）。
- **没有它会怎样**：一次格式小错就让整个任务失败，或者你在业务代码里写满 try/catch 和手动重试。
- **对应你熟悉的**：像表单校验：不是直接崩，而是把「手机号格式不对」显示给用户让他改。

两个要点：错误信息要**具体**（哪个字段、为什么错、允许的范围），模型才改得对；必须设**重试上限**，改两三次还不对就降级（问车主、用默认值或放弃）。

### 4. 保证也有例外

- **是什么**：即使开了约束解码，以下情况输出仍可能不符合 schema：模型**拒答**（`stop_reason` 为 `refusal`）、**被截断**（`max_tokens` 不够）。另外枚举值的大小写偶尔会不一致。
- **没有它会怎样**：直接 `json.loads` 解析被截断的半截 JSON，线上崩溃。
- **对应你熟悉的**：网络请求成功返回 200，也要检查 body 是不是完整的。

做法：先看 `stop_reason` 再解析；截断就加大 `max_tokens` 重试；枚举比较用忽略大小写，避免设计只差大小写的枚举值。

## 高频面试题

**1. 怎么保证 LLM 稳定输出合法 JSON？**
- 首选原生结构化输出（约束解码），格式由 API 保证。
- 不支持时用工具调用，把 schema 定义成工具参数。
- 拿到后统一用 Pydantic 校验，失败把错误回给模型重试，设上限。
- 解析前检查 `stop_reason`，处理拒答和截断。

**2.（追问）开了 strict / 约束解码，还会出错吗？为什么还要校验？**
- 会：拒答、`max_tokens` 截断时不保证符合 schema。
- schema 只管结构，数值范围、长度、正则以外的规则不支持。
- 跨字段、依赖外部状态的业务规则只能在代码里查。
- 所以约束解码降低格式错误，校验兜住业务错误，两者互补。

**3.（追问）JSON outputs 和 strict tool use 有什么区别？**
- JSON outputs 约束模型的**最终回答**长什么样。
- strict tool use 约束模型**调用工具时的参数**和工具名。
- 两者可以同一个请求里一起用。
- Agent 里多步调工具用 strict，最后要一份报告/对象用 JSON outputs。

**4.（场景题）车主说「空调调到 35 度」，你的 Agent 怎么处理？**
- schema 定义温度为整数、区域为枚举，用约束解码保证结构。
- Pydantic 校验 16~30，失败把「最高 30 度」回给模型。
- 模型可以改成 30 并在回复里告诉车主「已调到最高 30 度」，或反问确认。
- 下发 HAL 前执行层再做一次范围检查，不信任任何上游输出。

## 常见误区

1. **「提示词写清楚就够了，不需要 schema」**：测试时 99% 正常，线上千万次调用里那 1% 就是崩溃。格式靠机制保证，不靠提示词。
2. **「把所有约束都塞进 schema」**：Claude 的约束解码有不支持的特性（数值范围、递归等）和复杂度上限，塞多了会返回 400 或拖慢首次请求。schema 只放结构，规则放代码。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：定义 schema → 挂到 Agent → 取出结构化结果**（取自 `car_ac_struct.py`，去掉剧本模型部分）

```python
# 定义输出结构：继承 Pydantic 的 BaseModel
class ACCommand(BaseModel):
    # 类的文档字符串会成为给模型看的说明
    """车机空调指令"""
    # 区域只能是三个值之一，用 Literal 做枚举
    zone: Literal["driver", "passenger", "all"] = Field(description="作用区域")
    # 温度是整数，业务范围 16~30，交给 Pydantic 校验
    temp: int = Field(ge=16, le=30, description="目标温度，摄氏度")

# 看看 SDK 实际发给 API 的 schema：范围约束被挪进了描述
print(transform_schema(ACCommand)["properties"]["temp"])

# 创建 Agent：不给业务工具，只要求按 ACCommand 输出
agent = create_agent(model, tools=[],
                     # 用工具调用策略，校验失败默认把错误回给模型重试
                     response_format=ToolStrategy(ACCommand))
# 发起一次对话，输入是消息列表
res = agent.invoke({"messages": [{"role": "user", "content": "主驾太冷了，调到35度"}]})

# 逐条打印对话过程，看清「出错 → 回传 → 改正」
for m in res["messages"]:
    # 有文本打文本，没文本说明是工具调用，打参数
    body = m.text or m.tool_calls[0]["args"]
    print(f"{m.type:>5}: {str(body)[:120]}")
# 最终结果在 structured_response 里，是校验过的对象
cmd = res["structured_response"]
print(type(cmd).__name__, cmd)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 范围参数是 `ge` / `le`（大于等于 / 小于等于），不是 `min` / `max`。
- 结果的 key 是 `structured_response`，不是 `structured_output` 或 `output`。
- `response_format` 是 `create_agent` 的参数，`ToolStrategy` 要从 `langchain.agents.structured_output` 导入。

**昨日复习（只做第③步）**：Day 19 的「一轮多个动作的执行器 + 坐标换算 + 确认闸门」。只看你自己写的注释重写 `run` 和 `execute_turn`，重点核对：失败后的动作也要 `append`、id 字段叫 `tool_use_id`、坐标是乘 2。

## 动手练习（30 分钟）

在 `car_ac_struct.py` 上改：

1. 给 `ACCommand` 加一个跨字段规则：`zone` 为 `"passenger"` 时温度不得低于 18（用 Pydantic 的 `model_validator`），剧本里构造一次违规再改正。
2. 把 `ToolStrategy(ACCommand)` 改成 `ToolStrategy(ACCommand, handle_errors="温度必须在16到30之间，请修正")`，观察回给模型的错误消息变化。
3. 再改成 `handle_errors=False`，看异常类型是什么。用一句话写下：什么场景该关掉自动重试？

## 读后自测

1. 让模型输出 JSON 有哪几种做法？可靠性分别差在哪里？→ 见核心概念 1
2. 用了原生结构化输出，为什么还要自己做 Pydantic 校验？→ 见核心概念 2、4
3. 校验失败时怎样把错误喂回模型？什么情况下改也没用？→ 见核心概念 3、4

---

## 参考区（用时查，不必通读）

### 细节速查

**Claude 结构化输出**
- JSON outputs：`output_config={"format": {"type": "json_schema", "schema": {...}}}`，结果在 text 块里，需 `json.loads`；无需 beta 头。旧参数 `output_format` + 头 `structured-outputs-2025-11-13` 已弃用，SDK v1.0+ 的 `create()` 传它会 `TypeError`。
- Python SDK：`client.messages.parse(output_format=PydanticModel)` → `response.parsed_output`（这里的 `output_format` 是 SDK 辅助参数，实际发送 `output_config.format`）；`anthropic.transform_schema()` 只转换不发送。
- SDK 转换：去掉不支持的约束并写进 description、所有 object 加 `additionalProperties: false`、过滤 string format；解析后仍按原约束本地校验。
- Strict tool use：工具定义加 `strict: true`，保证工具名和参数符合 `input_schema`。
- 支持：object/array/string/integer/number/boolean/null、enum（仅标量）、const、anyOf、allOf（不含 `$ref`）、内部 `$ref`/`$defs`、default、required；format：date-time、time、date、duration、email、hostname、uri、ipv4、ipv6、uuid；`minItems` 仅 0 或 1；简单正则。
- 不支持（返回 400）：递归、外部 `$ref`、`minimum`/`maximum`/`multipleOf`、`minLength`/`maxLength`、`additionalProperties` 非 false、正则的回溯引用/前后瞻/`\b`。
- 复杂度上限（一个请求所有 strict schema 合计）：20 个 strict 工具、24 个可选参数、16 个用 anyOf 或类型数组的参数、编译超时 180 秒。
- 属性顺序：必填在前、可选在后；在意顺序就全设为必填。
- 例外：`stop_reason: "refusal"`（200 且计费，可能不符 schema；要求逐步推理可能触发 `reasoning_extraction` 拒答）、`"max_tokens"`（可能不完整）；枚举首字母大小写可能不一致。
- 性能：首次使用某 schema 要编译语法，缓存 24 小时；改 schema 结构或工具集会失效，只改 name/description 不失效；改 `output_config.format` 会使该对话的 prompt cache 失效；会额外注入一段系统提示。
- 不兼容：citations（400）、消息预填（prefill）。兼容：Batch、token 计数、流式、与 strict tool use 同用。约束只作用于模型直接输出，不约束 thinking、tool result。schema 里不要放 PHI。
- 支持模型：Opus 4.5 及以后、Sonnet 4.5 及以后、Haiku 4.5/5.5、Fable 5/5.1 等；Bedrock 仅旧版集成部分模型。

**LangChain**
- `create_agent(..., response_format=X)`，结果在 `result["structured_response"]`。
- `X` 可为：`ToolStrategy(schema)`、`ProviderStrategy(schema, strict=None)`、直接传 schema 类型（按模型 profile 自动选，支持原生就用 Provider，否则 Tool；`langchain>=1.1`）、`None`。JSON Schema dict 必须包在策略里，且需顶层 `title`、`description`。
- schema 类型：Pydantic（返回实例）、dataclass / TypedDict / JSON Schema（返回 dict）；`ToolStrategy` 还支持 `Union[...]` 让模型多选一。
- `ToolStrategy` 参数：`tool_message_content`、`handle_errors`：`True`（默认，全部重试）/ `str`（固定提示）/ 异常类型或元组（只重试这些）/ `Callable[[Exception], str]` / `False`（全部抛出）。
- 错误类型：`StructuredOutputValidationError`（不符 schema）、`MultipleStructuredOutputsError`（一次调了多个结构化工具）。
- `ChatAnthropic.with_structured_output(schema)` 默认走 function calling；`method="json_schema"` 走 Claude 原生（`langchain-anthropic>=1.1.0`）；`bind_tools(..., strict=True)` 开 strict tool use。

### 来源说明

- 示例代码的模型决策是剧本，未调用真实 Claude（本环境无 Anthropic API Key）。
- Claude Academy「Structured data」一课的正文需登录，仅参考了课程大纲。

### 延伸学习

- Claude Academy · Building with the Claude API → Accessing Claude with the API → Structured data；Tool use with Claude → Tool schemas
- LangChain 文档 · Structured output（含 Union schema、自定义错误处理示例）

### 参考资料

- [Structured outputs - Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
- [Structured output - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/structured-output)
- [ChatAnthropic integration - Docs by LangChain](https://docs.langchain.com/oss/python/integrations/chat/anthropic)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
