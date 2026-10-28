# Day 30 · 系统设计题（二）+ 总复习：Coding / 深度研究 Agent（2026-10-28）

## 今日 AI 动态（2026-10-28）

### 趋势观察

过去 24 小时关注的厂商里只有 Google 有一条新的官方发布：Gemini Robotics ER 2 在 Google Cloud 上开放公开预览。它的定位是机器人的「高层大脑」，负责理解视频、规划长任务，再调用底层控制模型和导航接口。这和软件 Agent 的「规划 + 调工具」是同一种结构，只是工具换成了机械臂。推测：「大模型规划、专用模型执行」的分层写法，会从软件 Agent 扩展到更多实体设备场景，车载也在其中。

### 今日要点

- Google 在 Google Cloud 上开放 Gemini Robotics ER 2 公开预览，这是给机器人做高层规划的模型，可调用 VLA 和导航等底层工具。

### 分公司动态

#### 1. Google

**[Gemini Robotics Embodied Reasoning 2 is now available on Google Cloud](https://cloud.google.com/blog/products/ai-machine-learning/gemini-robotics-embodied-reasoning-2-is-now-available-on-google-cloud)**（2026-10-27）

Gemini Robotics ER 2 是专门给机器人用的 Gemini 模型，现已在 Google Cloud 的 Model Garden 公开预览。它能理解连续视频里的空间和时间关系，做长任务规划，并通过函数调用指挥底层的 VLA 模型（视觉-语言-动作模型）和导航接口。官方给的例子包括：从视频里判断「什么时候该停止倒水」，关键帧定位准确率 91.3%；读取十多种仪表读数；多台机器人共享对工作区的理解并交接任务。企业部署方面支持 IAM 权限、边缘部署和 BigQuery 车队遥测。

**为什么重要**：这是「规划模型 + 工具调用」的 Agent 结构在实体机器人上的产品化，函数调用、权限控制、遥测这些软件 Agent 的工程能力也一并带了过去。

### 动态来源

- [Gemini Robotics Embodied Reasoning 2 is now available on Google Cloud](https://cloud.google.com/blog/products/ai-machine-learning/gemini-robotics-embodied-reasoning-2-is-now-available-on-google-cloud)

---

## 读前三问

1. 面试官说「设计一个深度研究 Agent」，为什么常用「主 Agent + 多个子 Agent」？代价是什么？
2. Coding Agent 怎么知道自己「改对了」？
3. 一个要跑几小时的 Agent 任务中途挂了，怎么接着跑而不是从头再来？

## 从一个场景开始

你在车机团队。产品经理丢来一句话：「语音助手冷启动太慢，查清楚原因，顺手把代码改了。」

你先试了个最简单的 Agent：一个模型，配上搜索文档、读代码、改文件三个工具。结果：

- 它搜了 20 篇文档，原文全塞进上下文，到第 15 轮已经忘了最初要查什么。
- 它改完代码说「已修复」，但没跑测试，CI 上一堆红。
- 任务跑了 40 分钟，网络抖了一下，所有进度都没了，只能重来。

这三个问题分别对应今天的三块内容：**分工**（主 Agent 只做调度，脏活交给子 Agent）、**验证**（用测试判断改没改对）、**外部状态**（进度写在上下文之外）。这也是把前 29 天的知识串起来的一道综合题。

## 一句话总结

深度研究和 Coding Agent 都是长任务：主 Agent 拆任务、派子 Agent 并行干活、只收结论；改代码靠测试确认结果；进度存在文件和 git 里，断了能接着跑。

## 先跑起来

用 LangChain 的 Deep Agents（`deepagents` 包）搭一个最小研究 Agent：主 Agent 把调研派给子 Agent `researcher`，拿回结论后把报告写进文件。模型换成按剧本出牌的假模型，不需要 API Key，只看流程。

```python
# research_agent.py —— 深度研究 Agent：主 Agent 派子 Agent 查资料，再把报告写进文件（剧本模型代替真实 LLM）
from deepagents import create_deep_agent
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage, ToolMessage
from langchain_core.outputs import ChatGeneration, ChatResult

DOCS = {"冷启动": "[D1] 车机语音冷启动 3.2 秒，其中模型加载占 2.1 秒；[D2] 预加载可降到 0.9 秒。"}
def search_docs(query: str) -> str:
    """检索内部技术文档，返回带编号的片段。"""
    return DOCS["冷启动"]

class Scripted(GenericFakeChatModel):  # 剧本模型：根据最后一条消息决定下一步
    def bind_tools(self, tools, **kw): return self
    def _generate(self, messages, stop=None, run_manager=None, **kw):
        last = messages[-1]
        call = lambda name, args, i: AIMessage("", tool_calls=[{"name": name, "args": args, "id": i}])
        if not isinstance(last, ToolMessage):
            if "子任务" in last.content: msg = call("search_docs", {"query": "冷启动"}, "s1")  # 子 Agent 收到任务
            else: msg = call("task", {"subagent_type": "researcher", "description": "子任务：查语音冷启动耗时构成"}, "t1")
        elif last.name == "search_docs": msg = AIMessage("结论：模型加载是瓶颈（2.1/3.2 秒）[D1]，预加载可到 0.9 秒 [D2]")
        elif last.name == "task": msg = call("write_file", {"file_path": "/report.md", "content": "# 冷启动报告\n" + last.content}, "w1")
        else: msg = AIMessage("报告已写入 /report.md")
        return ChatResult(generations=[ChatGeneration(message=msg)])

agent = create_deep_agent(
    model=Scripted(messages=iter([])),
    system_prompt="你是研究主管：拆分问题、派子 Agent 调研、汇总成带引用的报告。",
    subagents=[{"name": "researcher", "description": "负责检索文档并给出带引用的结论",
                "system_prompt": "只用 search_docs 查资料，结论必须带 [D编号]。", "tools": [search_docs]}],
)
r = agent.invoke({"messages": [("user", "车机语音助手冷启动为什么慢？出一份报告")]})
for m in r["messages"]:
    print(f"{type(m).__name__:12}", m.tool_calls[0]["name"] if getattr(m, "tool_calls", None) else m.content[:40])
print("文件:", r["files"]["/report.md"]["content"])
```

**运行**：`pip install -U deepagents && python research_agent.py`

**实测输出**（2026-10-28，Python 3.13，deepagents 0.7.32，langchain 1.4.7）：

```
HumanMessage 车机语音助手冷启动为什么慢？出一份报告
AIMessage    task
ToolMessage  结论：模型加载是瓶颈（2.1/3.2 秒）[D1]，预加载可到 0.9 秒 [D
AIMessage    write_file
ToolMessage  Updated file /report.md
AIMessage    报告已写入 /report.md
文件: # 冷启动报告
结论：模型加载是瓶颈（2.1/3.2 秒）[D1]，预加载可到 0.9 秒 [D2]
```

**该看什么**：
- 主 Agent 的消息里**没有** `search_docs`。子 Agent 查资料的过程留在它自己的上下文里，主 Agent 只拿到一句结论。这就是「上下文隔离」。
- 委派用的是内置工具 `task`，参数 `subagent_type` 指定派给谁，`description` 写任务内容。
- 报告写进了 Agent 的文件系统（`r["files"]`），没有留在对话里。长任务的中间产物都应该这样放到上下文之外。

## 核心概念

### 1. 编排者-工作者：主 Agent 调度，子 Agent 干活

- **是什么**：一个主 Agent 负责拆问题、派任务、汇总；多个子 Agent 各自在独立上下文里搜索、阅读，只把结论交回来。
- **没有它会怎样**：单个 Agent 把所有搜索原文都堆进上下文，越往后越乱，还只能一件件串行地查。
- **对应你熟悉的**：主线程只更新 UI，耗时任务丢给 `WorkManager` / 协程，只回调结果。

Anthropic 的多 Agent 研究系统就是这个结构：主 Agent 用 Opus 4，子 Agent 用 Sonnet 4，在内部研究评测上比单个 Opus 4 高出 90.2%。另有一个专门的引用 Agent，负责给结论标出处。

### 2. 投入要和问题难度匹配

- **是什么**：简单事实题 1 个 Agent、3–10 次工具调用就够；对比题派 2–4 个子 Agent；复杂研究才派 10 个以上。
- **没有它会怎样**：问一个简单问题也派 10 个子 Agent，token 和时间成倍浪费。
- **对应你熟悉的**：图片加载按尺寸选采样率，不是每张都解码原图。

Anthropic 的数据：Agent 大约用聊天的 4 倍 token，多 Agent 系统大约 15 倍；在 BrowseComp 上，token 用量一项就解释了 80% 的效果差异。所以多 Agent 适合「价值高、能并行、信息量超过单个上下文」的任务。改代码这类强耦合任务，通常不适合拆给很多子 Agent 并行。

### 3. Coding Agent 的验证闭环

- **是什么**：按「探索 → 计划 → 改代码 → 跑测试验证 → 提交」的顺序做，测试结果就是判断改没改对的标准。
- **没有它会怎样**：Agent 一上来就改，改完自称「已修复」，实际没跑过。
- **对应你熟悉的**：提 MR 前本地先跑 `./gradlew test`，CI 绿了才合入。

Claude Code 官方最佳实践强调两点：先读代码、先出计划，否则模型容易直接动手写；先写好测试、确认它失败，再让模型改到测试通过，「有明确目标可以对照时，Claude 表现最好」。Anthropic 的长任务实践还发现，只跑单元测试或 curl 不够，让 Agent 像真人一样做端到端测试，能多发现很多问题。

### 4. 长任务的状态放在上下文之外

- **是什么**：把计划、进度、功能清单写进文件，用 git 提交保存每个可用版本，每次会话先读进度再干活。
- **没有它会怎样**：上下文满了或进程挂了，前面的进度全部丢失，只能从头再来。
- **对应你熟悉的**：`onSaveInstanceState` + Room：进程被杀，回来时恢复现场。

Anthropic 的长任务方案分两种角色：「初始化 Agent」先把需求展开成功能清单（JSON，全部标为未通过），再写进度文件、做首次提交；之后每轮「编码 Agent」只做一个功能，测通后才标为通过，并写进度、提交 git。多 Agent 研究系统也会把计划存进记忆，因为上下文超过 20 万 token 会被截断；出错时从断点恢复，而不是重启。

### 5. 30 天知识串联

一道系统设计题可以把 30 天的内容串起来：

| 设计环节 | 关键问题 | 对应天数 |
|---|---|---|
| 要不要用 Agent | 流程固定就用 Workflow | Day 3、7 |
| 工具与协议 | 工具描述、MCP、权限 | Day 4–5、14 |
| 循环与规划 | ReAct、计划、反思 | Day 6–8 |
| 知识与记忆 | RAG、上下文工程、记忆 | Day 9–13 |
| 多 Agent | 主从分工、通信 | Day 15–18 |
| 可靠性 | 结构化输出、评测、可观测 | Day 20–22 |
| 安全与上线 | 护栏、人工审批、成本、部署 | Day 23–26 |

## 高频面试题

**1.（场景题）设计一个深度研究 Agent。**
- 主 Agent 先出研究计划（可让用户确认），存进记忆；按难度决定派几个子 Agent。
- 子 Agent 并行检索，先宽后窄，只回传带出处的结论。
- 汇总后由引用环节核对出处，产出报告文件。
- 评测：先用约 20 条真实问题起步，用 LLM 评分 + 人工抽查。

**2.（追问）多 Agent 这么好，为什么不全用多 Agent？**
- token 消耗约为聊天的 15 倍，低价值问题不划算。
- 子 Agent 之间不共享上下文，强耦合任务（如改代码）容易互相冲突。
- 同步等待子 Agent 会形成瓶颈，调试和观测也更难。
- 先把单 Agent 做到位，确认上下文或并行是瓶颈再拆。

**3. Coding Agent 怎么保证改对了？**
- 先探索、出计划，人确认后再改。
- 先写测试，确认失败，再让 Agent 改到通过。
- 除单元测试外做端到端验证；禁止 Agent 删改测试。
- 小步提交，出错用 git 回滚。

**4.（场景题）一个要跑 3 小时的 Coding 任务，怎么设计成可中断恢复？**
- 初始化阶段生成功能清单（JSON）和进度文件。
- 每轮只做一个功能，测通后才标记通过，再写进度、提交 git。
- 新会话先读进度文件和 git log，跑一遍基础测试再继续。
- 框架层面用 checkpointer 保存图状态，失败从断点续跑。

## 常见误区

1. **「子 Agent 越多越强」**：效果主要来自用了多少 token，而不是 Agent 数量。用错场景只会更贵、更难调试。
2. **「Agent 说做完了就是做完了」**：Agent 经常提前宣布完成。完成与否必须由测试和功能清单判断，不能听 Agent 自己说。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：带子 Agent 的研究 Agent**（取自 `research_agent.py`）

```python
# 定义一个检索工具：普通函数即可，docstring 就是给模型看的说明
def search_docs(query: str) -> str:
    # 说明用途和返回格式：带编号的片段，方便引用
    """检索内部技术文档，返回带编号的片段。"""
    # 返回带 [D编号] 的文档片段
    return DOCS["冷启动"]

# 用 create_deep_agent 组装研究 Agent
agent = create_deep_agent(
    # 传入模型（这里是剧本模型）
    model=Scripted(messages=iter([])),
    # 主 Agent 的角色：拆分问题、派子 Agent、汇总带引用的报告
    system_prompt="你是研究主管：拆分问题、派子 Agent 调研、汇总成带引用的报告。",
    # 注册子 Agent：名字和描述决定主 Agent 什么时候派给它
    subagents=[{"name": "researcher", "description": "负责检索文档并给出带引用的结论",
                # 子 Agent 自己的提示词和工具，只有它能用 search_docs
                "system_prompt": "只用 search_docs 查资料，结论必须带 [D编号]。", "tools": [search_docs]}],
)
# 发起任务：消息列表里放一条用户消息
r = agent.invoke({"messages": [("user", "车机语音助手冷启动为什么慢？出一份报告")]})
# 遍历主 Agent 的消息，打印类型和调用的工具名或内容
for m in r["messages"]:
    print(f"{type(m).__name__:12}", m.tool_calls[0]["name"] if getattr(m, "tool_calls", None) else m.content[:40])
# 报告存在 Agent 的文件系统里，从 files 中取出
print("文件:", r["files"]["/report.md"]["content"])
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `subagents` 是**字典列表**，必填键是 `name`、`description`、`system_prompt`；`tools` 可选，写了就替换掉主 Agent 的工具。
- 主 Agent 委派时调用的是内置工具 `task`，参数名是 `subagent_type` 和 `description`，不是 `name`。
- 文件内容在 `r["files"]["/report.md"]["content"]`，路径以 `/` 开头。

**昨日复习（只做第③步）**：Day 29 的「带人工审批的客服 Agent」。只看你自己写的注释重写，重点核对：`HumanInTheLoopMiddleware(interrupt_on=...)`、`checkpointer=InMemorySaver()`、恢复时的 `Command(resume={"decisions": [{"type": "approve"}]})`。

## 动手练习（30 分钟）

在 `research_agent.py` 上改：

1. 再加一个子 Agent `coder`，工具是 `run_tests() -> str`，返回「3 passed」。在剧本模型里：拿到调研结论后，先派 `coder`，再写报告。
2. 给 `researcher` 加 `"response_format"`（一个含 `conclusion`、`citations` 两个字段的 Pydantic 模型），观察主 Agent 拿到的 `ToolMessage` 变成什么。
3. 打印 `len(r["messages"])`，对比把 `search_docs` 直接给主 Agent 时的消息数，体会上下文隔离。

## 读后自测

1. 深度研究 Agent 为什么用「主 Agent + 子 Agent」？代价是什么？→ 见第 7 节核心概念 1、2，第 8 节面试题 2
2. Coding Agent 怎么知道自己改对了？→ 见第 7 节核心概念 3，第 8 节面试题 3
3. 长任务挂了怎么接着跑？→ 见第 7 节核心概念 4，第 8 节面试题 4

---

## AI 历史知识点：Gemini Deep Research（2024，Google）

**是什么**：Gemini 应用里的一个「深度研究」功能。用户提一个问题，它先起草一份多步研究计划，用户可以修改或批准；批准后它自己上网搜索、阅读，根据读到的内容再发起新的搜索，几分钟后交出一份带原始链接的报告，可导出到 Google Docs。

**何时由谁提出**：2024 年 12 月 11 日，Google 在 Gemini 产品博客上发布，作者是 Gemini 应用产品管理高级总监 Dave Citron。首发时面向 Gemini Advanced 订阅用户，基于 Gemini 1.5 Pro 和 100 万 token 上下文；同一篇文章还发布了 Gemini 2.0 Flash 实验版。

**为什么重要**：它是最早面向大众的「研究型 Agent」产品之一，把「先计划、让用户确认、再多轮搜索、最后带引用出报告」做成了一个按钮，此后各家都推出了类似的 Deep Research 功能。

**和今天的趋势有什么联系**：今天讲的深度研究 Agent 设计，「计划可确认、边搜边调整、报告带出处、产物写进文档」这几个要点，都能在这款产品上看到。

原始来源：[Gemini: Try Deep Research and Gemini 2.0 Flash Experimental（Google 官方博客）](https://blog.google/products-and-platforms/products/gemini/google-gemini-deep-research/)

---

## 参考区（用时查，不必通读）

### 细节速查

**Deep Agents（deepagents 0.7.x）**
- 安装：`pip install -U deepagents`（官方 README 用 `uv add deepagents`）
- `create_deep_agent(model, tools, system_prompt, middleware, subagents, skills, memory, permissions, backend, interrupt_on, response_format, checkpointer, store, ...)`
- `model` 用 `"provider:model"` 字符串或聊天模型对象
- 内置文件工具：`ls`、`read_file`、`write_file`、`edit_file`、`delete`（≥0.7）、`glob`、`grep`；`execute` 仅沙箱后端可用
- `write_todos`：从 0.7 起默认不带，需加 `TodoListMiddleware()`
- 委派工具 `task(description, subagent_type)`；同一轮可发多个 `task` 并行
- 子 Agent 字典：必填 `name`、`description`、`system_prompt`；可选 `mode`（`"isolated"` 默认 / `"fork"` 继承父对话）、`tools`（替换父工具）、`model`、`middleware`、`interrupt_on`、`skills`、`response_format`、`permissions`
- `CompiledSubAgent`：`name`、`description`、`runnable`（已编译的 LangGraph 图）
- 子 Agent 返回：默认是最后一条消息文本；设了 `response_format` 返回 JSON

**Anthropic 多 Agent 研究系统数据**

| 项目 | 数值 |
|---|---|
| 多 Agent（Opus 4 主 + Sonnet 4 子）vs 单 Opus 4 | 内部评测高 90.2% |
| BrowseComp 效果差异的解释因素 | token 用量 80%；加上工具调用次数、模型选择共 95% |
| token 消耗 | Agent 约 4× 聊天；多 Agent 约 15× 聊天 |
| 投入规模 | 简单 1 Agent / 3–10 次调用；对比 2–4 子 Agent / 各 10–15 次；复杂 10+ 子 Agent |
| 并行 | 主 Agent 同时起 3–5 个子 Agent，复杂问题耗时最多降 90% |
| 评测起步 | 约 20 条真实问题；LLM 评分 0.0–1.0 + 通过/不通过 |
| 上线 | 断点恢复 + 重试；彩虹部署（新旧版本并行、逐步切流量） |

**长任务 harness（Anthropic）**
- 初始化 Agent：`init.sh`、进度文件、功能清单 JSON（全部未通过）、首次 git 提交
- 编码 Agent：每轮一个功能；只能改通过状态，「不允许删除或修改测试」；结束时代码可合入
- 失败模式：一次做完整个应用、留下 bug 不记录、过早宣布完成、未验证就标记通过

### 来源说明

- 代码用剧本模型代替真实 LLM，只演示委派与文件写入流程；已在本地实测。
- 「今日 AI 动态」与「AI 历史知识点」来源为 Google 官方博客（非 LangChain/Claude 来源）。OpenAI 2026-10-26 的几篇发布不在本期时间范围内，未收录。

### 延伸学习

- Claude Academy · Introduction to Subagents
- Docs by LangChain · Deep Agents（Overview、Subagents）
- GitHub：langchain-ai/deepagents 的 `examples/deep_research`

### 参考资料

- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Effective harnesses for long-running agents - Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Claude Code: Best practices for agentic coding - Anthropic](https://www.anthropic.com/research/claude-code-best-practices)
- [Deep Agents overview - Docs by LangChain](https://docs.langchain.com/oss/python/deepagents/overview)
- [Subagents - Docs by LangChain](https://docs.langchain.com/oss/python/deepagents/subagents)
- [langchain-ai/deepagents - GitHub](https://github.com/langchain-ai/deepagents)
