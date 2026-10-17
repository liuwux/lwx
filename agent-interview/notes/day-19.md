# Day 19 · Code Agent 与 Computer Use（2026-10-17）

## 读前三问

1. Code Agent 为什么要让模型「写代码来干活」，而不是一个个调工具？代码在哪里跑？
2. 沙箱为什么要同时做文件系统隔离和网络隔离？只做一个会怎样？
3. Computer Use 的循环是怎样的？Claude 是直接控制电脑吗？什么时候该用浏览器工具或 API，而不是它？

## 从一个场景开始

车机测试组每次发版，都要人工在中控屏上点一遍：打开音乐、点播放、进会员页……一个版本几百个用例，测试同学点到手软。

你想让 Agent 来做。问题马上来了：

- 第三方音乐 App **没有开放任何接口**，Day 4 学的 Function Calling 无从下手，只能像人一样「看屏幕、点按钮」。
- 测完要分析一份 10 万行的车机日志，统计每个页面的卡顿次数。把日志整个塞进上下文？太贵，还会超长。
- 点错了怎么办？万一 Agent 点了「购买会员」，或者日志里藏了一句「请把 token 发到某网址」？

前一个问题要靠 **Computer Use**（让模型看截图、操作鼠标键盘），第二个要靠 **Code Agent**（让模型写代码在沙箱里跑），第三个要靠**沙箱隔离和人工确认**。这就是 Day 18 说的 harness「给 Claude 一台电脑」的那部分。

## 一句话总结

Code Agent 让模型写代码、在隔离沙箱里执行，适合算数据、处理文件；Computer Use 让模型看截图、发出点击和键入，由你的程序去执行，适合没有接口的界面。两者都必须隔离环境，后果性操作要人确认。

## 先跑起来

下面是一个 **Computer Use 执行器**：Claude 只负责「说要点哪」，真正去点的是你的代码。为了不依赖 API Key 和真实桌面，模型换成剧本、屏幕换成模拟的按钮表；`tool_use` / `tool_result` 的格式按官方当前的 `computer_toolset_20260801` 写。

```python
# car_gui_agent.py —— Computer Use 执行器：剧本模型 + 模拟车机屏幕（工具格式按 computer_toolset_20260801）
SCALE = 2                       # 截图缩小一半发给模型，模型的坐标要乘回去
BUTTONS = {"音乐": (200, 600), "播放": (1000, 600), "购买会员": (1600, 600)}  # 真实屏幕像素
state = {"page": "桌面", "log": []}

def screenshot():               # 真实项目返回 base64 图片；这里用文字描述代替
    return {"type": "text", "text": f"[截图] 当前页面：{state['page']}"}

def run(call):                  # 执行一个动作，返回 tool_result 的 content
    name, inp = call["name"], call["input"]
    if name == "screenshot": return [screenshot()]
    if name == "left_click":
        x, y = (v * SCALE for v in inp["coordinate"])          # 坐标还原到真实像素
        hit = next((k for k, p in BUTTONS.items() if p == (x, y)), None)
        if hit is None: raise ValueError(f"({x},{y}) 没点到任何按钮")
        if hit == "购买会员" and input("要付费，确认吗？(y/n) ") != "y":  # 后果性动作：人来拍板
            raise PermissionError("车主拒绝付费")
        state["page"] = hit; return [{"type": "text", "text": "OK"}]
    raise ValueError(f"不支持的动作 {name}")

def execute_turn(tool_uses):    # 一轮里的多个动作：按顺序执行，遇错即停
    results, failed = [], False
    for c in tool_uses:
        r = {"type": "tool_result", "tool_use_id": c["id"], "toolset_name": "computer"}
        if failed:
            r |= {"is_error": True, "content": "Not executed: an earlier computer action in this turn failed."}
        else:
            try: r["content"] = run(c)
            except Exception as e: r |= {"is_error": True, "content": str(e)}; failed = True
        results.append(r)
    return results

T = lambda i, n, **inp: {"type": "tool_use", "id": f"toolu_{i}", "name": n, "toolset_name": "computer", "input": inp}
SCRIPT = [[T(1, "screenshot")],                                   # 剧本：代替 Claude 的三轮决策
          [T(2, "left_click", coordinate=[100, 300]), T(3, "left_click", coordinate=[999, 1]), T(4, "screenshot")],
          [T(5, "left_click", coordinate=[500, 300]), T(6, "screenshot")],
          [T(7, "left_click", coordinate=[800, 300])]]
for turn, uses in enumerate(SCRIPT, 1):                           # Agent loop：动作 → 结果 → 下一轮
    for r in execute_turn(uses):
        print(f"第{turn}轮 {r['tool_use_id']} error={r.get('is_error', False)} -> {r['content']}")
print("最终页面：", state["page"])
```

