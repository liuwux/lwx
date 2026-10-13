# Day 15 · Agent 间通信：A2A 协议（2026-10-13）

## 读前三问

1. A2A 和 MCP 分别解决什么问题？为什么说它们是互补的？
2. 对方 Agent 需要反问时，A2A 怎么支持多轮对话？
3. 一个任务要跑两小时，调用方有哪几种方式拿到结果？

## 从一个场景开始

你在做车机语音助手。用户说：「帮我订明天上海的酒店。」

订酒店不是你们自己的能力，而是一家酒店集团提供的 Agent。它由另一个团队用另一套框架开发，部署在对方的服务器上，内部的 prompt、工具和数据都不会给你看。

你可能会想：把它包成一个 MCP 工具，调用一下不就行了？很快就会卡住：

- 酒店 Agent 需要反问「请问哪天入住？」，但工具调用是一问一答，没有地方接住这句反问。
- 订房可能要几分钟，工具调用只能干等。
- 你连对方「会什么」都不知道，只能靠人工对接文档。

A2A（Agent2Agent）就是为这种「Agent 委托 Agent」的场景设计的协议。

## 一句话总结

A2A 让两个互不了解内部实现的 Agent 能够互相发现、委派任务，并通过有状态的任务完成多轮协作。MCP 管「Agent 调工具」，A2A 管「Agent 找 Agent」，两者互补。

## 先跑起来

下面两个文件模拟上面的场景：`hotel_agent.py` 是酒店 Agent，`car_assistant.py` 是车机助手。客户端没用 SDK，直接发 HTTP 请求，方便你看到线上传输的原始格式。

```python
# hotel_agent.py —— 一个最小的 A2A「酒店 Agent」（a2a-sdk 1.2.2）
import uvicorn
from starlette.applications import Starlette
from a2a.types import AgentCard, AgentCapabilities, AgentInterface, AgentSkill
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events import EventQueue
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.tasks import InMemoryTaskStore, TaskUpdater
from a2a.server.routes import create_agent_card_routes, create_jsonrpc_routes
from a2a.helpers import new_task_from_user_message, new_text_part, get_message_text

card = AgentCard(  # 名片：别人靠它知道「我会什么、怎么找我」
    name="hotel-agent", description="帮用户预订酒店", version="1.0.0",
    supported_interfaces=[AgentInterface(url="http://localhost:9999",
                                         protocol_binding="JSONRPC", protocol_version="1.0")],
    capabilities=AgentCapabilities(streaming=True),
    default_input_modes=["text/plain"], default_output_modes=["text/plain"],
    skills=[AgentSkill(id="book", name="订酒店", description="按城市和日期预订酒店",
                       tags=["hotel"], examples=["帮我订上海的酒店"])])

class HotelExecutor(AgentExecutor):
    async def execute(self, context: RequestContext, event_queue: EventQueue):
        task = context.current_task
        is_new = task is None
        if is_new:  # 只有新任务才登记；续接时再登记会把旧状态当结果返回
            task = new_task_from_user_message(context.message)
            await event_queue.enqueue_event(task)
        up = TaskUpdater(event_queue, task.id, task.context_id)
        await up.start_work()
        if is_new:  # 第一轮：缺日期，反问
            await up.requires_input(up.new_agent_message([new_text_part("请问哪天入住？")]))
            return
        date = get_message_text(context.message).strip()
        await up.add_artifact([new_text_part(f"已预订上海外滩酒店，{date}入住")], name="booking")
        await up.complete()

    async def cancel(self, context: RequestContext, event_queue: EventQueue):
        await TaskUpdater(event_queue, context.task_id, context.context_id).cancel()

handler = DefaultRequestHandler(agent_executor=HotelExecutor(),
                                task_store=InMemoryTaskStore(), agent_card=card)
app = Starlette(routes=create_agent_card_routes(card) + create_jsonrpc_routes(handler, "/"))
if __name__ == "__main__":
    uvicorn.run(app, port=9999, log_level="warning")
```

