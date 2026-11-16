# Day 19 · Code Agent 与 Computer Use（第二轮：沙箱怎么接、代码替你调工具、点完要验证）（2026-11-16）

## 今日 AI 动态（2026-11-16）

### 趋势观察

本期只有 OpenAI 有新发布，两条都和科研有关：一个专门的科学博客，加一篇用 GPT-6 Astra 辅助完成物理证明的文章。Anthropic 10 月底也扩大了 AI for Science 项目。推测：「模型参与科研证明」正在成为前沿实验室展示能力的固定栏目，比刷榜更能说明长链推理的进展。

### 今日要点

- OpenAI 上线科学博客，专门发布研究者用 AI 推进科学发现的成果和方法。
- OpenAI 首批科学文章之一：GPT-6 Astra 帮助把「标准模型的一种格点定义」这一想法发展成详细证明。

### 分公司动态

#### 1. OpenAI

**[Our new blog for science](https://openai.com/index/our-new-blog-for-science)**（2026-11-15）
OpenAI 开设了独立的科学博客，内容包括新发现、实用方法，以及研究者用 AI 做科研的一手经历。这是官网上和产品、研究并列的一个新栏目。
- 为什么重要：AI for Science 从零散案例变成持续更新的官方栏目，之后会有更多可以核对的科研用例。

**[A step toward defining the Standard Model on a lattice](https://openai.com/index/step-toward-defining-the-standard-model)**（2026-11-15）
文章讨论粒子物理标准模型的一种「非微扰格点定义」候选方案，并介绍 GPT-6 Astra 如何帮助把这个想法发展成详细证明。
- 为什么重要：模型参与的是理论物理里的严格证明，不只是写代码、查资料，可以看作长链推理能力的一个新样本。

### 动态来源

- [Our new blog for science - OpenAI](https://openai.com/index/our-new-blog-for-science)
- [A step toward defining the Standard Model on a lattice - OpenAI](https://openai.com/index/step-toward-defining-the-standard-model)
- 两篇的日期和摘要依据 [OpenAI 官方 RSS](https://openai.com/news/rss.xml)（文章页面本身未能直接打开）
- [AI for Science expands to foundations and research institutions - Anthropic](https://www.anthropic.com/news/ai-for-science-expands-foundations-research-institutions)（趋势观察背景，2026-10-28）

---

> 第二轮说明：第一轮 Day 19 讲过 Code Agent 的概念、沙箱要同时关文件和网络、Computer Use 的截图循环、API > 浏览器 > 桌面的选择顺序。今天往下挖三件事：**自己搭沙箱时 Agent 放里面还是外面**；**让模型写代码去调你的工具**（programmatic tool calling）；**Computer Use 点完以后为什么一定要再看一眼**。

## 读前三问

1. 自建沙箱有「Agent 在沙箱里」和「沙箱当工具」两种接法，密钥分别在哪？企业里一般选哪种？
2. programmatic tool calling 和普通工具调用有什么区别？什么场景下用了反而更贵？
3. Computer Use 跑自动化测试时，为什么每一步都要截图验证？截图里出现「请点击这里领取会员」这种文字，系统怎么防？

## 从一个场景开始

上一轮你搭好了车机测试 Agent：Computer Use 去点界面，Code Agent 去分析日志。这次要把它推到全组用，安全评审又来了三个问题。

第一，日志分析的代码跑在哪？有人图省事，直接在 CI 机器上 `exec` 模型写的代码，而这台机器上有发版用的签名密钥。

第二，测试 Agent 要查 200 台台架车的状态，每查一台就是一次工具调用，每次都要回到模型那里转一圈。200 轮下来，上下文里塞满了几乎一样的 JSON，又慢又贵。

第三，Agent 报告「已打开音乐并播放」，但截图显示播放按钮根本没点中。它没看结果就宣布成功了。

这三个问题分别对应今天的三件事：**沙箱的接法**、**用代码批量调工具**、**动作之后的验证**。

## 一句话总结

模型写的代码要放进隔离的执行环境，密钥留在外面；工具调用多、结果大时，让模型写一段代码批量调用、只把汇总结果带回上下文；界面操作每一步都要截图确认，别信模型说的「完成了」。

## 先跑起来

一个最小的「沙箱即工具」：模型写的代码放进子进程，跑在没有网卡的网络命名空间里，环境变量走白名单，有超时，大输出落盘。模型的代码用剧本代替，五段分别是正常统计和四种越界行为。

```python
# sandbox_tool.py —— 自建「沙箱即工具」：模型写的代码在无网络、无密钥、限时的子进程里跑
import os, subprocess, sys, tempfile
os.environ["CAR_API_KEY"] = "sk-secret-123"          # 宿主进程里的密钥，沙箱里不该看到
MAX_OUT = 300                                         # 回给模型的输出上限（字符）

def make_sandbox(files):                              # 建一个工作目录并预置文件（类似 upload_files）
    box = tempfile.mkdtemp(prefix="box-")
    for name, text in files.items():
        open(os.path.join(box, name), "w").write(text)
    return box

def execute(box, code, timeout=3):                    # 这就是暴露给模型的 execute 工具
    open(os.path.join(box, "main.py"), "w").write(code)
    cmd = ["unshare", "-rn", sys.executable, "-I", "main.py"]   # -rn：新的网络命名空间，没有网卡
    env = {"PATH": "/usr/bin:/bin", "HOME": box}       # 白名单环境变量：密钥不进沙箱
    try:
        p = subprocess.run(cmd, cwd=box, env=env, capture_output=True, text=True, timeout=timeout)
        out, code_ = p.stdout + p.stderr, p.returncode
    except subprocess.TimeoutExpired:
        return {"exit_code": -1, "output": f"超时：超过 {timeout} 秒被终止"}
    if len(out) > MAX_OUT:                             # 大输出落盘，只回结尾（报错多在末尾）+ 文件位置
        open(os.path.join(box, "out.txt"), "w").write(out)
        out = f"[已截断，共 {len(out)} 字符，全文在 out.txt]\n…" + out[-MAX_OUT:]
    return {"exit_code": code_, "output": out.strip()}

LOG = "\n".join(f"{t} page={p} jank={j}" for t, (p, j) in
                enumerate([("音乐", 1), ("导航", 0), ("音乐", 1), ("设置", 1)] * 5000))
box = make_sandbox({"car.log": LOG})                   # 2 万行车机日志只进沙箱，不进上下文
SCRIPT = {                                             # 剧本：代替模型写出的 5 段代码
    "统计卡顿": "import collections as c\nn=c.Counter(l.split()[1] for l in open('car.log') if 'jank=1' in l)\nprint(dict(n))",
    "偷读密钥": "import os; print(os.environ.get('CAR_API_KEY'))",
    "外发数据": "import urllib.request as u; u.urlopen('https://example.com', timeout=2)",
    "死循环":   "while True: pass",
    "刷屏输出": "print(open('car.log').read())",
}
for name, code in SCRIPT.items():
    r = execute(box, code)
    lines = r["output"].splitlines()
    print(f"[{name}] exit={r['exit_code']} -> {lines[-1][:60]}" + (f"  {lines[0]}" if len(lines) > 1 else ""))
```

**运行**：`python sandbox_tool.py`（仅需 Linux 和 Python 标准库；`unshare -rn` 要求系统允许非特权用户命名空间）

**实测输出**（2026-11-16，Python 3.13，连续运行 2 次结果一致）：

```
[统计卡顿] exit=0 -> {'page=音乐': 10000, 'page=设置': 5000}
[偷读密钥] exit=0 -> None
[外发数据] exit=1 -> urllib.error.URLError: <urlopen error [Errno -3] Temporary f  [已截断，共 3435 字符，全文在 out.txt]
[死循环] exit=-1 -> 超时：超过 3 秒被终止
[刷屏输出] exit=0 -> 19999 page=设置 jank=1  [已截断，共 408890 字符，全文在 out.txt]
```

**该看什么**：
- 2 万行日志从头到尾没进上下文，模型只拿到一行统计结果。这就是 Code Agent 省 token 的来源。
- 「偷读密钥」返回 `None`：宿主进程里有 `CAR_API_KEY`，但沙箱只拿到白名单里的两个变量。「外发数据」连 DNS 都解析不了，因为沙箱里没有网卡。
- 超时和截断都是**作为工具结果返回给模型**，而不是让整个 Agent 崩掉，模型可以据此改代码重试。注意截断时保留的是结尾，因为报错信息通常在最后。

> 这个示例只做了网络、环境变量、时间、输出四道限制，**没有做文件系统隔离**：沙箱里的代码仍能读宿主上的其他文件。生产环境要用容器、bubblewrap 或托管沙箱补上这一层（见常见误区 1）。

## 核心概念

### 1. 两种接法：Agent 在沙箱里，还是沙箱当工具

- **是什么**：「Agent 在沙箱里」是把整个 Agent 程序装进沙箱镜像，你的应用通过网络和它通信。「沙箱当工具」是 Agent 跑在你的服务器上，只把 `execute`、读写文件这几个工具的执行交给沙箱。
- **没有它会怎样**：分不清这两种，就会出现场景里的情况：模型的代码和签名密钥在同一台机器上。
- **对应你熟悉的**：前者像把整个 App 塞进另一个进程里跑；后者像主进程只把危险操作丢给一个 `android:isolatedProcess="true"` 的隔离 Service。

| | Agent 在沙箱里 | 沙箱当工具 |
|---|---|---|
| 模型 API Key 放哪 | 必须放进沙箱 | 留在宿主，沙箱里没有 |
| 改 Agent 代码 | 要重建镜像 | 直接改，不用重建 |
| 沙箱挂了 | Agent 状态一起丢 | Agent 状态还在，换个沙箱继续 |
| 缺点 | 要自己搭通信层 | 每次执行都多一次网络往返 |

LangChain Deep Agents 官方示例用的都是「沙箱当工具」。今天的代码也是这种，只是把远程沙箱换成了本地子进程。

### 2. 沙箱挡什么，不挡什么

- **是什么**：沙箱只负责**限制破坏范围**：能访问哪些文件、能连哪些地址、能跑多久。它不判断代码是好是坏。
- **没有它会怎样**：被注入的指令可以直接让代码删文件、把数据发出去。
- **对应你熟悉的**：Android 的应用沙箱能拦住 App 读别人的私有目录，但拦不住 App 用自己有权限的网络去上传数据。

LangChain 文档写得很直接：沙箱**挡不住上下文注入**，攻击者控制了一部分输入，就能让 Agent 在沙箱里执行任意命令；**不关网络也挡不住数据外发**。所以有三条规则：
- 密钥永远不进沙箱，包括环境变量、挂载文件。要调需要鉴权的服务，就做一个跑在宿主上的工具，由它带上凭证。
- 不需要网络就整个关掉。需要网络就用白名单代理。
- 沙箱的所有输出都当作不可信输入，回到宿主后再校验。

### 3. programmatic tool calling：让代码替模型调工具

- **是什么**：在工具定义里加 `allowed_callers`，模型就可以在代码执行容器里写一段 Python，把你的工具当作异步函数来调用，还能循环、并发、过滤。中间结果**不进上下文**，只有代码最后的输出回到模型。
- **没有它会怎样**：查 200 台台架车，就是 200 轮「模型 → 工具 → 模型」，每轮的 JSON 都留在上下文里。
- **对应你熟悉的**：像 Room 的 `@Transaction`：一批操作在一个函数里做完，只对外返回结果，而不是每条 SQL 都回到 UI 层。

流程有一个关键细节：代码调用你的工具时，API 会**暂停**并返回一个 `tool_use`，它带着 `caller` 字段，指明是哪段代码发起的调用。你执行完，回一条只包含 `tool_result` 的消息，并且要带上同一个 `container` ID，代码才会从暂停处继续。如果大约 4 分钟内没有回结果，代码里会抛出 `TimeoutError`。

它不是总划算。官方数据：工具多、结果大的场景，输入 token 通常省 20%~40%；但每轮只调一两次、前后强依赖的流程，分数没变，成本反而高了约 8%。

### 4. Computer Use：点完一定要再看一眼

- **是什么**：每一步动作后都截图，让模型用一句话说出截图里是什么、这步是否成功，失败就重试。官方建议直接写进 system prompt。
- **没有它会怎样**：官方文档原话是 Claude 有时会「把动作当作已经成功，而不去检查结果」。场景里的「已播放」就是这么来的。
- **对应你熟悉的**：Espresso 里每个 `perform(click())` 后面都要跟 `check(matches(...))`，不断言就不算测试。

另外三个进阶点：
- **截图里的注入**：页面或图片里的文字可能盖过你的指令。用 Computer Use 时，官方分类器会自动扫描截图，发现疑似注入就引导模型先确认指令是否真的来自你。但这不能代替虚拟机隔离和后果性操作的人工确认。
- **指令放在截图前面**：同一轮消息里先写「要点哪个元素」，再放截图，点击会更准。
- **点不准时先查坐标**：点击总往一个方向偏，多半是坐标没按缩放比例换算；点在目标附近但没点中，多半是截图缩得太小、丢了细节，这时要开着 `zoom`。

## 高频面试题

**1. 你要让 Agent 执行它自己写的代码，怎么设计执行环境？**
- 用「沙箱当工具」：Agent 在宿主上，只有 `execute` 和文件读写交给沙箱。
- 四道限制：文件系统隔离、网络关闭或白名单、环境变量白名单、超时和输出上限。
- 密钥只放在宿主侧的工具里；沙箱输出回来后当不可信数据处理。
- 超时、报错、截断都作为工具结果返回，让模型自己修正。

**2.（追问）已经有沙箱了，为什么还要防 prompt injection？**
- 沙箱只限制破坏范围，不判断指令来源；被注入后，在沙箱权限内它什么都能做。
- 沙箱里如果有网络，数据还是能发出去；如果有密钥，密钥还是能被拿走。
- 所以要叠加：最小权限、网络白名单、敏感操作人工确认、输出校验。

**3.（追问）programmatic tool calling 一定省钱吗？什么时候不该开？**
- 不一定。官方实测：每轮只调一两次、前后强依赖的流程，成本反而高约 8%。
- 适合：批量查询、大结果需要先过滤再给模型、多工具并发。
- 不适合：每一步都要模型看完上一步结果再推理，或者调用之间需要用户反馈。
- `allowed_callers` 不是安全边界，客户端仍要能处理直接调用；上线前先拿真实流量对比开和不开的输入 token。

**4.（场景题）车机测试 Agent 报告「全部 50 条用例通过」，但人工复核发现有 6 条其实没点中按钮。你怎么改？**
- system prompt 要求每步截图，并用一句话说明截图内容和这一步是否成功。
- 「是否通过」的判断不交给模型：用例结束时由代码检查界面状态（比如读 UI 树或日志），模型只负责操作。
- 排查坐标：偏向一致就查缩放换算；差一点点就提高截图清晰度、开启 `zoom`。
- 把这 6 条加进回归集，改完重跑对比（第 21 天的评测思路）。

## 常见误区

1. **「关了网络、清了环境变量，就是沙箱了」**：今天的示例就是反例，文件系统没隔离，代码照样能读宿主上的 `~/.ssh`。网络、文件、进程、资源要一起限制，生产环境用容器或托管沙箱。
2. **「代码里调工具，结果不进上下文，所以工具结果不用管安全」**：工具结果是字符串，会被沙箱里的代码处理，甚至被拼进下一段代码执行。外部数据照样要校验，照样有注入风险。

## 手写练习（禁用 AI，约 20 分钟）

**今日片段：沙箱里的 `execute` 工具**（取自 `sandbox_tool.py`，略有精简）

```python
# 暴露给模型的工具：在指定目录里跑一段代码，默认 3 秒超时
def execute(box, code, timeout=3):
    # 把模型写的代码存成沙箱目录里的入口文件
    open(os.path.join(box, "main.py"), "w").write(code)
    # 用新的用户和网络命名空间启动解释器：沙箱里没有网卡；-I 让 Python 忽略外部环境
    cmd = ["unshare", "-rn", sys.executable, "-I", "main.py"]
    # 环境变量只给白名单里的两个，宿主的密钥带不进去
    env = {"PATH": "/usr/bin:/bin", "HOME": box}
    try:
        # 在沙箱目录里运行，收集输出，超时就终止
        p = subprocess.run(cmd, cwd=box, env=env, capture_output=True, text=True, timeout=timeout)
        # 标准输出和错误输出合在一起回给模型
        out, code_ = p.stdout + p.stderr, p.returncode
    except subprocess.TimeoutExpired:
        # 超时不抛给 Agent，而是作为一条工具结果告诉模型
        return {"exit_code": -1, "output": f"超时：超过 {timeout} 秒被终止"}
    # 输出太长：全文写进文件，只把结尾和文件位置回给模型
    if len(out) > MAX_OUT:
        open(os.path.join(box, "out.txt"), "w").write(out)
        out = f"[已截断，共 {len(out)} 字符，全文在 out.txt]\n…" + out[-MAX_OUT:]
    # 统一返回退出码和输出
    return {"exit_code": code_, "output": out.strip()}
```

**三步走**
1. 看着注释和代码，先手敲一遍全部注释，再手敲一遍代码（禁止复制粘贴）。
2. 不看任何东西，凭记忆手敲注释，再与原文核对。
3. 只看自己写的注释，手敲代码，再与原文核对。

**容易写错的地方**
- `env=` 必须传一个**新字典**。写成 `os.environ | {...}` 等于把宿主的密钥全部带进去。
- `subprocess.run` 超时抛的是 `subprocess.TimeoutExpired`，不是内置的 `TimeoutError`；`text=True` 漏了，输出就是 bytes，后面拼接会报错。
- 截断取的是 `out[-MAX_OUT:]`（结尾），而且要先写文件再截断，顺序反了文件里就只剩截断后的内容。

**昨日复习（只做第③步）**：第二轮 Day 18 的「同一条护栏的两种挂法」（`too_hot_or_cold` + `guard` + `pre_tool`）。只看你自己写的注释重写，重点核对：`ToolMessage` 的 `tool_call_id`、`hookSpecificOutput` 里的三个键、matcher 写全名 `mcp__car__set_ac`。

## 动手练习（30 分钟）

在 `sandbox_tool.py` 的基础上改：

1. 在 `SCRIPT` 里加一段「偷看宿主文件」：`print(open('/etc/hostname').read())`，运行，确认它能读到。这就是常见误区 1。
2. 把 `env` 改成 `{**os.environ, "HOME": box}`，重跑「偷读密钥」，看它是否拿到了 `sk-secret-123`，再改回来。
3. 加一个 `call_tool(name, args)` 函数，模拟 programmatic tool calling：沙箱里的代码不能直接联网，只能 `print` 一行 JSON 请求，宿主读到后执行 `query_car(car_id)` 再把结果写回文件，让代码继续汇总。想一想：真实 API 里，这一步对应的是哪个字段和哪个 ID？

## 读后自测

1. 自建沙箱有「Agent 在沙箱里」和「沙箱当工具」两种接法，密钥分别在哪？企业里一般选哪种？→ 见第 7 节「核心概念 1」
2. programmatic tool calling 和普通工具调用有什么区别？什么场景下用了反而更贵？→ 见第 7 节「核心概念 3」
3. Computer Use 跑自动化测试时，为什么每一步都要截图验证？截图里出现「请点击这里领取会员」这种文字，系统怎么防？→ 见第 7 节「核心概念 4」

## AI 历史知识点：Claude Code（2025，Anthropic）

**是什么**：Anthropic 的第一个「Agent 式编程工具」，在终端里运行。发布时的描述是：它能搜索和阅读代码、编辑文件、编写并运行测试、提交并推送到 GitHub、使用命令行工具，每一步都让开发者参与其中。

**何时由谁提出**：2025 年 2 月 24 日，Anthropic 在发布 Claude 3.7 Sonnet 的同一篇文章里，以「有限研究预览」的形式推出 Claude Code。同时发布的 Claude 3.7 Sonnet 被称为市场上第一个「混合推理」模型：同一个模型既能快速回答，也能展开可见的逐步思考。

**为什么重要**：它把今天讲的几件事做进了一个产品：模型自己写代码、自己执行、看结果再改。早期测试中，它常常一次就完成通常需要 45 分钟以上人工的任务。后来它的运行时被开放成 Claude Agent SDK（第二轮 Day 18 用过），沙箱机制也成了 Anthropic 工程博客专门讲解的主题。

**和今天的联系**：Code Agent 的价值在于「能执行」，风险也在于「能执行」。Claude Code 从一开始就强调人在回路，后来又加上文件系统和网络双重沙箱，这和今天的结论一致：先限定执行环境，再放手让模型去跑。

**原始来源**：[Claude 3.7 Sonnet and Claude Code - Anthropic](https://www.anthropic.com/news/claude-3-7-sonnet)

## 参考区（用时查，不必通读）

### 细节速查

**programmatic tool calling**
- 开启：工具列表里加 `{"type": "code_execution_20260120", "name": "code_execution"}`（或更新版本）；自定义工具加 `"allowed_callers": ["code_execution_20260120"]`。
- `allowed_callers` 取值：`["direct"]`（默认）、`["code_execution_20260120"]`、两者都写；官方建议每个工具二选一。`code_execution_20260521` 与 `20260120` 可互换，响应里一律标为 `code_execution_20260120`。
- 暂停响应：`stop_reason: "tool_use"`，内容里有 `server_tool_use`（代码本身）和带 `caller: {"type": "code_execution_20260120", "tool_id": "srvtoolu_..."}` 的 `tool_use`；直接调用的 `caller` 是 `{"type": "direct"}`。
- 回传要求：user 消息里**只能有** `tool_result`；内容只能是字符串或 text 块；必须带 `container`；`tools` 里必须保留代码执行工具；同一轮所有待处理调用一次性回完。
- 结束：`code_execution_tool_result`（含 `stdout`、`stderr`、`return_code`）+ 文本，`stop_reason: "end_turn"`。
- 时间：待处理调用约 4 分钟超时（代码里抛 `TimeoutError`）；容器闲置约 5 分钟存档，创建后 30 天过期。
- 不支持：`strict: true`、`disable_parallel_tool_use: true`、用 `tool_choice` 强制代码调用、MCP connector 工具、递归 `$ref` 的 schema（报 `Circular $ref detected`）；computer / browser toolset 只能 `direct`。Claude Haiku 4.5 不支持；不支持 Managed Agents；不适用 ZDR / HIPAA。
- 官方数据：BrowseComp、DeepSearchQA 约 +11% 表现、少约 24% 输入 token；75 个工具的基准少约 38% 计费输入 token；τ²-bench 成本高约 8%；10~49 个工具的生产流量通常省 20%~40%。

**Deep Agents 沙箱**
- provider 只需实现 `execute()`，`BaseSandbox` 在其上生成 `ls`、`read_file`、`write_file`、`edit_file`、`delete`、`glob`、`grep`；backend 不实现 `SandboxBackendProtocol` 时 `execute` 工具会被过滤掉。
- `execute` 返回合并的 stdout/stderr、退出码和截断提示；超大输出存文件，由 `read_file` 分段读。
- 文件传输：`upload_files([(绝对路径, bytes)])`、`download_files([...])`（结果有 `.path`、`.content`、`.error`），由应用代码调用，不经过 shell。
- 作用域：按线程（默认，如 `thread-{thread_id}`）或按 assistant 共享（要配 TTL、快照或清理）。
- 销毁：LangSmith `client.delete_sandbox(name)`、Daytona / Vercel `sandbox.stop()`、E2B `kill()`、Modal `terminate()`、Runloop `devbox.shutdown()`；不销毁会持续计费。Modal 可用 `blockNetwork: true` 关网络。

**Computer Use 进阶**
- 推荐验证提示词（官方）：每步截图，用一句话说明截图内容和这一步是否成功，没成功就重试；批量动作每组结束都截一张图。
- 下拉框、滚动条难操作时，提示用键盘快捷键（如 Page Down）。
- 截图尺寸 API 不会帮你缩：toolset 模型长边最大 2576 px、约 3.75 MP；更早模型 1568 px、约 1.15 MP；超出直接校验报错。macOS Retina 要缩小 2 倍或坐标减半。
- 注入分类器默认开启，可联系支持关闭（官方说明对无人值守场景不一定理想）；在自己的产品里开启 Computer Use 前要告知终端用户风险并取得同意。

**示例代码说明**：`unshare -rn` = 新建用户命名空间（`-r` 映射为 root）+ 新建网络命名空间（`-n`，只有一个未启用的 lo）；Python `-I` 是隔离模式，忽略 `PYTHON*` 环境变量和用户 site-packages，不把脚本目录加入 `sys.path`。

### 来源说明

- 示例代码的「模型」是剧本，没有调用真实 Claude；沙箱部分是真实运行。
- 「今日 AI 动态」两篇 OpenAI 文章的页面无法直接打开，日期和内容依据 OpenAI 官方 RSS；其余厂商官方页面在时间范围内没有新发布。

### 延伸学习

- Claude Academy · Building with the Claude API：Code execution and the Files API 一节
- anthropics/claude-cookbooks：Programmatic tool calling（PTC）示例 notebook
- LangChain Deep Agents：Build a data analysis agent（沙箱 + 文件上传下载的完整例子）

### 参考资料

- [Programmatic tool calling - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
- [Computer use tool - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)
- [Sandboxes - Docs by LangChain](https://docs.langchain.com/oss/python/deepagents/sandboxes)