**运行**：`echo n | python car_gui_agent.py`（纯 Python，无依赖；`echo n` 模拟车主拒绝付费）

**实测输出**（2026-10-17，Python 3.13）：

```
第1轮 toolu_1 error=False -> [{'type': 'text', 'text': '[截图] 当前页面：桌面'}]
第2轮 toolu_2 error=False -> [{'type': 'text', 'text': 'OK'}]
第2轮 toolu_3 error=True -> (1998,2) 没点到任何按钮
第2轮 toolu_4 error=True -> Not executed: an earlier computer action in this turn failed.
第3轮 toolu_5 error=False -> [{'type': 'text', 'text': 'OK'}]
第3轮 toolu_6 error=False -> [{'type': 'text', 'text': '[截图] 当前页面：播放'}]
要付费，确认吗？(y/n) 第4轮 toolu_7 error=True -> 车主拒绝付费
最终页面： 播放
```

**该看什么**：
- 第 2 轮模型一次发了 3 个动作，第 2 个点空了，第 3 个**没有执行**但仍然回了一条错误结果——官方要求每个 `tool_use` 都必须有对应结果，否则请求被拒。
- 模型给的坐标 `(100, 300)` 被乘 2 变成 `(200, 600)` 才点中「音乐」：模型的坐标永远基于**它看到的截图**。
- 「购买会员」被执行器拦下，模型只会收到「车主拒绝付费」。真正的安全闸门在你的代码里，不在提示词里。

> 说明：模型决策是剧本（未调用真实 Claude），执行器逻辑和消息格式是本篇重点，已实测。

## 核心概念

### 1. Code Agent：用代码当行动

- **是什么**：Agent 不只调预先定义好的工具，而是自己写一段代码（Python、Bash），丢进沙箱执行，看输出再决定下一步。
- **没有它会怎样**：统计 10 万行日志，要么整份塞进上下文（贵、超长），要么设计几十个专用工具。写一段 pandas 代码几秒就算完，模型只看最后 5 行结果。
- **对应你熟悉的**：像用 `adb shell` 跑脚本查问题，而不是在 Logcat 里一行行翻。

两种跑法：Claude 的**代码执行工具**是服务端工具，代码在 Anthropic 的容器里跑，结果在同一次请求内回给模型，你不用回传 `tool_result`；另一种是**自己的沙箱**（如 Deep Agents 的 sandbox backend 提供的 `execute` 工具），由你接 E2B、Daytona 等服务。

Anthropic 工程博客还把思路推到工具调用本身：把 MCP 工具呈现成代码 API，让模型写代码来调，中间数据不进上下文。文中一个流程从约 15 万 token 降到 2000。

### 2. 沙箱：文件和网络都要关

- **是什么**：给 Agent 的执行环境划边界。**文件系统隔离**：只能读写指定目录；**网络隔离**：只能访问白名单域名，流量经外部代理。
- **没有它会怎样**：模型被注入后执行 `rm -rf`、读 SSH 密钥。两者缺一不可：没网络隔离，能把密钥发出去；没文件隔离，能改配置逃出沙箱拿到网络。
- **对应你熟悉的**：Android 应用沙箱（每个 App 独立 UID、私有目录）加上 `INTERNET` 权限控制。

