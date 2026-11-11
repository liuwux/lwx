# Day 14 · MCP 协议（第二轮：问用户要两跳、注解不可信、令牌不透传）（2026-11-11）

## 今日 AI 动态（2026-11-11）

### 趋势观察

- 前沿模型的比拼正从「每百万 token 多少钱」转向「完成一个任务花多少钱」：Google 给 Gemini 4 Carbon 打出的卖点是单任务成本不到第二名的一半，前一天 Anthropic 的 Fable 5.2 主打大小模型分工，思路一致。
- 形式化验证、网络安全成了展示模型能力的新舞台：DeepMind 用 Carbon 驱动的智能体在 Lean 里证明了 Signal 协议实现代码的安全性。
- 推测：Claude 推出 Sheets、Whiteboards 并让 Data 正式可用，又在同一天启动 IPO 路演，对话产品正在被包装成企业办公平台，以支撑上市后的收入故事。

### 今日要点

- Anthropic 启动 IPO 路演，拟在纳斯达克上市，代码 ANT，预计每股 70–78 美元。
- Claude 推出 Whiteboards、Sheets（均为 beta），Claude Dashboards 改名 Claude Data 并正式可用。
- Gemini 4 Carbon 全面开放给 API 付费层、企业版和 Ultra 订阅者，Antigravity 同步上线 `/swarm` 多智能体预览。
- DeepMind 发布 AlphaProof Code，对 Signal 协议实现代码完成形式化验证。
- Pixel 10 开始获得 Gemini Intelligence，手机端支持跨 App 完成订餐、叫车等多步任务。

### 分公司动态

#### 1. Anthropic / Claude