```python
# car_assistant.py —— 车机助手：发现酒店 Agent → 委派任务 → 回答反问
import uuid, httpx

card = httpx.get("http://localhost:9999/.well-known/agent-card.json").json()
print("① 发现:", card["name"], "会", [s["name"] for s in card["skills"]])
url = card["supportedInterfaces"][0]["url"]

def send(text, task_id=None, context_id=None):
    msg = {"messageId": str(uuid.uuid4()), "role": "ROLE_USER", "parts": [{"text": text}]}
    if task_id: msg.update(taskId=task_id, contextId=context_id)
    body = {"jsonrpc": "2.0", "id": "1", "method": "SendMessage", "params": {"message": msg}}
    return httpx.post(url, json=body, headers={"A2A-Version": "1.0"}).json()["result"]["task"]

t1 = send("帮我订上海的酒店")
print("② 第一轮:", t1["status"]["state"], "|", t1["status"]["message"]["parts"][0]["text"])

t2 = send("10月20日", t1["id"], t1["contextId"])
print("③ 第二轮:", t2["status"]["state"], "|", t2["artifacts"][0]["parts"][0]["text"])
print("   同一个任务吗?", t1["id"] == t2["id"])
```

**运行**：`pip install "a2a-sdk[http-server]==1.2.2" httpx uvicorn`，先在一个终端运行 `python hotel_agent.py`，再在另一个终端运行 `python car_assistant.py`。

**实测输出**（2026-10-13，连续运行 3 次结果一致）：

```
① 发现: hotel-agent 会 ['订酒店']
② 第一轮: TASK_STATE_INPUT_REQUIRED | 请问哪天入住？
③ 第二轮: TASK_STATE_COMPLETED | 已预订上海外滩酒店，10月20日入住
   同一个任务吗? True
```

**该看什么**：
- 第①行：车机助手事先什么都不知道，读一张「名片」就知道对方会订酒店。
- 第②行：对方没有直接给结果，而是返回一个「需要补充信息」的任务，并附上反问。
- 第③行：车机助手带着同一个任务 ID 回答，任务从「等待输入」变成「完成」，结果放在单独的 `artifacts` 里，而不是混在对话里。

下面的核心概念，就是在解释这三行输出。

## 核心概念

### 1. A2A 和 MCP：一个横向，一个纵向

```
            A2A（横向：Agent 委托 Agent，对方是黑盒）
  车机助手 Agent ──────────────────────────────► 酒店 Agent
        │                                          │
        │ MCP（纵向：Agent 调用工具）                │ MCP
        ▼                                          ▼
   车控 MCP Server                           订房系统 MCP Server
```

- **是什么**：MCP 让一个 Agent 接入工具和数据；A2A 让一个 Agent 把任务交给另一个 Agent。
- **没有它会怎样**：只用 MCP 把远程 Agent 包成工具也能跑，但反问、长任务、取消这些能力都得自己补。
- **对应你熟悉的**：MCP 像在 App 里直接调用一个库；A2A 像通过 AIDL 调用另一个 App 的 Service，对方怎么实现你看不到，也不需要看到。

两者是互补关系：酒店 Agent 对外通过 A2A 接活，对内通过 MCP 接自己的订房系统。

### 2. Agent Card：能力发现

- **是什么**：每个 A2A Agent 在固定路径 `/.well-known/agent-card.json` 发布一张「名片」，写明名字、会哪些技能（skills）、地址和鉴权方式。
- **没有它会怎样**：每接一个外部 Agent，都要人工对接文档、手写配置。
- **对应你熟悉的**：`AndroidManifest` 里的 intent-filter。调用方先按「能干什么」查询，再决定交给谁，就像 `PackageManager.queryIntentActivities()`。

skills 里的描述会被调用方的大模型读来决定「委派给谁」，所以要像 Day 5 写工具描述一样写清楚。企业内部通常还会建一个注册中心，集中登记所有 Card。

### 3. Task：有状态的任务，支持多轮

