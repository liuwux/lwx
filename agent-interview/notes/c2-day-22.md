# Day 22 · 可观测性（第二轮：trace 跨服务串起来、采样不丢整棵树、敏感数据和线上打分）（2026-11-19）

## 今日 AI 动态（2026-11-19）

### 趋势观察

本期官方新内容只有 Google 的 Gemini 4 Neon。它的卖点不只是跑分，还包括「当编排者调度并行子 Agent」「长会话里可靠地压缩上下文」「间接提示注入攻击成功率 0%」。这说明厂商发布中型模型时，已经把编排、上下文管理和抗注入当作主要指标来报。推测：Agent 跑得越长、子 Agent 越多，trace 树越深，今天讲的「跨进程串 trace」会越来越常被问到。

### 今日要点

- Google 发布 Gemini 4 Neon，定位在 Carbon 和 Hydrogen 之间，可在 Antigravity 中作为编排者调度并行的 Hydrogen 子 Agent。
- Neon 定价为每百万 token 输入 $3、缓存输入 $0.15、输出 $15，上下文 1M，开发者可以在 Gemini API 付费层和 Android Studio 中使用。

### 分公司动态

#### 1. Google

**[Introducing Gemini 4 Neon](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-neon/)**（2026-11-18）
Gemini 4 家族的中间档：Carbon 负责最难的长程任务，Hydrogen 负责快速日常执行，Neon 兼顾推理深度和成本。Google 称它在高思考档位下单任务成本比 Carbon 低 32%。官方自报的成绩包括 Terminal-Bench 4.1 65.2%、Artificial Analysis Coding Agent Index 第 1（58.6）。最高思考档在 Gray Swan 间接提示注入基准上的攻击成功率为 0.0%。开发者可以通过 Gemini API 付费层（AI Studio）和 Android Studio 使用；Google AI Pro / Ultra 订阅用户可以在 Antigravity 和 Gemini app 中使用。
- 为什么重要：Android Studio 第一天就能用，而它主打的「编排者 + 并行子 Agent」「上下文压缩」「抗间接注入」正是企业 Agent 面试里多智能体、上下文工程、安全三类题的现实例子。

### 动态来源

- [Introducing Gemini 4 Neon - Google](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-neon/)

---

> 第二轮说明：第一轮 Day 22 讲过 trace / span 是一棵树、结构默认记、内容谨慎记、标签与元数据、从告警到评测用例的排障闭环。今天往下挖三件事：**trace 怎么跨进程不断开**、**采样怎么省钱又不丢关键信息**、**线上 trace 怎么自动打分**。

## 读前三问

1. 车机端发请求，云端 Agent 处理，两边各自打了 trace。怎样让它们在后台显示成一棵树？
2. 线上流量太大，只能留 10% 的 trace。采样是按什么单位决定的？出错的请求被丢了怎么办？
3. trace 里会有用户手机号、行程地址。哪些数据该记、怎么打码，而且不影响排障？

## 从一个场景开始

车主说：「给老婆发短信，说我晚到 10 分钟。」车机上的语音助手把请求发给云端 Agent，云端查联系人、调短信工具。

第二天测试反馈：短信发给了错误的号码。

你打开 LangSmith，发现有两条互不相干的 trace：一条叫 `car_voice_request`，只有一个节点，什么也看不出来；另一条是云端的 `cloud_agent`，但不知道它对应的是哪辆车、哪一句话。早高峰时同一分钟有上千条云端 trace，根本对不上号。

更糟的是，为了省钱你开了 10% 采样，出问题的那一条很可能根本没被记下来。法务还提醒：trace 里有明文手机号，不能这样存。

今天要解决的就是这三件事：**串起来**、**采得巧**、**记得安全**。

## 一句话总结

跨服务时把 trace 上下文放进请求头传下去，让下游接成子节点；采样按整条 trace 决定，关键流量用条件追踪全量保留；敏感内容默认不记，要记就先打码；再用线上评委对采样到的 trace 自动打分、触发告警。

## 先跑起来

两个文件。`trace_demo.py` 在一个进程里模拟「车机端 → HTTP → 云端 Agent」，用官方的 `to_headers()` 和 `tracing_context(parent=...)` 把两端接成一棵树，并用 `create_anonymizer` 给手机号打码。`sampling.py` 在 10% 采样率下跑 1000 轮 Agent，看保留下来的是不是整棵树。