**[Anthropic 启动 IPO 路演](https://www.anthropic.com/news/roadshow-launch)**（2026-11-10）
Anthropic 正式启动首次公开募股路演，发行 8.25 亿股 A 类普通股，大部分由公司发行，少部分由现有股东出售，预计每股 70–78 美元。公司已申请在纳斯达克全球精选市场上市，代码 "ANT"。页面没有写定价日和上市日。
**为什么重要**：头部大模型公司即将上市，信息披露和商业化节奏都会随之改变。

**[Claude Whiteboards、Claude Sheets 与 Claude Data](https://claude.com/resources/articles/sheets-whiteboards-data)**（2026-11-10）
Whiteboards（beta）能把零散想法整理成带负责人和日期的计划，并导出到 Miro、Lucid 等工具。Sheets（beta）支持和 Claude 一起建表、改公式，每处修改都有标注，可导出 xlsx/CSV。10 月推出的 Dashboards 改名 Claude Data 并正式可用，新增认证指标、实时共享和可回滚的修改记录。文中还提到，Adobe、Atlassian、Notion 等交互式连接器现在可以以完整应用的形式在对话旁打开。
**为什么重要**：Claude 从聊天助手进一步变成办公协作平台，和 Google、微软的办公套件正面竞争。

#### 2. Google

**[Gemini 4 Carbon 全面开放](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-carbon-now-available/)**（2026-11-10）
Carbon 此前只通过 Fairwind 计划开放给网络安全防御方，现在开放给 Gemini API 付费层、企业版、Google AI Ultra 订阅者和 Antigravity。11 月 30 日前 API 优惠价为每百万 token 输入 2 美元、输出 10 美元，之后翻倍。官方称它在 CursorBench 4.0 等榜单排名第一，Antigravity 还推出了协调多个 Carbon 智能体的 `/swarm` 预览。
**为什么重要**：Google 最强模型正式商用，以「性能第一、单任务更便宜」直接对标 Anthropic 和 OpenAI 的旗舰。

**[AlphaProof Code：形式化验证 Signal 协议实现](https://deepmind.google/blog/introducing-alphaproof-code/)**（2026-11-10）
人类专家决定「证明什么」，Gemini Carbon 驱动的智能体用 Lean 写证明，验证到实际编译运行的代码这一层。全程 4 周，约 1% 的代码交给人工复核；团队把两个已修复的旧漏洞放回去，系统都能证明这些版本不安全。验证材料已开源。
**为什么重要**：「AI 写证明、机器来检查」可能成为保障高安全代码的新做法。

**[Gemini Intelligence 11 月更新](https://blog.google/products-and-platforms/platforms/android/gemini-intelligence-november-features-devices/)**（2026-11-10）
Pixel 10 系列开始获得 Gemini Intelligence，首款搭载它的 Motorola 手机将于 11 月 19 日在美国开启预订。新功能包括实时整理口述内容的 Rambler、用一句话生成桌面小组件，以及跨 App 完成订餐、叫车、订座等多步任务。
**为什么重要**：手机上的 Agent 正从「回答问题」变成「替你在各个 App 里办事」，这正是 Android 开发者转岗 Agent 的主战场。

### 动态来源

- https://www.anthropic.com/news/roadshow-launch
- https://claude.com/resources/articles/sheets-whiteboards-data
- https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-carbon-now-available/
- https://deepmind.google/blog/introducing-alphaproof-code/
- https://blog.google/products-and-platforms/platforms/android/gemini-intelligence-november-features-devices/

---

> 第二轮说明：第一轮 Day 14 讲过 MCP 的三方架构、六种原语、stdio 与 HTTP 传输、`MCPServer` 写法和 `ToolError`。今天只讲进阶：协议变成无状态以后，「中途问用户」怎么做；结构化输出；注解能不能信；令牌怎么管。

## 读前三问

1. MCP 2026-07-28 版去掉了会话，server 在工具执行到一半时还能请用户确认吗？流程是怎样的？
2. 工具标了 `destructiveHint`，客户端能不能只根据它决定要不要弹确认？
3. 车机 MCP server 要调用车企云的 API，能不能直接把客户端传来的令牌转发过去？为什么？

## 从一个场景开始

第一轮你写的车机 MCP server 在本地跑得很好。这次团队要把它部署到云上，放在负载均衡后面跑三个实例，再加一个新需求：车速超过 100 km/h 时开车窗，要先让司机点确认。

你照着旧教程在工具里写了 `await ctx.elicit(...)`，一跑就报错：

```
NoBackChannelError: Cannot send 'elicitation/create': this transport context has no back-channel for server-initiated requests.
```

原因是新版协议不再允许 server 在工具执行中途主动给客户端发请求。server 不能「暂停等回复」，只能先返回「我需要这些输入」，由客户端问完用户后带着答案重新调用一次。

今天就讲这种「两跳」的写法，以及上云之后必须考虑的三件事：返回什么格式、注解能不能信、令牌怎么管。

## 一句话总结

新版 MCP 是无状态的：server 要问用户，就先返回「需要输入」，客户端问完后带着答案和加密状态重新调用；危险操作的确认不能依赖工具注解，令牌必须是发给这个 server 本身的。

## 先跑起来

一个车机 MCP server：`tire_pressure` 返回结构化胎压；`open_window` 执行前请司机确认。客户端在同一进程内连接（默认按 2026-07-28 协议），分别模拟司机点「确认」和「拒绝」。

```python
# car_mcp.py —— 车机 MCP 进阶：结构化输出 + 危险操作先让司机确认（2026-07-28 无状态写法）
import asyncio
from typing import Annotated
from pydantic import BaseModel
from mcp import Client
from mcp.server.mcpserver import MCPServer, Elicit, Resolve, ElicitationResult, AcceptedElicitation
from mcp.types import ElicitResult, ToolAnnotations

mcp = MCPServer("car")
class Tire(BaseModel):               # 返回类型 → SDK 自动生成 outputSchema
    fl: float; fr: float; rl: float; rr: float
class Confirm(BaseModel):            # 让司机填的表单（只能用简单类型）
    ok: bool
def ask_driver(percent: int):        # 解析器：返回 Elicit 表示“这个值要问用户”
    return Elicit(f"车速 110km/h，确认把车窗开到 {percent}%？", Confirm)

@mcp.tool(annotations=ToolAnnotations(readOnlyHint=True))
def tire_pressure() -> Tire:
    """读取四个轮胎的胎压（bar）。"""
    return Tire(fl=2.3, fr=2.3, rl=2.1, rr=2.4)
@mcp.tool(annotations=ToolAnnotations(destructiveHint=True))
def open_window(percent: int, answer: Annotated[ElicitationResult[Confirm], Resolve(ask_driver)]) -> str:
    """打开车窗到指定百分比，执行前需司机确认。"""
    if isinstance(answer, AcceptedElicitation) and answer.data.ok:
        return f"车窗已打开到 {percent}%"
    return f"未执行：司机选择了 {answer.action}"

def driver(choice: str):             # 模拟车机 UI：弹窗并返回司机的选择
    async def on_elicit(ctx, params):
        print(f"  [车机弹窗] {params.message} → {choice}")
        return ElicitResult(action=choice, content={"ok": True} if choice == "accept" else None)
    return on_elicit

async def main():
    async with Client(mcp, elicitation_callback=driver("accept")) as c:
        for t in (await c.list_tools()).tools:
            print(t.name, t.annotations.model_dump(exclude_none=True), "| 入参:", list(t.input_schema["properties"]))
        print("structuredContent:", (await c.call_tool("tire_pressure", {})).structured_content)
        raw = await c.session.call_tool("open_window", {"percent": 50}, allow_input_required=True)
        print("第一跳:", raw.result_type, "| requestState:", raw.request_state[:10] + "…")
        print("open_window:", (await c.call_tool("open_window", {"percent": 50})).content[0].text)
    async with Client(mcp, elicitation_callback=driver("decline")) as c:
        print("open_window:", (await c.call_tool("open_window", {"percent": 50})).content[0].text)

asyncio.run(main())
```

**运行**：`pip install -U "mcp>=2.3" && python car_mcp.py`

**实测输出**（2026-11-11，Python 3.13，mcp 2.3.0）：

```
tire_pressure {'read_only_hint': True} | 入参: []
open_window {'destructive_hint': True} | 入参: ['percent']
structuredContent: {'fl': 2.3, 'fr': 2.3, 'rl': 2.1, 'rr': 2.4}
第一跳: input_required | requestState: v1.LscGc4m…
  [车机弹窗] 车速 110km/h，确认把车窗开到 50%？ → accept
open_window: 车窗已打开到 50%
  [车机弹窗] 车速 110km/h，确认把车窗开到 50%？ → decline
open_window: 未执行：司机选择了 decline
```

**该看什么**：
- `open_window` 的入参只有 `percent`，没有 `answer`。确认结果由框架向用户索取，模型无法自己填一个「已确认」。
- 「第一跳」返回的不是结果，而是 `input_required` 加一段加密的 `requestState`。`c.call_tool` 会自动完成「弹窗 → 带着答案重试」，所以后面直接拿到最终结果。
- 司机拒绝时，工具收到 `decline` 并返回说明。拒绝是正常结果，不是异常。

## 核心概念

### 1. 无状态以后，「问用户」变成两跳

- **是什么**：2026-07-28 版协议规定，server 要向用户要输入（elicitation）、借客户端的模型（sampling）或查询目录（roots），都必须用「多轮往返请求」（MRTR）：第一次调用返回 `InputRequiredResult`，里面列出要问的问题和一段 `requestState`；客户端问完用户，带着答案和原样的 `requestState` 重新调用。
- **没有它会怎样**：旧写法是 server 在执行中途主动发请求，再挂起等待回复。这要求同一个连接一直保持，负载均衡后的多实例就必须做粘性路由或共享会话存储。
- **对应你熟悉的**：Android 的运行时权限。`requestPermissions()` 之后不会在原地阻塞，而是在 `onRequestPermissionsResult` 里拿到结果，再重新执行刚才被拦下的操作。

重试请求可以落到任意一个实例上，因为需要的上下文都在请求里。

### 2. 重试会重跑工具：先问再做

- **是什么**：第二跳是一次全新的调用，「提问之前」的代码每一轮都会再跑一遍。示例用的 `Resolve` 写法里，解析器 `ask_driver` 每轮都执行，工具主体要等答案齐了才执行一次；如果你手写「检查有没有答案，没有就返回 `InputRequiredResult`」，整个工具函数每轮都会重跑。LangChain 的文档也专门提醒：问题必须放在任何副作用之前。
- **没有它会怎样**：如果先调用车窗电机、再问确认，第一跳就已经开了窗；如果先扣费再问，用户会被扣两次。
- **对应你熟悉的**：`Activity` 因配置变化重建时 `onCreate` 会再跑一遍，所以不能在里面直接发起支付请求。

所以解析器里只能「算问题」，不能「做动作」；动作写在确认通过之后。

另外，`requestState` 会经过客户端，规范要求 server 把它当成「攻击者可控的输入」：影响授权或业务逻辑时必须做完整性保护（如 HMAC、AEAD），并绑定用户、请求和过期时间。上面输出里的 `v1.…` 就是 SDK 默认加密后的结果。

### 3. 结构化输出：给程序看的那一份

- **是什么**：工具可以声明 `outputSchema`，返回时在 `structuredContent` 里放符合该结构的 JSON。为了兼容旧客户端，规范建议同时在文本内容里放一份序列化后的 JSON。
- **没有它会怎样**：客户端只能拿到一段文字，想在车机 UI 上画四个轮胎的数字就得自己解析字符串，格式一变就出错。
- **对应你熟悉的**：Retrofit 接口返回一个数据类，而不是一段 HTML。

注意区分：`structuredContent` 是 server 产出的结果数据，和让模型按 schema 生成内容的「结构化输出」不是一回事。

### 4. 工具注解只是提示，不能当作安全依据

- **是什么**：注解（annotations）用来描述工具的行为，例如只读（`readOnlyHint`）、可能造成破坏（`destructiveHint`）。规范明确说这些都是**提示**，除非 server 本身可信，客户端必须把它们当作不可信的信息。
- **没有它会怎样**：如果客户端看到「只读」就自动放行，恶意 server 只要给「删除数据」工具标个只读，就能绕过确认。
- **对应你熟悉的**：APK 自己声明的版本说明。可以用来展示，但安装时的权限检查要由系统来做，不能信 APK 的自述。

更稳的做法是两层确认：Host 侧按自己的白名单决定哪些工具要弹窗；server 侧对真正危险的操作像示例一样，用 elicitation 自己把关。

### 5. 令牌要发给「我」，敏感信息不走表单

- **是什么**：远程 MCP server 是一个 OAuth 资源服务器，只能接受明确签发给它自己的令牌（校验 audience）。它要访问第三方 API 时，应该自己作为 OAuth 客户端去拿第三方的令牌，并和用户身份绑定保存。
- **没有它会怎样**：把客户端的令牌原样转发（token passthrough）会绕过下游的限流和审计，被盗的令牌还能拿这个 server 当跳板。规范对此是「MUST NOT」。
- **对应你熟悉的**：车机 App 不应该把用户的账号 token 直接塞给第三方 SDK，而是走车企后台换一个权限更小的令牌。

相关的一条规定：密码、API Key、支付信息不能用表单方式（form）索取，必须用 URL 方式，让用户在浏览器里直接提交给 server，凭证不经过客户端和模型。

## 高频面试题

**Q1（场景题）：车机 MCP server 上云多实例后，「开窗前确认」开始报错或偶发失效。原因和修法？**
- 旧写法在工具中途主动发请求、挂起等回复，依赖固定连接，无状态和多实例下没有回传通道。
- 改成两跳：先返回 `InputRequiredResult`，客户端问完带着答案和 `requestState` 重试。
- `requestState` 要加密并绑定用户、请求和过期时间，任意实例都能验证。
- 确认问题放在副作用之前，避免重试时动作重复执行。

**Q2：客户端能不能根据 `readOnlyHint` / `destructiveHint` 自动放行或拦截？**
- 注解只是提示，规范要求来自不可信 server 的注解一律当作不可信。
- 放行策略由 Host 自己维护（白名单、按用户设置），注解只能用来改善展示。
- 真正危险的操作，server 侧也要自己发起确认，形成双重把关。

**Q3（追问）：既然第二跳会重跑工具，有哪些写法会出问题？怎么避免？**
- 在提问之前就执行了副作用（开窗、下单、扣费），每一轮都会执行一次。
- 解析器只负责生成问题、保持无副作用；动作写在确认通过的分支里。
- 必须只执行一次的操作，在 server 端用幂等键去重，不能只靠 `requestState`。
- 用户拒绝或关闭弹窗时返回正常结果，让模型能换个方案继续。

**Q4：MCP server 需要调用车企云 API，能不能把客户端带来的令牌直接转发？**
- 不能。server 只接受签发给自己的令牌，必须校验 audience。
- 转发会绕过下游的限流和审计，也会造成「混淆代理」问题。
- 正确做法：server 作为 OAuth 客户端，通过 URL 方式的 elicitation 让用户授权，再把第三方令牌和用户身份绑定保存。
- 权限范围从最小开始，用到高权限操作时再逐步申请。

## 常见误区

- **「`ctx.elicit()` 在哪个版本都能用」**：它依赖 server 主动发请求。按 2026-07-28 协议连接时会直接抛 `NoBackChannelError`，要改成 `Resolve` + `Elicit` 或直接返回 `InputRequiredResult`。
- **「表单里问一下 API Key 很方便」**：规范禁止用表单索取密码、密钥、支付信息，这些必须走 URL 方式，不能经过客户端和模型。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：结构化输出 + 执行前确认**（取自 `car_mcp.py`）

```python
# 定义胎压的返回结构，SDK 会据此生成 outputSchema
class Tire(BaseModel):
    # 四个轮胎各一个浮点数字段
    fl: float; fr: float; rl: float; rr: float
# 定义要用户填写的确认表单
class Confirm(BaseModel):
    # 只有一个布尔字段：是否确认
    ok: bool
# 解析器：根据开窗百分比生成要问用户的问题
def ask_driver(percent: int):
    # 返回 Elicit，告诉框架这个值要向用户索取，并给出表单结构
    return Elicit(f"车速 110km/h，确认把车窗开到 {percent}%？", Confirm)

# 注册为工具，并标注为只读
@mcp.tool(annotations=ToolAnnotations(readOnlyHint=True))
# 声明返回类型为 Tire，结果会放进 structuredContent
def tire_pressure() -> Tire:
    # 工具说明，会作为描述交给模型
    """读取四个轮胎的胎压（bar）。"""
    # 返回四个轮胎的数值
    return Tire(fl=2.3, fr=2.3, rl=2.1, rr=2.4)
# 注册为工具，并标注为可能造成破坏
@mcp.tool(annotations=ToolAnnotations(destructiveHint=True))
# answer 由解析器填充，不出现在模型可见的入参里
def open_window(percent: int, answer: Annotated[ElicitationResult[Confirm], Resolve(ask_driver)]) -> str:
    # 工具说明
    """打开车窗到指定百分比，执行前需司机确认。"""
    # 只有用户点了确认、并且表单里 ok 为真才执行
    if isinstance(answer, AcceptedElicitation) and answer.data.ok:
        # 返回执行结果
        return f"车窗已打开到 {percent}%"
    # 拒绝或取消时返回说明，而不是抛异常
    return f"未执行：司机选择了 {answer.action}"
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 注解参数是驼峰写法 `readOnlyHint=True`；打印出来时 SDK 会显示成 `read_only_hint`，这是 Python 字段名。
- `Resolve(ask_driver)` 传的是函数本身，不要写成 `ask_driver()`。
- 判断是否确认要同时检查两件事：类型是 `AcceptedElicitation`，并且 `answer.data.ok` 为真。

**昨日复习（只做第③步）**：第二轮 Day 13 的「带保护名单的工具结果清理」（`context_clear.py` 里的 `run(exclude)`）。只看你自己写的注释重写，重点核对：`ContextEditingMiddleware(edits=[edit])` 的 `edits` 是列表；工具结果消息的类型是 `"tool"`。

## 动手练习（30 分钟）

在 `car_mcp.py` 的基础上：

1. 在 `ask_driver` 里加 `print("解析器运行")`，在 `open_window` 的 `if` 之前加 `print("电机动作！")`。运行后数一数：解析器每次调用跑了几遍？工具主体跑了几遍？司机拒绝时电机有没有「动作」？再把电机那行挪进 `if`，确认拒绝时不再打印。
2. 把客户端改成 `Client(mcp, mode="legacy", elicitation_callback=...)`，用旧版握手协议连接，看 `Resolve` 写法是否同样能用。
3. 给 `Confirm` 加一个字段 `percent: int = 30`，模拟司机在弹窗里改开窗幅度（回调里返回 `{"ok": True, "percent": 20}`），工具按司机填的值执行。

## 读后自测

1. MCP 2026-07-28 版去掉了会话，server 在工具执行到一半时还能请用户确认吗？流程是怎样的？——见第 1、2 节和 Q1
2. 工具标了 `destructiveHint`，客户端能不能只根据它决定要不要弹确认？——见第 4 节和 Q2
3. 车机 MCP server 要调用车企云的 API，能不能直接把客户端传来的令牌转发过去？为什么？——见第 5 节和 Q4

## AI 历史知识点：Agent2Agent（A2A）协议（2025，Google）

**是什么**：一个让不同厂商、不同框架开发的 Agent 互相协作的开放协议。每个 Agent 用一份 JSON 格式的 Agent Card 声明自己能做什么；协作的基本单位是有生命周期的任务（task），任务产出叫 artifact，消息由不同内容类型的 parts 组成。它基于 HTTP、SSE、JSON-RPC 这些现成标准。

**何时由谁提出**：2025 年 4 月 9 日，Google 在开发者博客发布，首发时有 50 多家技术合作伙伴支持。2025 年 6 月 23 日，Linux 基金会宣布成立 A2A 项目接手治理，由 Google 贡献，AWS、微软、Salesforce、SAP、ServiceNow 等参与，支持者超过 100 家。

**为什么重要**：企业里的 Agent 往往各自困在一个系统里。A2A 把「Agent 之间怎么发现彼此、怎么委托长时间任务、怎么同步进度」标准化了，而且不要求双方共享记忆或工具。

**和今天的趋势有什么联系**：Google 发布时就说明 A2A 是 MCP 的补充：MCP 负责「Agent 连工具和数据」，A2A 负责「Agent 连 Agent」。今天讲的 MCP 无状态化和 elicitation，与 A2A 里需要人参与的长任务，解决的是同一类问题：跨进程、跨厂商时，谁在什么时候向用户要确认。明天 Day 15 会详细讲 A2A。

**原始来源**：
- [Announcing the Agent2Agent Protocol (A2A) - Google Developers Blog](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [Linux Foundation Launches the Agent2Agent Protocol Project](https://linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents)

## 参考区（用时查，不必通读）

### 细节速查

| 主题 | 要点 |
|---|---|
| 2026-07-28 版变化 | RC 于 2026-05-21 发布；去掉 `initialize` 握手和 `Mcp-Session-Id`；版本、客户端信息和能力放在每个请求的 `_meta` 里；新增 `server/discover`；HTTP 要求 `Mcp-Method` / `Mcp-Name` 头；list 结果带 `ttlMs` / `cacheScope`；Tasks 改为官方扩展；Roots、Sampling、Logging 标为弃用（至少保留 12 个月）；工具 schema 改为完整的 JSON Schema 2020-12；资源不存在的错误码由 `-32002` 改为 `-32602` |
| MRTR | 支持的请求：`tools/call`、`resources/read`、`prompts/get`；`resultType: "input_required"`；`inputRequests` 的值只能是 `elicitation/create`、`sampling/createMessage`、`roots/list`；重试的 JSON-RPC `id` 必须不同；客户端必须原样回传 `requestState`，不能解析；server 不能发送客户端没声明能力的请求 |
| `requestState` 安全 | 视为攻击者可控；影响授权或业务时必须 HMAC/AEAD 保护；建议绑定用户、短过期时间、原请求摘要；需要只用一次的，server 端自己保证 |
| elicitation | `mode`：`form`（默认）/ `url`；表单只能是扁平对象，字段为 string / number / integer / boolean / enum；string 的 `format` 支持 email、uri、date、date-time；响应 `action`：`accept`（带 `content`）/ `decline`（明确拒绝）/ `cancel`（关闭弹窗未选择）；客户端必须显示是哪个 server 在请求 |
| URL 方式 | 客户端不能预取 URL、不能未经同意打开、必须展示完整 URL；要用不能被客户端检查内容的方式打开（iOS 用 SFSafariViewController 而非 WKWebView）；server 必须确认打开链接的人就是发起请求的用户（防钓鱼） |
| 工具注解默认值 | `readOnlyHint`=false、`destructiveHint`=true、`idempotentHint`=false、`openWorldHint`=true；后三个只在非只读时有意义；全部是提示 |
| 工具错误 | 协议错误（未知工具、请求格式错）走 JSON-RPC error；执行错误（API 失败、参数越界、业务错误）走 `isError: true`，应交给模型自我纠正 |
| 有状态工具 | 协议层没有会话；需要状态时由工具返回显式句柄（如 `basket_id`），后续调用作为参数传入；句柄不等于授权，每次都要校验归属 |
| 安全最佳实践 | 禁止 token passthrough；代理型 server 要做按客户端的授权同意，防混淆代理；客户端获取 OAuth 元数据 URL 时防 SSRF（屏蔽内网和 169.254.0.0/16）；授权 URL 只允许 http(s)，不用 shell 打开；权限范围逐步申请 |
| Python SDK 2.3.0 | `Resolve` + `Elicit` 写法在新旧协议下都可用（新协议自动转成 `InputRequiredResult`）；实测解析器每轮都执行，工具主体在答案齐后只执行一次；`ctx.elicit()` 在新协议下抛 `NoBackChannelError`；`Client(..., mode="auto")` 为默认值，`"legacy"` 走旧握手；客户端默认最多重试 10 轮；`c.session.call_tool(..., allow_input_required=True)` 可手动拿到第一跳 |
| LangChain | LangSmith Agent Server 把 `interrupt()` 映射成 MCP elicitation（需 `langgraph-api>=0.18.0`、2026-07-28 协议）；`MCPAdapter` 重试时会从头重放工具调用，问题必须在副作用之前 |

### 来源说明

- 示例在同一进程内连接，没有走真实的 HTTP 多实例部署；「多实例下旧写法失效」是根据规范和 SDK 报错推断的，未实测负载均衡场景。
- 历史知识点引用的是 Google 开发者博客和 Linux 基金会新闻稿（非 LangChain/Claude 来源）。
- 规范的 schema 参考页过长，工具注解默认值取自本机 SDK `mcp_types` 中与规范同步的注释。

### 延伸学习

- Claude Academy「Model Context Protocol: Advanced Topics」：sampling、通知、roots，以及 stdio / StreamableHTTP 传输和无状态 HTTP 的取舍，适合配合本篇第 1 节学习。

### 参考资料

- [Model Context Protocol: Advanced topics - Claude Academy](https://academy.claude.com/courses/model-context-protocol-advanced-topics)
- [Tools - MCP Specification (draft)](https://modelcontextprotocol.io/specification/draft/server/tools)
- [Multi Round-Trip Requests - MCP Specification (draft)](https://modelcontextprotocol.io/specification/draft/basic/patterns/mrtr)
- [Elicitation - MCP Specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation)
- [Security Best Practices - MCP](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)
- [The 2026-07-28 MCP Specification Release Candidate - MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
- [Model Context Protocol (MCP) - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/mcp)
- [Human-in-the-loop over MCP - Docs by LangChain](https://docs.langchain.com/langsmith/server-mcp-elicitation)