- **是什么**：发出消息后，对方返回的是一个 Task，它有自己的 ID 和状态：已提交 → 处理中 → 完成 / 失败 / 已取消。中间可能停在「需要输入」或「需要授权」。
- **没有它会怎样**：对方无法反问，长任务也无从追踪进度。
- **对应你熟悉的**：WorkManager 的 `WorkInfo.State`，一项工作有 ID、有状态，可以查询，也可以取消。

多轮对话的关键是两个 ID：

| | 含义 | 什么时候带 |
|---|---|---|
| `taskId` | 一件具体的事（这次订房） | 回答对方的反问时，带上原来的 taskId |
| `contextId` | 一次会话，可以包含多件事 | 在同一会话里开始一件新事时，只带 contextId |

### 4. Artifact：过程和交付物分开

- **是什么**：任务的最终产出放在 `artifacts` 里，和过程中的对话分开存放。
- **没有它会怎样**：调用方得从整段对话里自己找结果；如果每一层 Agent 都转述一遍，信息会越传越失真。
- **补充**：Anthropic 在多 Agent 研究系统里也是这么做的：子 Agent 把大块结果写到外部存储，只把轻量的引用传回给协调者。A2A 中，大文件同样可以只传一个链接。

### 5. 长任务怎么拿结果

| 方式 | 适合场景 | 对应你熟悉的 |
|---|---|---|
| 同步等待（或先拿 Task 再轮询） | 秒级任务 | 普通的同步调用 |
| SSE 流式推送 | 需要实时进度，并且网络稳定 | 收集一个 Flow |
| Webhook 推送通知 | 小时级任务、移动端、弱网 | FCM 推送 |

车机场景要特别注意：弱网环境下不要让车机一直保持流式长连接，长任务优先用推送。

## 高频面试题

**1. A2A 和 MCP 有什么区别？能只用其中一个吗？**
- MCP：Agent 调工具，对方的输入输出格式完全透明，一次调用一个结果。
- A2A：Agent 委托 Agent，对方是黑盒，交互单位是有生命周期的 Task。
- 两者互补：A2A Agent 的内部通常用 MCP 接工具。
- 只用 MCP 也行，但多轮反问、任务状态、取消、推送都要自己实现。

**2.（追问）一个任务要跑两小时，调用方怎么拿结果？**
- 秒级任务用同步；也可以先拿到 Task，再用 `GetTask` 轮询。
- 需要实时进度且连接稳定时，用 SSE 流式；断线后可以重新订阅。
- 小时级任务或弱网环境用 Webhook 推送。
- 推送要做三件事：校验回调地址（防 SSRF）、回调带鉴权、调用方做幂等处理（推送可能重复送达）。

**3.（追问）contextId 和 taskId 有什么区别？**
- contextId 是一次会话，taskId 是会话里的一件具体的事。
- 回答对方的反问时带原 taskId；开始一件新事时只带 contextId。
- LangChain 的 Agent Server 直接把 contextId 当作 LangGraph 的 thread_id，所以它必须是 UUID。

**4.（场景题）客服 Agent 要调用财务部门的退款 Agent 和物流部门的查件 Agent，三个团队技术栈不同，你怎么设计？**
- 每个部门的 Agent 独立部署，对外只暴露 A2A，内部各自用 MCP 接自己的系统；用注册中心登记所有 Card。
- 服务间用 OAuth2 鉴权；退款这类高风险操作，由财务 Agent 返回「需要授权」或「需要输入」，触发人工确认。
- 退款请求带幂等键，并设置超时和取消；把 contextId 作为全链路追踪 ID。
- 对方返回的内容一律当作数据处理，不能直接变成上游 Agent 的指令（防注入）。

## 常见误区

1. **「A2A 是用来替代 MCP 的」**：两者解决不同层面的问题，生产系统里通常同时使用。面试时先说「互补」，再画出上面那张横向/纵向的图。
2. **把对方 Agent 的回复当成可信指令**：对方是黑盒，也可能被注入了恶意内容。它返回的文本只能当数据，高风险动作要在自己这边再校验一次。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：酒店 Agent 的任务生命周期**（取自上面的 `HotelExecutor`）