没有 LangSmith API Key，所以写了一个 `RecordingClient`：它继承真实的 `Client`，只把「采样、打码之后准备上传的数据」存到内存里。采样和打码都是 SDK 自己的逻辑。

```python
# trace_demo.py —— 车机端 → 云端 Agent：跨服务串起一条 trace，并给手机号打码（langsmith 0.15）
import langsmith as ls
from langsmith.anonymizer import create_anonymizer
from langsmith.run_helpers import get_current_run_tree

class RecordingClient(ls.Client):                  # 离线替身：只记录「将要上传」的数据，不联网
    def __init__(self, **kw):
        super().__init__(api_url="http://localhost:1984", api_key="offline",
                         auto_batch_tracing=False, info={}, **kw)
        self.sent = {}
    def _create_run(self, run, **kw): self.sent.setdefault(str(run["id"]), {}).update(run)
    def _update_run(self, run, **kw): self.sent.setdefault(str(run["id"]), {}).update(run)

# ---------- 云端服务 ----------
@ls.traceable(run_type="tool")
def send_sms(phone: str, text: str):
    return f"已发送给 {phone}"

@ls.traceable(name="cloud_agent")
def cloud_agent(query: str):
    return send_sms("13812345678", "我晚到 10 分钟")

def http_handler(headers: dict, body: dict):        # 相当于 FastAPI 路由
    with ls.tracing_context(parent=headers):        # 用请求头接上游的 trace
        return cloud_agent(body["query"])

# ---------- 车机端 ----------
@ls.traceable(name="car_voice_request")
def car_side(query: str):
    headers = get_current_run_tree().to_headers()    # 把当前 trace 放进请求头
    print("请求头:", list(headers))
    return http_handler(headers, {"query": query})  # 真实场景是一次 HTTP 调用

if __name__ == "__main__":
    mask = create_anonymizer([{"pattern": r"1[3-9]\d{9}", "replace": "<手机号>"}])
    client = RecordingClient(anonymizer=mask)
    ls.configure(client=client, enabled=True)
    print("返回给车主:", car_side("给老婆发短信说我晚到"))
    client.flush()
    for r in sorted(client.sent.values(), key=lambda r: r["dotted_order"]):
        pid = str(r.get("parent_run_id") or "----")[-4:]
        print("  " * r["dotted_order"].count(".") + f"{r['name']} id={str(r['id'])[-4:]} "
              f"trace={str(r['trace_id'])[-4:]} 父={pid} 输入={r['inputs']}")
```

```python
# sampling.py —— 采样率 10%：按 trace 整棵保留或丢弃，而且决定可复现
import langsmith as ls
from collections import Counter
from trace_demo import RecordingClient

client = RecordingClient(tracing_sampling_rate=0.1)
ls.configure(client=client, enabled=True)

@ls.traceable(run_type="llm")
def call_llm(q): return "ok"

@ls.traceable(run_type="tool")
def open_window(): return "车窗已开"

@ls.traceable(name="agent_turn")                     # 一轮 = 根 run + 3 个子 run
def agent_turn(q):
    call_llm(q); open_window(); return call_llm(q)

for i in range(1000):
    agent_turn(f"请求{i}")
client.flush()

per_trace = Counter(str(r["trace_id"]) for r in client.sent.values())
print("1000 次请求，上传的 trace 数:", len(per_trace))
print("每条 trace 上传的 run 数:", Counter(per_trace.values()))
print("同一 trace_id 再判一次结果一致:", all(client._should_sample(t) for t in per_trace))
```

**运行**：`pip install langsmith && python trace_demo.py && python sampling.py`

**实测输出**（2026-11-19，Python 3.13，langsmith 0.15.0；ID 是随机生成的，每次不同）：

```
请求头: ['langsmith-trace', 'baggage']
返回给车主: 已发送给 13812345678
car_voice_request id=4088 trace=4088 父=---- 输入={'query': '给老婆发短信说我晚到'}
  cloud_agent id=40f5 trace=4088 父=4088 输入={'query': '给老婆发短信说我晚到'}
    send_sms id=ed44 trace=4088 父=40f5 输入={'phone': '<手机号>', 'text': '我晚到 10 分钟'}
```

```
1000 次请求，上传的 trace 数: 96
每条 trace 上传的 run 数: Counter({4: 96})
同一 trace_id 再判一次结果一致: True
```