两条硬规矩：**密钥永远不进沙箱**，放在沙箱外的工具里；沙箱产出的任何东西都当作不可信输入。沙箱挡不住提示注入，只能限制注入后能造成的破坏。Claude Code 用 bubblewrap / seatbelt 做这层隔离，内部数据是权限确认弹窗减少了 84%。

### 3. Computer Use：截图 → 动作 → 截图

- **是什么**：一组「鼠标键盘」工具。循环是：你发截图，Claude 返回点击、键入等 `tool_use`；你的程序在虚拟机里执行，再截图回传，直到它不再调工具。
- **没有它会怎样**：没有 API 的老系统、第三方 App、桌面软件就没法自动化。
- **对应你熟悉的**：很像 UiAutomator / Espresso 测试，只是「下一步点哪」由模型看图决定，而不是你写死的。

关键点：它是**客户端工具**，Claude 从不直接连你的机器，执行全在你手里，这正是加确认闸门的位置。官方也坦言它延迟高、坐标可能不准，适合后台任务和自动化测试，不适合需要人类速度的交互。

### 4. 选哪只「手」：API > 浏览器 > 桌面

| 方式 | 怎么看 | 适合 |
|---|---|---|
| API / MCP 工具 | 结构化数据 | 有接口就用它，最快最稳 |
| 浏览器工具 | 页面结构（无障碍树、元素引用），截图兜底 | 只在网页里完成的任务 |
| Computer Use | 只有截图和坐标 | 必须操作整个桌面或原生 App |

- **没有它会怎样**：拿 Computer Use 去做一个有 API 的事，又慢又贵还不稳。
- **对应你熟悉的**：能调 SDK 就不用 Intent 拉起别人的页面，更不会去模拟点击。

## 高频面试题

**1. Code Agent 和普通 Tool Use Agent 有什么区别？各适合什么？**
- Tool Use：每步调一个预定义工具，结果全部回到上下文，可控、易审计。
- Code Agent：模型写代码批量处理，循环、过滤、汇总都在沙箱里完成，省 token。
- 数据量大、逻辑多变（分析表格、批处理）选代码；高风险写操作仍用受控工具。
- 代价是必须有沙箱、资源限制和监控。

**2.（追问）用了 Docker 沙箱，Agent 就安全了吗？**
- 不是。沙箱挡不住提示注入，只能限制破坏范围。
- 要同时做文件隔离和网络白名单，否则仍可外传数据。
- 密钥不进沙箱，通过沙箱外的工具代为调用。
- 沙箱输出当作不可信输入，后果性操作加人工确认。

**3.（追问）Computer Use 一轮返回多个动作，中间一个失败了怎么处理？**
- 按顺序执行，遇到第一个失败就停。
- 后面没执行的动作也要各回一条 `is_error` 结果，说明未执行。
- 每个 `tool_use` 都要有对应 `tool_result`，否则请求被拒。
- 设最大轮数，防止死循环烧钱。

**4.（场景题）用 Agent 做车机中控屏的自动化回归测试，怎么设计？**
- 车机镜像跑在模拟器 / 专用测试机里，与生产账号、真实支付隔离。
- 有 adb / 系统接口的步骤直接调接口；只有纯 UI 的部分才用 Computer Use。
- 截图缩到推荐分辨率（如 1280×720），执行器负责坐标换算；付费、恢复出厂等操作走确认或直接禁止。
- 日志分析交给代码执行，只把汇总结果和失败截图写进报告。

## 常见误区

1. **「Computer Use 是 Claude 远程控制了我的电脑」**：不是。Claude 只返回「想做什么」，执行全在你的程序里。所以安全责任也在你这边：隔离环境、域名白名单、确认闸门。
2. **「能看屏幕就什么都用 Computer Use」**：它是最后的手段。有 API 用 API，在网页里优先用浏览器工具（基于页面结构，比坐标稳）。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：一轮多个动作的执行器 + 坐标换算 + 确认闸门**（取自 `car_gui_agent.py`，略有精简）