```python
# 定义执行器：A2A 框架收到消息后会调用它
class HotelExecutor(AgentExecutor):
    # 处理一次请求，状态变化通过事件队列发出去
    async def execute(self, context: RequestContext, event_queue: EventQueue):
        # 取出正在续接的任务；没有就说明是新请求
        task = context.current_task
        # 记下这是不是一个新任务
        is_new = task is None
        # 只有新任务才需要创建并登记
        if is_new:
            # 根据用户消息新建任务
            task = new_task_from_user_message(context.message)
            # 把新任务登记出去，状态为已提交
            await event_queue.enqueue_event(task)
        # 创建状态更新助手，绑定任务 ID 和会话 ID
        up = TaskUpdater(event_queue, task.id, task.context_id)
        # 状态切换为处理中
        await up.start_work()
        # 第一轮缺少日期，就反问调用方
        if is_new:
            # 状态切换为需要输入，并附上反问
            await up.requires_input(up.new_agent_message([new_text_part("请问哪天入住？")]))
            # 交出控制权，等对方补充信息后再被调用
            return
        # 第二轮：从用户消息里取出日期
        date = get_message_text(context.message).strip()
        # 把预订结果作为产出物挂到任务上
        await up.add_artifact([new_text_part(f"已预订上海外滩酒店，{date}入住")], name="booking")
        # 状态切换为完成
        await up.complete()
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 续接任务时不能再 `enqueue_event(task)`。否则第二轮返回的会是旧的「需要输入」状态，任务其实稍后才完成（本篇实测踩过这个坑）。
- `TaskUpdater` 的参数顺序是：队列、任务 ID、会话 ID，不要漏掉会话 ID。
- `requires_input` 之后必须 `return`，否则会继续往下执行到 `complete()`。

**昨日复习（只做第③步）**：Day 14 的「最小 MCP Server（tool + resource + prompt）」。只看你自己写过的注释，重写代码，重点核对导入路径 `mcp.server.mcpserver`、三个装饰器都带括号，以及抛 `ToolError` 而不是 `ValueError`。

## 动手练习（30 分钟）

在上面两个文件的基础上改：

1. 跑通后，在 `car_assistant.py` 末尾再发一条消息「再订一晚北京的」，**只带 contextId、不带 taskId**。打印新的 task ID，确认它和 t1 不同。用一句话写下 context 和 task 的区别。
2. 改 `hotel_agent.py`：第二轮如果用户回答的是空字符串，就再反问一次，而不是直接完成。
3. 思考题：如果让 Claude 自动决定把任务委派给哪个 A2A Agent，你会怎么把 Card 里的 `skills` 转成给模型看的工具描述？

## 读后自测

1. A2A 和 MCP 分别解决什么问题？为什么说它们是互补的？→ 见「核心概念 1」
2. 对方 Agent 需要反问时，A2A 怎么支持多轮对话？→ 见「先跑起来」的第②③行和「核心概念 3」
3. 一个任务要跑两小时，调用方有哪几种方式拿到结果？→ 见「核心概念 5」

---

## 参考区（用时查，不必通读）

### 细节速查（A2A 1.0，以 a2a-sdk 1.2.2 为准）

- **Agent Card 其他字段**：`provider`、`documentationUrl`、`iconUrl`、`securitySchemes`（API Key / HTTP Bearer / OAuth2 / OpenID Connect / mTLS）、`securityRequirements`、`signatures`（对 Card 签名防篡改）。`capabilities` 包括 `streaming`、`pushNotifications`、`extensions`、`extendedAgentCard`。
- **Message**：`messageId`、`contextId`、`taskId`、`role`（`ROLE_USER` / `ROLE_AGENT`）、`parts`、`metadata`、`referenceTaskIds`。
- **Part**：1.0 把旧版的 TextPart / FilePart / DataPart 合并成一个 Part，用 `text`、`raw`（字节）、`url`（文件链接）、`data`（结构化 JSON）区分，另有 `filename`、`mediaType`。
- **任务状态**：`TASK_STATE_SUBMITTED / WORKING / COMPLETED / FAILED / CANCELED / INPUT_REQUIRED / REJECTED / AUTH_REQUIRED`。0.3 版是小写短横线写法，如 `input-required`。
- **方法名**：

| 1.0 | 0.3 | 作用 |
|---|---|---|
| `SendMessage` | `message/send` | 发消息，返回 Task 或 Message |
| `SendStreamingMessage` | `message/stream` | SSE 流式返回 |
| `GetTask` / `ListTasks` | `tasks/get` / — | 查询任务 |
| `CancelTask` | `tasks/cancel` | 取消 |
| `SubscribeToTask` | `tasks/resubscribe` | 断线重新订阅 |
| `Create/Get/List/DeleteTaskPushNotificationConfig` | — | 管理推送 webhook |
| `GetExtendedAgentCard` | — | 鉴权后获取扩展版 Card |

- **传输方式**：JSON-RPC 2.0、HTTP+JSON（REST）、gRPC。请求头 `A2A-Version` 声明协议版本。
- **版本坑**：用 0.3 方法名 `message/send` 调用只支持 1.0 的 Server，返回 `-32601 Method not found`（已实测）。Server 端可以通过 `create_jsonrpc_routes(..., enable_v0_3_compat=True)` 打开兼容。动手前先看对方 Card 里的 `protocolVersion`。

### LangChain Agent Server 的 A2A 支持

- 每个 assistant 自动暴露在 `/a2a/{assistant_id}`，Card 地址为 `/.well-known/agent-card.json?assistant_id=...`。
- 实现的是 A2A 1.0 的 JSON-RPC 绑定，同时接受 4 个 0.3 方法名；gRPC 和 HTTP+JSON 尚未实现。
- 要求图的 state 里有 `messages` 键。
- `contextId` 直接当作 LangGraph 的 `thread_id`，必须是 UUID，否则返回 `-32602`；向已完成的 task 再发消息返回 `-32004`。
- `SubscribeToTask` 和推送配置目前返回 `-32601`。需要 `langgraph-api >= 0.4.21`；在 `langgraph.json` 中设置 `{"http": {"disable_a2a": true}}` 可关闭。

### Anthropic 多 Agent 系统的经验

- 委派任务时要写清楚：目标、输出格式、可用工具、任务边界。说明太模糊会导致子 Agent 误解任务或重复工作。
- 子 Agent 把结果写到外部存储，只回传引用，避免层层转述造成信息丢失。
- 同步等待子 Agent 是瓶颈；改为异步能提高并行度，但会带来状态协调和错误处理的难题。
- 多 Agent 系统大约消耗普通对话 15 倍的 token，只适合价值高的任务。

### 来源说明

A2A 协议细节属于「非 LangChain/Claude 来源」，来自官方 Python SDK `a2a-sdk` 1.2.2 的源码与实际运行结果；a2a-protocol.org 的规范正文本次未打开，建议自行对照阅读。

### 延伸学习

- [A2A endpoint in Agent Server](https://docs.langchain.com/langsmith/server-a2a)（LangChain）
- [Deploying multi-agent systems using MCP and A2A with Claude on Vertex AI](https://www.anthropic.com/webinars/deploying-multi-agent-systems-using-mcp-and-a2a-with-claude-on-vertex-ai)（Anthropic × Google Cloud）
- [Introduction to subagents](https://academy.claude.com/courses/introduction-to-subagents)（Claude Academy，Day 16 会用到）

### 参考资料

- [A2A endpoint in Agent Server - Docs by LangChain](https://docs.langchain.com/langsmith/server-a2a)
- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [A2A Protocol Specification 1.0](https://a2a-protocol.org/v1.0.0/specification/)（非 LangChain/Claude 来源，未打开正文）
- [a2aproject/a2a-samples](https://github.com/a2aproject/a2a-samples)（非 LangChain/Claude 来源）