**该看什么**：
- 三个节点的 `trace` 都是 `4088`，云端 `cloud_agent` 的父节点就是车机端的根节点。两端接成了一棵树，靠的只是请求头里的 `langsmith-trace`。
- 车主听到的回复里号码是完整的，上传的 trace 里号码变成了 `<手机号>`。打码只作用于观测数据，不影响业务。
- 采样后留下约 10% 的 trace（多跑几次在 75~110 之间波动），而且每条 trace 都是完整的 4 个 run，没有「只剩半棵树」的情况。

## 核心概念

### 1. 上下文传播：trace 跨进程不断开

- **是什么**：上游把「我是哪条 trace 的哪个节点」写进请求头，下游读出来，把自己的 run 挂在它下面。
- **没有它会怎样**：每个服务各自生成一条孤立的 trace，排障时只能靠时间戳去猜对应关系。
- **对应你熟悉的**：Android 里 `startActivityForResult` 时把 requestCode 带过去，回来才知道是哪次请求。

LangSmith 用两个头：`langsmith-trace`（父节点位置）和 `baggage`（元数据、标签）。OpenTelemetry 世界用的是 W3C 标准的 `traceparent`。Claude Agent SDK 会自动把当前 OTel 上下文写成环境变量 `TRACEPARENT` 传给 CLI 子进程，所以 Agent 的 `claude_code.interaction` span 会挂在你应用的 span 下面；子 Agent 的 span 也会嵌在父 Agent 的 tool span 里。

**安全提醒**（官方原话的意思）：只接受来自可信服务的追踪头。公网入口要在网关把这些头剥掉，否则外部请求可以把自己伪装成你内部 trace 的一部分。

### 2. 采样：按 trace 决定，关键流量别靠运气

- **是什么**：只上传一部分 trace 以控制成本。LangSmith 用 `LANGSMITH_TRACING_SAMPLING_RATE` 或 `Client(tracing_sampling_rate=...)` 设置。
- **没有它会怎样**：全量记录在高峰期会让存储和费用失控。
- **对应你熟悉的**：Firebase Performance 只上报一部分会话，但崩溃日志 Crashlytics 是全量的。

本例看到两个关键事实（来自 SDK 源码 `is_sampled_by_id`）：采样按 `trace_id` 哈希决定，**一棵树要么全留、要么全丢**；同一个 ID 在任何进程、任何 SDK 里得出的结论都一样，所以跨服务的同一条 trace 不会出现「车机端留了、云端丢了」。

代价是：这是**请求开始时就决定的**，出错的请求一样可能被丢掉。补救办法是：错误率、延迟这类**指标**单独全量上报，不受 trace 采样影响；重要租户、灰度流量用**条件追踪**（`tracing_context(enabled=...)`）强制全记；零留存客户则强制不记。

### 3. 敏感数据：默认只记结构，内容要主动开

- **是什么**：trace 默认记录耗时、模型名、工具名，提示词和工具参数等内容要显式开启；需要记录时，先打码再上传。
- **没有它会怎样**：观测平台变成一个存着全部用户隐私的数据库，权限稍松就是数据泄露。
- **对应你熟悉的**：上架前检查 Logcat 有没有打印 token 和手机号，Release 包关掉详细日志。

Claude Agent SDK 就是这么设计的：默认只导出结构，`OTEL_LOG_USER_PROMPTS`、`OTEL_LOG_TOOL_DETAILS` 等开关要逐个打开。LangSmith 提供三档：整段隐藏（`hide_inputs`）、按正则打码（`create_anonymizer`）、按函数单独处理（`@traceable(process_inputs=...)`）。正则不可能覆盖所有情况，要对真实样本测过再上线。

### 4. 线上评估器：让 trace 自己报警

- **是什么**：对线上 trace 按条件过滤、按比例抽样，交给 LLM 评委或代码自动打分，分数作为 feedback 挂回 trace。
- **没有它会怎样**：trace 存了一堆，只有用户投诉后才有人去看。
- **对应你熟悉的**：Play 后台的 ANR 率告警：不是等用户差评，而是指标越线就通知你。