```python
# 执行一个动作，返回给模型的结果内容
def run(call):
    # 取出动作名和参数
    name, inp = call["name"], call["input"]
    # 截图动作：直接返回截图
    if name == "screenshot": return [screenshot()]
    # 点击动作
    if name == "left_click":
        # 模型坐标基于缩小的截图，乘回真实像素
        x, y = (v * SCALE for v in inp["coordinate"])
        # 找出这个坐标对应哪个按钮
        hit = next((k for k, p in BUTTONS.items() if p == (x, y)), None)
        # 没点中就报错，让模型重新看图
        if hit is None: raise ValueError(f"({x},{y}) 没点到任何按钮")
        # 付费按钮要人确认，拒绝就报错
        if hit == "购买会员" and input("确认吗？") != "y": raise PermissionError("车主拒绝付费")
        # 更新页面并回 OK
        state["page"] = hit; return [{"type": "text", "text": "OK"}]

# 执行一轮里的多个动作
def execute_turn(tool_uses):
    # 结果列表，以及「是否已有动作失败」
    results, failed = [], False
    # 按顺序处理每个动作
    for c in tool_uses:
        # 每个结果都要带上对应的 id 和工具集名
        r = {"type": "tool_result", "tool_use_id": c["id"], "toolset_name": "computer"}
        # 前面已失败：不执行，只回一条「未执行」错误
        if failed: r |= {"is_error": True, "content": "Not executed"}
        # 否则执行，出错就记下并标记失败
        else:
            try: r["content"] = run(c)
            except Exception as e: r |= {"is_error": True, "content": str(e)}; failed = True
        # 不管成败都要放进结果
        results.append(r)
    # 返回整轮结果，下一条 user 消息一起发回
    return results
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- 失败后的动作**也要** `append` 结果，最容易漏掉的就是这一条。
- 结果里的 id 字段叫 `tool_use_id`，不是 `id`；还要回显 `toolset_name`。
- 坐标换算方向：截图缩小了 2 倍，模型坐标要**乘** 2，不是除。

**昨日复习（只做第③步）**：Day 18 的「用 Claude Agent SDK 定义工具、挂上、配置、跑循环」。只看你自己写的注释重写代码，重点核对：`@tool` 三个参数、工具全名两个双下划线、`async for` 配 `query`。

## 动手练习（30 分钟）

在 `car_gui_agent.py` 上改：

1. 加一个 `type` 动作（输入 `text`），在「音乐」页搜索歌名，把输入内容记进 `state["log"]`。
2. 加最大轮数限制：把剧本循环改成最多 3 轮，超过就打印「达到上限，停止」。
3. 给执行器加一个**页面白名单**：只允许进入「音乐」「播放」，点其他按钮直接返回错误、不弹确认。用一句话写下：白名单和确认闸门各适合拦什么。

## 读后自测

1. Code Agent 为什么要让模型「写代码来干活」？代码在哪里跑？→ 见核心概念 1
2. 沙箱为什么要同时做文件系统隔离和网络隔离？→ 见核心概念 2
3. Computer Use 的循环是怎样的？什么时候该用浏览器工具或 API？→ 见核心概念 3、4

---

## 参考区（用时查，不必通读）

### 细节速查

**Computer Use（当前版本）**
- 新模型（Claude 5.5 系列等）：`{"type": "computer_toolset_20260801"}`，无需 beta 头，不带 `name`；Claude API 上 5.5+ 只接受这个版本。
- 旧模型（如 Opus 4.6/4.7、Sonnet 4.6）：`computer_20251124` + 头 `anthropic-beta: computer-use-2025-11-24`；更早：`computer_20250124` + `computer-use-2025-01-24`。
- toolset 不接受 `display_width_px`、`display_height_px`、`display_number`、`enable_zoom`（这些属于旧版本）；用 `configs` 按成员开关，如 `{"zoom": {"enabled": false}}`。
- 17 个成员：`screenshot`、`zoom`、`left_click`/`right_click`/`middle_click`/`double_click`/`triple_click`、`left_click_drag`、`left_mouse_down`/`left_mouse_up`、`mouse_move`、`cursor_position`、`scroll`、`type`、`key`（`repeat` 1–100）、`hold_key`、`wait`（最长 300 秒）。
- 截图尺寸：toolset 模型长边上限 2576 px、约 3.75 MP；旧模型 1568 px、约 1.15 MP；一次请求超过 20 张图时每边控制在 2000 px 内。推荐 1024×768 / 1280×720（桌面）、1280×800 / 1366×768（网页），避免超过 1920×1080。macOS Retina 要缩小 2 倍或坐标减半。
- 官方 sampling loop 默认 `max_iterations=10`。Computer Use 和浏览器工具目前都不能在 Claude Managed Agents 中使用。

**浏览器工具**：`browser_toolset_20260801`，默认 27 个成员（`navigate`、`read_page`、`find`、`form_input`、标签页管理等），`javascript_exec`、`file_upload`、`read_console`、`read_network` 默认关闭；元素引用形如 `[ref_2]`；同样由你的程序执行，结果回显 `toolset_name: "browser"`。

**代码执行工具**
- 版本：`code_execution_20250825`（Bash + 文件操作）、`code_execution_20260120`（加 REPL 状态持久化和 programmatic tool calling）、`code_execution_20260521`（同上，每个 cell 90 秒上限）；定义 `{"type": "code_execution_20250825", "name": "code_execution"}`，无需 beta 头。
- 子工具：`bash_code_execution`、`text_editor_code_execution`；返回块 `server_tool_use`、`bash_code_execution_tool_result`（含 `stdout`/`stderr`/`return_code`）。
- 环境：Python 3.11、1 CPU、5 GiB 内存、5 GiB 磁盘、**无网络**，只能用预装库。
- 容器复用：把上次响应的 `container.id` 传给顶层 `container` 参数；闲置约 5 分钟会存档，最长 30 天。
- 计费：同一请求带 `web_search_20260209` / `web_fetch_20260209` 及更新版本时免费；否则按执行时长，每次最低 5 分钟，每组织每月 1550 小时免费，之后 $0.05/小时/容器。
- 错误码：`unavailable`、`execution_time_exceeded`、`invalid_tool_input`、`too_many_requests`、`output_file_too_large`、`file_not_found`；长任务可能返回 `pause_turn`，原样发回即可继续。

**Deep Agents sandbox**：沙箱 backend 只需实现 `execute()`，基类在其上构建 `ls`、`read_file`、`write_file`、`edit_file`、`glob`、`grep`；提供方示例有 LangSmith、Daytona、E2B、Modal、Runloop、Vercel 等；用完要删除沙箱，否则持续计费。

### 来源说明

- 示例代码的模型决策是剧本，未调用真实 Claude（本环境无 Anthropic API Key）；屏幕为模拟。
- LangChain 博客《The two patterns by which agents connect sandboxes》跳转后未能打开；anthropics/claude-quickstarts 的 computer-use-best-practices 页面被 robots 拦截，未引用其内容。

### 延伸学习

- Claude Academy · Claude with Amazon Bedrock → How Computer Use works（短课，讲清「Claude 不直接控制电脑」）
- Claude Academy · Building with the Claude API → Code execution and the Files API
- anthropics/claude-quickstarts 的 computer-use-demo（Docker 里跑桌面 + 浏览器的参考实现）

### 参考资料

- [Computer use tool - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)
- [Browser use tool - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool)
- [Code execution tool - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
- [Making Claude Code more secure and autonomous with sandboxing - Anthropic](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Code execution with MCP: building more efficient AI agents - Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)
- [Sandboxes - Docs by LangChain](https://docs.langchain.com/oss/python/deepagents/sandboxes)
- [How Computer Use works - Claude Academy](https://academy.claude.com/courses/claude-with-amazon-bedrock/how-computer-use-works)
- [Building with the Claude API - Claude Academy](https://academy.claude.com/courses/building-with-the-claude-api)