LangSmith 的在线评估器可以只评「用户点了踩」「调用了某个工具」「某个 metadata 值」的 run，并可以设置采样率控制评委开销；也有按整段对话打分的线程级评估器。打分之后可以再接一条 webhook 自动化规则，分数低于阈值就发告警。注意：被评估过的 trace 会升级为长期保留，这会影响费用。

## 高频面试题

**1. 端云协同的 Agent，怎么让一次请求在观测平台里是一棵完整的树？**
- 上游从当前 run 生成追踪头（LangSmith 的 `langsmith-trace` 或 W3C `traceparent`），随 HTTP 请求带过去。
- 下游用中间件或 `tracing_context(parent=headers)` 接上，后续 run 自动成为子节点。
- 子进程（如 Agent SDK 拉起的 CLI）用环境变量 `TRACEPARENT` 传递。
- 只信任内部服务的追踪头，公网入口在网关剥除。

**2.（追问）开了 10% 采样，偏偏出错的请求没被记下来，怎么办？**
- 采样在请求开始时按 trace_id 决定，本来就不知道会不会出错。
- 错误率、延迟、token 用量走指标通道全量上报，用来发现问题。
- 关键租户、新版本灰度流量用条件追踪全量记录，用来定位问题。
- 可以按需临时调高采样率复现，排障结束再调回。

**3.（追问）为什么采样要按整条 trace 决定，而不是每个 span 各自随机？**
- 各自随机会留下「有父无子」或「有子无父」的半棵树，无法排障。
- 按 trace_id 哈希决定，同一条 trace 在所有服务里结论一致。
- LangSmith 文档也提醒：父节点没上传的子 span 会在缓冲期后被静默丢弃。

**4.（场景题）车机语音 Agent 上线，trace 里会有手机号和导航目的地，你怎么设计观测方案？**
- 默认只记结构：节点、耗时、工具名、token、错误。
- 需要内容时用正则打码手机号，用 `process_inputs` 把地址只保留城市级别，并用真实样本测试打码规则。
- 用 metadata 记录车型、版本、车架号后几位，方便按维度筛选，但不记完整身份信息。
- 在线评估器只抽样评「用户重复说了一遍」「调了发短信工具」的 trace，低分走 webhook 告警。

## 常见误区

1. **「打码会让返回给用户的结果也变成 `<手机号>`」**：不会。本例中车主听到的回复是完整号码，打码只作用于上传到观测平台的数据。反过来，也别指望打码能防止模型把隐私说出去，那是护栏的工作。
2. **「采样率调低了，告警也会跟着漏」**：只要告警是基于指标的（错误率、延迟），就不受 trace 采样影响。把「发现问题」和「定位问题」分开设计：前者用全量指标，后者用采样的 trace。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：跨服务传递 trace 上下文 + 打码**（取自 `trace_demo.py`）

```python
# 云端：一个被追踪的工具，类型标成 tool
@ls.traceable(run_type="tool")
def send_sms(phone: str, text: str):
    # 返回发送结果（里面带着手机号）
    return f"已发送给 {phone}"

# 云端：路由函数，参数是请求头和请求体
def http_handler(headers: dict, body: dict):
    # 用请求头当父节点，把下面的调用挂进上游的 trace
    with ls.tracing_context(parent=headers):
        # 在这个上下文里运行云端 Agent
        return cloud_agent(body["query"])

# 车机端：根节点，起一个好认的名字
@ls.traceable(name="car_voice_request")
def car_side(query: str):
    # 取当前 run，转成请求头
    headers = get_current_run_tree().to_headers()
    # 带着请求头调用云端
    return http_handler(headers, {"query": query})

# 打码规则：中国大陆手机号替换成占位符
mask = create_anonymizer([{"pattern": r"1[3-9]\d{9}", "replace": "<手机号>"}])
# 把打码器交给 Client，上传前生效
client = RecordingClient(anonymizer=mask)
# 全局使用这个 Client，并打开追踪
ls.configure(client=client, enabled=True)
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `get_current_run_tree()` 后面要接 `.to_headers()`，返回的是字典，不是字符串。
- `tracing_context(parent=headers)` 的参数名是 `parent`，不是 `headers` 或 `parent_run`。
- `create_anonymizer` 接收的是**列表**，每项是含 `pattern` 和 `replace` 两个键的字典；正则字符串前面要加 `r`。

**昨日复习（只做第③步）**：第二轮 Day 21 的「配对比较 + 置信区间」。只看你自己写的注释重写，重点核对：标准误除以 `len(xs) ** 0.5`、`zip(a1, b1)` 后是 `b - a`、不配对公式用 `st.variance`。

## 动手练习（30 分钟）

在今天两个文件的基础上改：

1. 在 `car_side` 的 `@ls.traceable` 里加 `metadata={"car_model": "X7", "app_version": "3.2.0"}`，看云端子节点的 `extra` 里能不能拿到这些 metadata（提示：它们通过 `baggage` 头传递）。
2. 给 `send_sms` 加 `process_inputs`：手机号只保留后 4 位，短信正文只记长度。对比它和 `anonymizer` 同时存在时谁生效。
3. 在 `sampling.py` 里给 `agent_turn` 加一个参数 `tenant`，用 `tracing_context(enabled=...)` 实现「VIP 租户全量记录，其他租户按 10% 采样」，统计两类租户各上传了多少条。

## 读后自测

1. 车机端发请求，云端 Agent 处理，两边各自打了 trace。怎样让它们在后台显示成一棵树？→ 见第 7 节「核心概念 1」
2. 线上流量太大，只能留 10% 的 trace。采样是按什么单位决定的？出错的请求被丢了怎么办？→ 见第 7 节「核心概念 2」
3. trace 里会有用户手机号、行程地址。哪些数据该记、怎么打码，而且不影响排障？→ 见第 7 节「核心概念 3」

## AI 历史知识点：DeepSeek-V3（2024，DeepSeek-AI）

**是什么**：一个混合专家（MoE）大模型，总参数 671B，每个 token 只激活其中 37B，上下文 128K。它沿用了 V2 的多头潜在注意力（MLA）和 DeepSeekMoE 架构，新增了「不靠辅助损失的专家负载均衡」和「多 token 预测（MTP）」训练目标。

**何时由谁提出**：DeepSeek-AI 于 2024 年 12 月 27 日在 arXiv 发布技术报告（2412.19437），权重在 Hugging Face 的 deepseek-ai 组织下开放，代码采用 MIT 许可，模型协议允许商用。

**为什么重要**：报告称完整训练只用了 278.8 万 H800 GPU 小时，预训练 14.8T token，并在超大规模上验证了 FP8 混合精度训练；整个训练过程没有出现不可恢复的 loss 尖峰，也没有回滚。它让「开源模型以较低训练成本接近闭源前沿模型」成为业界公认的事实，后来的 DeepSeek-R1 就是在 V3 基座上训练的。

**和今天的联系**：MoE 模型每次只走一部分专家，推理成本和延迟更难预估，所以线上更需要按请求记录 token 用量和耗时。V3 的报告把训练中「没有 loss 尖峰、没有回滚」当作卖点，本质上也是长期监控训练指标的结果，和今天讲的「指标全量、trace 采样」是同一种工程思路。

**原始来源**：[DeepSeek-V3 Technical Report（Hugging Face Papers）](https://huggingface.co/papers/2412.19437)、[deepseek-ai/DeepSeek-V3 README](https://huggingface.co/deepseek-ai/DeepSeek-V3/blob/main/README.md)

## 参考区（用时查，不必通读）

### 细节速查

**LangSmith 分布式追踪**
- 请求头：`langsmith-trace`（父 run 的 dotted_order）、`baggage`（metadata、tags、project 等，可选）。
- 上游：`get_current_run_tree().to_headers()`；下游三选一：`TracingMiddleware`（FastAPI/Starlette，langsmith ≥ 0.1.133）、`tracing_context(parent=request.headers)`、`fn(..., langsmith_extra={"parent": request.headers})`。

**采样与条件追踪**
- `LANGSMITH_TRACING_SAMPLING_RATE`（0~1）或 `Client(tracing_sampling_rate=...)`；不设置则全量记录。
- SDK 0.15 源码：`is_sampled_by_id` 用 XXH3-128 对 trace_id 哈希，Python 和 JS SDK 结论一致；采样率会写进 metadata `ls_tracing_sample_rate`。
- 优先级：`tracing_context(enabled=...)` > `ls.configure(enabled=...)` > `LANGSMITH_TRACING`，嵌套时内层优先。
- `tracing_context` 还能设置 `address=ls.agent_address(...)`、`tags`、`metadata`，可按租户路由。

**打码**
- 环境变量：`LANGSMITH_HIDE_INPUTS` / `LANGSMITH_HIDE_OUTPUTS` / `LANGSMITH_HIDE_METADATA`。
- `Client(hide_inputs=fn, hide_outputs=fn, hide_metadata=True|fn, anonymizer=create_anonymizer([...]))`；`create_anonymizer` 默认检查 10 层嵌套（`max_depth` 可调），需 Python SDK ≥ 0.1.81。
- 优先级：`HIDE_*=true` 时跳过该侧 anonymizer；anonymizer 优先于 `hide_inputs`；`@traceable(process_inputs=..., process_outputs=...)` 优先于 Client 级设置（`process_outputs` 需 ≥ 0.1.98）。

**LangSmith 与 OpenTelemetry**
- `LANGSMITH_OTEL_ENABLED=true`（需 `langsmith[otel]`，建议 ≥ 0.4.25）；`LANGSMITH_OTEL_ONLY=true` 只走 OTel。
- OTLP 端点 `https://api.smith.langchain.com/otel`，头 `x-api-key`、`Langsmith-Project`。
- 属性映射：`langsmith.span.kind` → run 类型；`langsmith.metadata.{key}`；`gen_ai.request.model`、`gen_ai.usage.input_tokens` 等。

**Claude Agent SDK 遥测**
- 开关：`CLAUDE_CODE_ENABLE_TELEMETRY=1`；trace 需再加 `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` 与 `OTEL_TRACES_EXPORTER=otlp`；指标和日志事件分别用 `OTEL_METRICS_EXPORTER`、`OTEL_LOGS_EXPORTER`。
- span 名：`claude_code.interaction` → `claude_code.llm_request` / `claude_code.tool`（含 `tool.blocked_on_user`、`tool.execution`）/ `claude_code.hook`。
- 默认导出间隔：指标 60 秒，trace 和日志 5 秒；导出失败默认静默，`CLAUDE_CODE_OTEL_DIAG_STDERR=1` 可显示错误。
- 不要在 SDK 下用 `console` 导出器（会占用 stdout 消息通道）。
- 内容开关：`OTEL_LOG_USER_PROMPTS`、`OTEL_LOG_TOOL_DETAILS`、`OTEL_LOG_TOOL_CONTENT`（默认截断 60 KB）、`OTEL_LOG_RAW_API_BODIES`。
- 终端用户归属：`OTEL_RESOURCE_ATTRIBUTES="enduser.id=...,tenant.id=..."`，值需百分号编码。

**LangSmith 在线评估器**
- 位置：Tracing → 项目 → Evaluators → + Evaluator；可设置过滤条件、采样率，创建时可回填历史 run。
- 有 run 级和线程级（多轮）两种；每个评估器有每周 LLM 花费上限，达到后暂停到周一 0 点（UTC）。

### 来源说明

- 示例用离线的 `RecordingClient` 截获将要上传的数据（重写了 SDK 内部方法 `_create_run` / `_update_run`，仅用于教学），没有连接 LangSmith 服务端；采样、打码、上下文传播都是 SDK 原逻辑。
- 「按 trace_id 哈希采样」来自 langsmith 0.15.0 源码，官方文档页没有明确写出；Claude Academy 暂无专门讲可观测性的课程。

### 延伸学习

- Claude Agent SDK 文档 · Observability with OpenTelemetry；Track cost and usage
- LangSmith 文档 · Implement distributed tracing；Set up LLM-as-a-judge online evaluators

### 参考资料

- [Implement distributed tracing - Docs by LangChain](https://docs.langchain.com/langsmith/distributed-tracing)
- [Set a sampling rate for traces - Docs by LangChain](https://docs.langchain.com/langsmith/sample-traces)
- [Conditional tracing - Docs by LangChain](https://docs.langchain.com/langsmith/conditional-tracing)
- [Prevent logging of sensitive data in traces - Docs by LangChain](https://docs.langchain.com/langsmith/mask-inputs-outputs)
- [Trace with OpenTelemetry - Docs by LangChain](https://docs.langchain.com/langsmith/trace-with-opentelemetry)
- [Set up LLM-as-a-judge online evaluators - Docs by LangChain](https://docs.langchain.com/langsmith/online-evaluations)
- [Observability with OpenTelemetry - Claude Code Docs（Agent SDK）](https://code.claude.com/docs/en/agent-sdk/observability)
