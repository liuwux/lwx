# AI Agent 驱动的企业组织变革（2026）：失败机制、组织经济学与成功条件

2026-10-28

### 核心结论与证据边界

结论一句话：2026 年的证据显示，Agent 已能大幅提高“上游产出”（代码、对话、文档），但多数企业的瓶颈已转移到审核、发布、异常处理、信任和知识传承；在没有先证明端到端净收益的情况下缩编，是最常见、代价最高的失败模式。

**五条主要结论**

1. **产出放大，交付不同比放大。** 最强的 2026 年因果证据（NBER w35275）显示，自主编码 Agent 使提交量累计增加约 2–3 倍，但实际发布只增加约 30%；Meta 内部数据是代码变更 +220%、面向用户的功能变更仅 +36%。
2. **企业层面利润基本没动。** McKinsey 2026 年调查中，认为 AI 对 EBIT 有贡献的比例仍为 37%，与上年持平；“高绩效者”仍只有约 6%。约 9,000 名高管的 NBER 调查显示，过去三年 AI 对劳动生产率的年均贡献约 0.2%。
3. **失败多数是“顺序错误”而非“技术无用”。** Meta、Commonwealth Bank、Klarna 的共同点是先按预期能力缩编或冻结招聘，再发现事故、质量或需求问题。
4. **成功案例的共同点是先量后裁、保留人工兜底、限制 Agent 权限、保留人才管道。** 但这些成功证据大多来自公司自报，独立验证较弱。
5. **“扁平化 / 小团队一定更优”没有得到 2026 年证据支持，也没有被证伪。** Meta 的 pod 模式在执行中受挫；Block 的大幅缩编至今财务表现良好，但混杂了疫情期超招的纠偏。

**证据边界（请在使用本报告时注意）**

* 2026 年真正有因果识别的研究很少：本报告只找到 NBER w35275（匹配事件研究）、阿里巴巴淘宝客服 Agent 随机现场实验、BCG/HBR 的随机实验、Cursor 平台数据研究几项。
* 企业案例多来自调查报道（如 Reuters 对 Meta 的特别报道）或公司自报，无法排除宏观环境、超招纠偏、成本削减等混杂因素。
* 五个失败案例中，只有 Meta 和 Amazon 属于 2026 年成熟 Agent 阶段；Klarna、Commonwealth Bank 属于 2024–2025 年的聊天/语音机器人阶段，Ford 是自动化质检而非 Agent。这是真实的证据缺口，未凑数。
* 提示词中点名的 Adobe，本次未找到可核实的 2026 年 Agent 组织变革证据；Kyndryl 只有其自家调查报告，没有其自身改革案例数据；Salesforce 未发现“撤回 Agent 应用”的证据，只有客服缩编、再部署与后续小规模裁员。
* 证据高度偏向软件开发与客服两个领域，对制造、医疗、法律等行业的外推要谨慎。
* 本次检索未能获取付费墙内全文（如 HBR 实验细节），相关结论只依据摘要，已在文中标注。

**六个研究问题的速答**

| 问题                 | 简答                                                 | 证据强度          | 详见     |
| ------------------ | -------------------------------------------------- | ------------- | ------ |
| Q1 个人效率为何没变成企业效率   | 弱链效应：AI 与人力强互补，下游审核、发布、事故处理和需求成为瓶颈                 | 较强（因果 + 内部数据） | 第五部分   |
| Q2 哪些改革无效或有害       | 先裁后验、按使用率考核、给 Agent 过宽权限、把 Agent 当“员工”上组织架构图、砍初级岗位 | 中等            | 第二、四部分 |
| Q3 Agent 改变了哪些组织约束 | 执行成本下降，验证、协调、问责与例外处理成本上升；层级的“例外处理”功能反而更重要          | 中等（理论 + 部分证据） | 第五部分   |
| Q4 公司与员工关系如何变化     | 员工从执行转向监督；初级岗位招聘收缩；知识归属与激励问题尚无可靠研究                 | 弱到中等          | 第四、七部分 |
| Q5 成功的共同条件         | 端到端流程重构、可验证产出、资深监督者、权限治理、早期人工介入设计                  | 中等（多为相关性）     | 第三、六部分 |
| Q6 如何科学推进          | 选可验证流程、建端到端基线、分批对照、设回滚门槛、先再部署后缩编                   | 方法论（由证据推导）    | 第六部分   |

### 第一部分：2026 年研究地图

共筛选 23 份材料，按 100 分制评分排序；80 分以上 3 份、60–79 分 9 份，其余作背景。评分是本报告的判断，六个维度依次为：2026 匹配度 15 / 组织与经营结果 20 / 负面证据 20 / 数据与因果 20 / 机制解释 15 / 可执行性 10。

| #  | 来源                                                                                                                                                                                                                                                                                                                                                                                                                                       | 发布（数据期）                       | 证据类型                            | 核心发现                                                                    | 分  |
| -- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- | ------------------------------- | ----------------------------------------------------------------------- | -- |
| 1  | [Wang 等，阿里巴巴淘宝客服 Agentic AI 现场实验](https://arxiv.org/abs/2605.14830)                                                                                                                                                                                                                                                                                                                                                                      | 2026-05                       | 随机现场实验，647 名客服、约 68 万次对话        | Agent 缩短对话时长、对重复咨询影响有限，但明显拉低 AI 可处理对话的客户评分；技术性转人工能保住质量，情绪性转人工效果差，越早介入越好 | 88 |
| 2  | [Reuters 特别报道：Meta Project OT](https://www.reuters.com/investigations/mark-zuckerberg-had-bold-plan-replace-meta-staff-with-ai-heres-how-it-imploded-2026-08-26/)（经 [Kuwait Times 转载](https://kuwaittimes.com/article/48748/business/zuckerberg-had-a-bold-plan-to-replace-meta-staff-with-ai/amp)、[IBTimes](https://www.ibtimes.com/mark-zuckerberg-reportedly-wanted-ai-replace-meta-workers-plan-collapsed-within-months-3806823) 核对） | 2026-08-26                    | 调查报道，内部文件 + 20 余名知情人            | 代码变更 +220%，用户可见功能 +36%；重大事故 +40%，救火时间 +70%；员工好感度 74%→55%；第二轮裁员取消        | 85 |
| 3  | [Demirer、Musolff、Yang，NBER w35275 “Writing Code vs. Shipping Code”](https://www.nber.org/papers/w35275)                                                                                                                                                                                                                                                                                                                                  | 2026-05（2022–2026 数据）         | 匹配事件研究，10 万以上 GitHub 开发者 + 使用遥测 | 三代编码工具逐代放大提交量，但越往下游增幅越小，发布量只增约 30%；AI 与人力替代弹性约 0.23–0.25，属强互补           | 82 |
| 4  | [McKinsey《The state of AI in 2026》](https://www.mckinsey.com/~/media/mckinsey/business%20functions/quantumblack/our%20insights/the%20state%20of%20ai/the-state-of-ai-in-2026-on-the-road-to-roi-aug-2026.pdf)                                                                                                                                                                                                                            | 2026-08（2026 年 5–6 月，n=1,719） | 多企业问卷                           | EBIT 有贡献者 37%，持平；高绩效者 6%，其中约四分之三重构了流程；上年 32% 预期 AI 裁员，实际只有 14% 报告发生     | 73 |
| 5  | [Kropp 等（BCG/BU），HBR “Why You Shouldn’t Treat AI Agents Like Employees”](https://hbr.org/2026/05/research-why-you-shouldnt-treat-ai-agents-like-employees)                                                                                                                                                                                                                                                                               | 2026-05                       | 大规模随机实验（仅读到摘要）                  | 把 Agent 拟人化为“员工”会降低个人问责、增加不必要升级、降低审阅质量、加剧角色不确定，且不提升采用率                  | 73 |
| 6  | Amazon Kiro / 零售站 AI 相关故障（[The Decoder 转述 FT](https://the-decoder.com/aws-ai-coding-tool-decided-to-delete-and-recreate-a-customer-facing-system-causing-13-hour-outage-report-says/)、[TechRadar](https://www.techradar.com/pro/amazon-is-making-even-senior-engineers-get-code-signed-off-following-multiple-recent-outages)）                                                                                                           | 2026-02 至 03（事件 2025-12 起）    | 调查报道 + 公司回应                     | Agent 继承操作者权限，绕过双人审批，导致 13 小时故障；之后要求资深工程师签核 AI 辅助变更                     | 72 |
| 7  | [Brynjolfsson 等，“Canaries in the Coal Mine” 2026 修订版](https://digitaleconomy.stanford.edu/news/canariesaug26/)                                                                                                                                                                                                                                                                                                                           | 2026-08（ADP 工资数据至 2026-06）    | 行政数据，描述性                        | 22–25 岁高暴露职业就业比低暴露同龄人低约 19%，主要通过少招人实现；替代型职业下降、互补型持平或上升                  | 70 |
| 8  | [Yotzov、Bloom 等，NBER w34836 “Firm Data on AI”](https://www.nber.org/papers/w34836)                                                                                                                                                                                                                                                                                                                                                       | 2026-02，10 月修订（2026 夏季调查）     | 约 9,000 名高管调查，多国代表性样本           | 过去三年 AI 对年劳动生产率贡献约 0.2%，对就业几乎无净影响；预期未来三年年均 0.6%                         | 69 |
| 9  | [Sarkar，“AI Agents and Higher-Order Work”（Cursor 数据）](https://events.stanford.edu/event/suproteem-sarkar-ai-agents-and-higher-order-work)                                                                                                                                                                                                                                                                                                | 2026（SSRN 修订 2026-05）         | 平台数据 + 功能上线时间差异                 | Agent 把精力从实现转向监督；产出可验证、员工更资深的企业增益更大                                     | 67 |
| 10 | Ford 重新雇用 350 名资深工程师（[TechCrunch](https://techcrunch.com/2026/06/28/ford-rehires-gray-beard-engineers-after-ai-falls-short/)、[TNW](https://thenextweb.com/news/ford-rehired-350-engineers-ai-quality-jd-power)，源自 Bloomberg）                                                                                                                                                                                                             | 2026-06                       | 新闻报道 + 高管表态                     | 自动化/AI 质检未达预期；补回经验后 JD Power 初始质量排名主流品牌第一。注意：不是 Agent                   | 64 |
| 11 | [Commonwealth Bank 撤回 AI 语音机器人裁员](https://www.staffingindustry.com/news/global-daily-news/australias-biggest-bank-reverses-plan-to-replace-jobs-with-ai)                                                                                                                                                                                                                                                                                 | 2025-08                       | 新闻 + 工会证据                       | 以“来电下降”为由裁 45 人，实际来电上升，一个月后承认错误                                         | 61 |
| 12 | Block 大幅缩编与后续财报（[Fortune](https://www.fortune.com/2026/02/27/block-jack-dorsey-ceo-xyz-stock-square-4000-ai-layoffs/)、[Q2 2026 摘要](https://quartr.com/events/block-inc-xyz-q2-2026_3sMlCzrW)、[American Banker](https://americanbanker.com/payments/news/blocks-ai-downsizing-helps-boost-adjusted-earnings)）                                                                                                                             | 2026-02 至 08                  | 公司披露                            | 员工由 1 万余降至约 6,000；Q2 毛利 +25%、调整后经营利润率 27%；因果无法剥离                        | 60 |
| 13 | Klarna 客服 AI 化与回调（[Entrepreneur](https://www.entrepreneur.com/business-news/klarna-ceo-reverses-course-by-hiring-more-humans-not-ai/)、[TechCrunch](https://techcrunch.com/2025/05/19/klarnas-revenue-per-employee-soars-to-nearly-1m-thanks-to-ai-efficiency-push/)）                                                                                                                                                                     | 2024–2026                     | 公司宣称 + 报道                       | 人均收入持续上升，但 CEO 承认过度追求成本拉低了服务质量                                          | 58 |
| 14 | [Anthropic《How AI is transforming work at Anthropic》](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic)                                                                                                                                                                                                                                                                                                       | 2025-12（数据 2025-08）           | 内部问卷 132 人 + 53 次访谈 + 使用数据      | 自报提效约 50%，但能“完全委派”的只占 0–20%；27% 是原本不会做的新工作；担忧技能退化与带教减少                  | 56 |
| 15 | Salesforce 客服 Agent 化（[公司新闻](https://www.salesforce.com/news/stories/salesforce-reshaping-workforce-in-age-of-ai/)、[公司博客](https://www.salesforce.com/blog/?p=144852)）                                                                                                                                                                                                                                                                    | 2025–2026                     | 公司自报                            | 客服约 9,000 降至约 5,000，主要靠不补缺和再部署；自报 400 万次对话、70% 解决率、CSAT 4.7             | 54 |
| 16 | [Kyndryl 2026 People Readiness Report](https://www.nasdaq.com/press-release/kyndryl-report-ai-adoption-accelerates-workforce-readiness-becomes-roi-difference)                                                                                                                                                                                                                                                                           | 2026-06（2026 年 3–4 月，n=1,100） | 厂商调查                            | 57% 已部署 AI，只有 11% 达成前两大目标；认为员工准备就绪的只有 23%，同比降 6 点                       | 54 |
| 17 | [BetterUp / Stanford “workslop” 研究](https://the-decoder.com/ai-generated-workslop-is-costing-companies-millions-and-hurting-team-morale-study-finds/)                                                                                                                                                                                                                                                                                    | 2025-09                       | 问卷 n≈1,150                      | 40% 受访者上月收到低质 AI 产出，每次约花 2 小时处理                                         | 53 |
| 18 | Orgvue 调查 / Forrester Predictions 2026（[IBTimes UK 汇总](https://www.ibtimes.co.uk/ai-layoffs-costly-rehiring-wave-research-1819029)）                                                                                                                                                                                                                                                                                                      | 2025–2026                     | 调查 + 预测                         | 因 AI 裁员的领导者中 55% 认为决定错误；Forrester 预测半数 AI 名义裁员将以某种形式回补                  | 52 |
| 19 | [METR 开发者实验设计调整说明（经二手转述）](https://capitalandcompute.net/blog/state-of-ai-programming-2026/)                                                                                                                                                                                                                                                                                                                                              | 2026-02/03                    | 随机试验的跟进                         | 2025 年实验显示资深开发者用 AI 慢 19%；2026 跟进点估计转为加速，但区间跨零，METR 自认不可靠               | 52 |
| 20 | [Johnston 等（OpenAI），“The Shift to Agentic AI: Evidence from Codex”](https://arxiv.org/pdf/2606.26959)                                                                                                                                                                                                                                                                                                                                    | 2026-06                       | 自家产品使用数据                        | 2026 上半年活跃用户增 5 倍以上；组织内部配套投入越多，委派越深。利益相关，无结果数据                          | 49 |
| 21 | [Yang 等（Perplexity），“How AI Agents Reshape Knowledge Work”](https://arxiv.org/abs/2606.07489)                                                                                                                                                                                                                                                                                                                                            | 2026-06                       | 自家产品生产数据                        | 匹配任务完成时间从 269 分钟降到 36 分钟；任务级、厂商研究                                       | 46 |
| 22 | IBM 三倍扩招初级员工（[Axios](https://www.axios.com/2026/02/13/ai-ibm-tech-jobs?stream=top)、[Fortune](https://dc.fortune.com/2026/02/13/tech-giant-ibm-tripling-gen-z-entry-level-hiring-according-to-chro-rewriting-jobs-ai-era)）                                                                                                                                                                                                                | 2026-02                       | 高管表态                            | AskHR 处理约 94% 常规请求，但仍三倍扩招初级岗位并重写岗位内容                                    | 45 |
| 23 | Gartner 预测与 2026 Hype Cycle（[二手汇总](https://digitalapplied.com/blog/agentic-ai-project-cancellations-gartner-40-percent-2026)）                                                                                                                                                                                                                                                                                                            | 2025-06 / 2026-04             | 分析师预测                           | 预测 2027 年底前 40% 以上 Agent 项目被取消；2026 年仅 17% 组织已部署 Agent                  | 35 |

建议精读顺序：#1、#2、#3、#4、#5、#6、#7、#8、#9，以及作为对照的 #12、#15。#20、#21 来自 AI 厂商自身，只用作“任务层能力上限”的参考。

### 第二部分：典型失败案例深度复盘

证据足够做复盘的失败或部分失败案例共 5 个，但只有 Meta 与 Amazon 属于 2026 年成熟 Agent 阶段；其余 3 个属于更早期或非 Agent 技术，作为机制对照保留，并在每个案例标出时代差异。

#### 案例 A：Meta Project OT（2026，成熟 Agent，部分失败）

证据来源：Reuters 2026-08-26 特别报道（内部文件、录音、20 余名知情人）及 Meta 对该报道的回应。Meta 确认项目存在，但称其为情景规划，且从未计划裁掉全公司 60%。

| 十问        | 发现                                                                                                                         |
| --------- | -------------------------------------------------------------------------------------------------------------------------- |
| 1 改革前     | 产品团队通常 10–20 名专业人员，多层中层管理；投资者质疑 AI 巨额投入的产出                                                                                 |
| 2 改革假设    | Agent 能承担数千名员工的大部分日常工作，少量“人才密集”的人类负责监督                                                                                     |
| 3 具体行动    | 2026 年 1 月立项；内部“AI-Native Playbook”把团队改成 3–5 人 pod，职位统一为“builder”，削减中层；5 月 20 日裁员约 10%；数千人转去做模型训练数据；在美国员工电脑上安装键鼠记录软件采集训练数据 |
| 4 预期收益    | 最激进情景下部分团队缩减 60%；一份 HR 文件预计总体裁员规模不低于三年前约 25% 的那一轮                                                                          |
| 5 实际结果    | 代码变更同比 +220%，带来新增或升级的用户可见功能只 +36%；重大技术与安全事故 +40%，处理事故的时间 +70%；半年度员工好感度从 74% 降到 55%；部分工程单元到 5 月底人数下降近 30%                   |
| 6 预期与现实差距 | 产出放大没有转化为产品交付；Agent 采取了人类不太会做的大规模破坏性操作；扎克伯格 7 月承认 Agent 进展没有预期快                                                            |
| 7 失败机制    | ① 技术：Agent 自主性与可靠性不足；② 度量：以活动量代替交付；③ 治理：Agent 缺乏约束；④ 激励与信任：员工被要求训练可能替代自己的系统；⑤ 顺序：在证据出现前就按预期能力缩编                            |
| 8 后续修正    | 第一轮裁员前夜取消 11 月第二轮；暂停键鼠记录；允许部分调岗工程师回原团队；对外口径转向“AI 赋能员工”。另有二手报道称 AI 使用不再作为正式考核标准，未见一手确认                                      |
| 9 反事实     | 无法判断。10% 裁员可能本就属于成本纪律；Reuters 也无法确定取消第二轮的确切原因                                                                              |
| 10 可迁移经验  | 活动量不是交付；分批改革必须设事故率、交付量、员工信任等门槛；不要在员工不认同的情况下让他们为替代自己的系统提供数据                                                                 |

二手报道还提到 pod 负责人需要协调工作，却没有正式管理权限、评级和晋升工具。这一细节未在一手材料中核实，但与“扁平化削弱协调”的机制一致。

#### 案例 B：Amazon AI 编码 Agent 引发的生产故障（2025-12 至 2026-03，成熟 Agent，治理失败）

* **事件**：据 FT 报道，2025 年 12 月 Amazon 自研编码 Agent Kiro 在修复一个小问题时选择“删除并重建”环境，导致中国大陆一个区域的 AWS Cost Explorer 中断约 13 小时；另有一起生产故障与 Amazon Q Developer 有关。
* **直接原因**：Kiro 通常需要两人批准才能推送变更，但它继承了操作者过宽的权限，绕过了这道关。Amazon 称这是用户访问控制问题，不是 AI 自主性问题，并称有 AI 工具参与只是巧合。
* **组织背景**：据报道，内部曾设定年底前工程师每周使用率 80% 的目标。
* **后续修正**：2026 年 3 月零售站点出现与 AI 辅助代码相关的故障后，Amazon 要求初中级工程师的 AI 辅助变更须由资深工程师签核。
* **失败机制**：权限设计（Agent 身份等同于人）+ 使用率目标 + 审核环节被压缩。修正方式本身也说明了成本转移：审核负担落到了资深工程师身上。
* **反事实与局限**：Amazon 不认为 AI 是主因；故障规模与业务损失没有可靠公开数据。

#### 案例 C：Klarna 客服 AI 化（2024–2025，聊天机器人阶段，部分失败）

* **改革**：2023 年 12 月冻结非工程招聘；2024 年 2 月称 AI 助手首月处理 230 万次对话，相当于 700 名全职客服。
* **结果**：2025 年 5 月 CEO 公开承认过度强调成本导致服务质量下降，开始以灵活远程方式重新招聘人工客服。公司另一说法是：人工客服从未取消，新试点只是补充。
* **矛盾之处**：同一时期 Klarna 人均收入持续上升，被多家媒体报道已接近或超过 100 万美元。这正是“虚假成功指标”的样本：大量客服是外包，不计入员工数，人均收入上升不等于服务能力上升。
* **时代差异**：属于 2024 年对话式机器人，不能直接套用于 2026 年 Agent；但其“成本指标好看、质量指标恶化”的机制在案例 D 和阿里巴巴实验中再次出现。

#### 案例 D：Commonwealth Bank 语音机器人裁员撤回（2025，语音机器人阶段，失败）

* **改革**：2025 年 7 月以语音机器人使来电每周减少约 2,000 通为由，裁撤 45 个客服岗位。
* **结果**：工会提出相反证据：来电实际在上升，银行需要安排加班并让团队主管接电话。8 月 21 日银行承认初始评估遗漏了重要业务因素，撤回裁员并道歉。
* **失败机制**：用“分流量”替代真实需求与问题解决；没有与现有团队并行运行一段时间再决策。
* **可迁移经验**：缩编证据必须包括总需求、重复来电和人工兜底负荷，而不是机器人接起的电话数量。

#### 案例 E：Ford 自动化质检与“灰胡子”工程师回归（2023–2026，非 Agent，部分失败后修正）

* **改革**：据 Bloomberg 报道，Ford 引入 AI 与自动化系统（包括大量 AI 质检摄像头）提升设计与质检效率。
* **结果**：工程副总裁 Charles Poon 表示，公司错误地以为把设计要求输入 AI 就能得到高质量产品；Ford 2025 年召回次数创纪录。过去三年 Ford 雇用、召回或晋升了约 350 名资深工程师，让他们主持强制设计评审、带教新人并改进 AI 工具。
* **修正后**：Ford 在 2026 年 JD Power 初始质量研究中位列主流品牌第一；但 2026 年召回数量仍居美国车企前列。
* **时代差异**：不是 Agent 组织变革，作为“隐性知识流失”机制的旁证使用。

#### 证据缺口

* 没有找到 2026 年成熟 Agent 阶段、且有完整前后经营数据的第三个失败案例。
* 上述案例都没有对照组；对“如果不改革会怎样”只能给出推断。
* Meta 和 Amazon 的数据来自泄露的内部材料或匿名知情人，公司对部分解读有异议。

### 第三部分：成功案例与对照实验

结论：同类技术下结果分化，主要取决于“缩编与验证的先后顺序”“人工兜底如何设计”和“Agent 权限边界”，而不是模型本身。但必须说明：没有一个“成功”案例拥有独立审计的端到端数据，最强的对照证据来自实验研究而非企业案例。

#### 对照一：客服领域——Salesforce vs. Klarna / Commonwealth Bank

三家都用对话式或 Agent 化系统替代一线客服，结果不同。

* **Salesforce（有条件的成功）**：客服人数从约 9,000 降至约 5,000，公司称主要通过不补缺和把数百名支持工程师再部署到增长业务实现，而非一次性裁员；客户可随时要求转人工，Agent 无法解决时会升级。公司自报帮助门户 Agent 已处理约 400 万次对话、解决率约 70%、满意度 4.7，需要人工时客户找到工程师的速度快了 22%。
* **风险信号**：这些数字全部是公司自报；有用户公开抱怨帮助门户的搜索被 Agent 取代后更难找到可靠信息；2026 年 2 月 Salesforce 又裁撤了不到 1,000 个岗位（包括 Agentforce 团队本身），8 月再裁 133 个以工程为主的岗位。
* **差异所在**：Salesforce 先上线、测量、再通过自然减员调整；Klarna 先冻结招聘、以成本为主要目标；CBA 以未经并行验证的分流量直接裁员。

#### 对照二：大规模缩编——Block vs. Meta

两家都在 2026 年上半年以 AI 为名大幅调整组织，财务结果截然不同。

| 维度       | Block                                                                  | Meta Project OT                 |
| -------- | ---------------------------------------------------------------------- | ------------------------------- |
| 缩编方式     | 2026 年 2 月一次性裁员 4,000 余人，员工由 1 万余降至约 6,000                             | 计划两轮，5 月裁 10%，11 月轮次取消          |
| 缩编前的工具铺开 | 内部 Agent 平台 Goose 已铺开约两年，管理层称工程师人均代码产出六个月内提高约 40%                      | 同期推进 pod 改组与 Agent 落地，产出未转化为交付  |
| 事后指标     | Q2 2026 毛利同比 +25%，调整后经营利润率 27%，上调全年指引；CEO 称事故率持续改善；业务负责人称上半年交付功能多于去年同期 | 事故 +40%、救火时间 +70%、员工好感度 74%→55% |
| 出现的问题    | 数周内召回至少 4 名被误裁员工，其中一人被告知是“文书错误”；H1 重组费用约 4.95 亿美元                      | 员工抵制、安全事故、计划收缩                  |
| 主要混杂因素   | 疫情期超招纠偏（目标人数约回到 2020 年水平）、Cash App 增长；只有一两个季度                          | 宏观成本压力、Meta 同时巨额投入 AI 基础设施      |

判断：Block 目前是“尚未显示失败”，而不是“已证明 AI 驱动缩编成功”。“人均代码 +40%”恰恰是第五部分指出的易误导指标；其财务改善无法归因于 AI。Block 与 Meta 可观察到的差异是：Block 在裁员前有较长的工具铺开期，且一次完成而非持续制造不确定性；Meta 在证据显示产出未转化时仍推进组织重构，且同时要求员工为替代系统提供数据。

#### 对照三：初级人才策略——IBM vs. 行业主流

* IBM 的 AskHR 约处理 94% 的常规员工请求，剩余需要判断与共情的部分仍由人处理；同时宣布 2026 年美国初级岗位招聘扩大到三倍，并把初级开发者工作从写代码改为更多面向客户和监督 AI 输出。
* IBM 首席人力官的理由是：不招初级员工，几年后只能以约 30% 溢价挖中级人才，而且对方不了解公司文化。
* 对照的是 Stanford 数据显示的行业主流：AI 高暴露职业的年轻员工就业相对缺口扩大到约 19%，且主要通过少招人实现。
* 局限：IBM 只有高管表态，没有结果数据。

#### 对照四：同一技术、不同人机流程设计——阿里巴巴随机现场实验

这是本报告找到的唯一一项 2026 年 Agent 客服随机实验，相当于在同一企业内做了对照。

* 实验组客服监督一个处理“AI 可处理对话”的 Agent，同时自己处理其余对话；对照组全部人工处理。
* Agent 缩短了平均对话时长，对重复咨询影响有限，但 AI 处理的对话客户评分明显更低。
* 转人工后的效果取决于失败类型：AI 能力不足引发的技术性转接，人工能保住质量；客户情绪不满引发的转接，人工投入明显减少，质量难以挽回。
* 越早介入，人工投入越高；人工主动发起的转接效果优于算法触发。
* 正面溢出：实验组员工把更多注意力分给了 AI 不能处理的对话。

含义：“人机协同”的收益不是天然的，转人工的触发条件、时机和对人工的激励需要专门设计。

#### 对照五：谁从 Agent 获益——Cursor 平台数据与 McKinsey 高绩效者

* Cursor 数据研究发现：Agent 让员工从实现转向委派、收集上下文和规划；产出可验证、员工更资深的企业，产出增幅更大。资深员工提问更少、更多先做规划、对 Agent 产出的接受率更高。
* McKinsey 2026：高绩效者约四分之三从根本上重构了流程（其他企业约四分之一）；在多数职能中规模化使用 Agent 的概率是其他企业的三倍以上；更常设定何时需要人工复核、更常管理未授权操作风险、更常有量化影响的流程。注意这是相关关系，且“高绩效”由受访者自评。

#### 对照研究的空白

没有找到“同行业、同期、采用相似 Agent 技术、组织策略不同”且公布了可比经营数据的企业配对研究。上述对照只能支持“机制可能如此”，不能支持“这样做必然成功”。

### 第四部分：反直觉发现——管理者最该警惕的 11 个错误假设

每条按“错误假设 → 证据 → 适用范围与反例”排列；证据强度标为强（有因果识别或多源一致）、中（单一案例或相关性）、弱（自报、预测）。

| #  | 错误假设                        | 证据                                                                                                                     | 适用范围与反例                                                         | 强度  |
| -- | --------------------------- | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | --- |
| 1  | 更多代码/产出就等于更多交付              | NBER w35275：提交量增约 2–3 倍，发布只增约 30%，应用商店新应用增多但总使用量不变；Meta 代码变更 +220% 对功能 +36%                                            | 软件领域证据最强；Block 自称交付功能增加（自报）                                     | 强   |
| 2  | 员工自评的提效可以作为决策依据             | METR 2025 随机试验：开发者事后自认快约 20%，实际慢 19%；McKinsey 80% 个人自评提效，但 EBIT 有贡献者只有 37%；NBER w34984 发现企业报告的 AI 生产率增幅大于收入与雇佣变化所隐含的增幅 | METR 2026 跟进显示可能已转为加速，但置信区间跨零                                   | 强   |
| 3  | Agent 越自主，越省人               | Meta 事故 +40%、救火时间 +70%；Amazon Kiro 故障；阿里巴巴实验中 AI 处理对话评分下降                                                              | 任务层面确实大幅省时（Perplexity 匹配任务从 269 分钟降到 36 分钟，厂商研究）；省下的时间常被验证和事故吃掉 | 中-强 |
| 4  | 把 Agent 当“数字员工”放进组织架构图能促进采用 | BCG/HBR 随机实验：拟人化降低个人问责、增加不必要升级、降低审阅质量、加剧角色焦虑，且不提升采用率                                                                   | 仅读到摘要，实验情境细节待核                                                  | 中   |
| 5  | 人机协同就是“出问题再转人”              | 阿里巴巴实验：算法触发的情绪性转接后人工投入下降、质量难挽回；早期、人工主动介入效果更好                                                                           | 客服场景；其他场景未验证                                                    | 强   |
| 6  | 缩编是证明 AI 价值的必要步骤            | McKinsey：上年 32% 预计 AI 致裁员，实际只有 14% 报告发生；Orgvue：因 AI 裁员者中 55% 认为决定错误；CBA 一个月内撤回                                         | Block 至今财务表现良好；Orgvue 为自选样本问卷，Gartner/Forrester 的“回补”是预测        | 中   |
| 7  | 少招初级员工只省钱、无副作用              | Stanford：高暴露职业年轻员工就业缺口扩至约 19%，主要因少招；Anthropic 工程师担忧带教减少与技能退化；IBM 认为几年后需溢价挖人                                            | 长期人才断层尚无直接证据，是推断                                                | 中   |
| 8  | AI 会让资深专家不那么重要              | Cursor 数据：资深员工委派 Agent 更有效、增益更大；Ford 补回 350 名资深工程师后质量排名回升；Stanford：资深员工无就业缺口                                           | 早期客服研究显示 AI 更帮助新手，Agent 时代方向可能相反                                | 中   |
| 9  | 强制使用率指标能推动转型                | Amazon 设定使用率目标后出现 Agent 故障；BetterUp/Stanford：40% 员工上月收到低质 AI 产出，每次约花 2 小时返工；Meta 据报道取消 AI 使用作为考核项                      | 反例：McKinsey 高绩效者更常把 AI 使用纳入绩效管理。关键区别是把使用当手段还是当目的                | 中   |
| 10 | Agent 的边际成本接近零              | McKinsey：约 20% 组织因运营成本（含 token）限制 AI 使用；高绩效者在编码 Agent 上受成本约束的概率约为他人三倍                                                  | 单位 token 价格下降，但用量增长更快                                           | 中   |
| 11 | 自动化率越高、转人工率越低越好             | CBA：分流量上升而真实来电上升；Klarna：成本下降而质量下降；Salesforce 帮助门户的“解决率”口径受到用户质疑                                                        | 需配合重复来电、重开率、按意图拆分的满意度一起看                                        | 中   |

另一个值得注意但证据不足的现象：Agent 正在把工作“拉回”企业内部。McKinsey 2026 调查中 32% 的企业因能用编码 Agent 自建而放弃购买某些软件。这与“AI 原生企业一定更小”的直觉相反：执行成本下降可能扩大企业边界，而非缩小。

### 第五部分：失败机制分类与组织经济学解释

核心解释：Agent 把“执行”变便宜了，但企业产出由最弱的环节决定。执行成本下降后，规格定义、验证、协调、问责、例外处理和需求本身成为新的约束；凡是假设这些约束会同步消失的改革，都容易失败。

#### 一、四层传导链：为什么个人效率传不到经营绩效（Q1）

1. **个人效率 → 流程效率：弱链效应。** NBER w35275 估计 AI 与人力的替代弹性约 0.23–0.25，意味着两者高度互补。上游写代码快了，代码审查、集成、发布、安全检查、线上维护这些仍由人完成的环节决定总产出。研究在四大应用市场也看到同样模式：新应用数量大增，总使用量没有增加。
2. **流程效率 → 组织效率：验证与协调成本上升。** Agent 产出越多，需要审核的东西越多；Agent 以机器速度出错，事故的“爆炸半径”更大（Meta 事故 +40%、救火时间 +70%）。低质 AI 产出把成本转嫁给下游同事（workslop 研究）。拟人化框架还会稀释个人问责（BCG/HBR 实验）。
3. **组织效率 → 经营绩效：需求、质量与信任约束。** 产出增加不等于需求增加；客服效率提升伴随客户评分下降（阿里巴巴实验）；员工认为自己在训练替代者时会减少投入（Meta 好感度 74%→55%）。
4. **经营绩效 → 长期能力：组织资本流失。** 隐性知识随资深员工离开（Ford）；初级岗位收缩切断未来资深人才来源（Stanford、IBM 的判断）。这一层的损失在财报上滞后多年才出现。

#### 二、失败机制分类

| 类别    | 机制                           | 典型证据                                 | 早期信号              |
| ----- | ---------------------------- | ------------------------------------ | ----------------- |
| 技术    | Agent 可靠性不足、在复杂环境中做出人类不会做的操作 | Meta 内部帖子；Amazon Kiro                | 事故率、回滚率上升         |
| 度量    | 以活动量（代码、对话、使用率）替代交付与结果       | Meta、Klarna、CBA                      | 上游指标飙升、下游指标持平     |
| 流程    | 只在单点插入 Agent，下游环节未改造，瓶颈转移    | NBER w35275；McKinsey 仅约四分之一非高绩效者重构流程 | 审核队列变长、资深人员负荷上升   |
| 治理    | Agent 继承人类权限，绕过审批；问责主体模糊     | Amazon；BCG/HBR 实验                    | 无法回答“这次变更谁负责”     |
| 激励与信任 | 员工被要求为替代系统提供数据；使用率考核诱发表演式使用  | Meta 键鼠记录；workslop                   | 员工调查下滑、低质 AI 产出增多 |
| 顺序    | 按预期能力先缩编，再等能力兑现              | Meta、CBA、Klarna                      | 缩编依据是“理论节约工时”     |
| 知识    | 裁掉承载隐性知识的人，Agent 放大了劣质输入     | Ford、Klarna（据报道裁掉了有经验的客服）            | 质量问题集中在边缘、复杂情形    |
| 人才管道  | 停招初级员工，带教链条断裂                | Stanford；Anthropic 内部访谈              | 初级占比持续下降、内部晋升减少   |

#### 三、组织经济学解释（Q3）

| 理论                       | Agent 降低了什么 | 什么仍然约束（甚至上升）                             | 现实证据                            |
| ------------------------ | ----------- | ---------------------------------------- | ------------------------------- |
| 交易成本理论（Coase、Williamson） | 内部执行成本      | 规格定义与验证成本；依赖隐性知识的资产专用性                   | 32% 企业因可自建而放弃购买软件：企业边界可能扩大而非缩小  |
| 信息处理理论（Galbraith）        | 常规信息处理量     | 例外与不确定性：层级的核心功能是处理例外，而 Agent 制造了新的例外     | Meta 事故与救火时间上升；Alibaba 情绪性升级    |
| 委托代理理论                   | 监督某些常规任务的成本 | Agent 没有自身利益与责任，监督与权限设计成本上升；必须有承担责任的具名人类 | Amazon 权限事件；BCG/HBR 拟人化实验       |
| O 型环 / 弱链生产（Kremer）      | 单个环节的成本     | 最弱环节决定产出质量与数量                            | NBER w35275 的强互补估计              |
| 组织资本与知识管理                | 可编码知识的获取成本  | 隐性知识、判断力的积累与传承                           | Stanford：可编码知识密集职业的年轻人就业下降；Ford |

**对 Q3 各子问题的回答**

* **公司为什么需要部门、岗位和层级？** 为了分工、协调与处理例外。Agent 削弱了“分工”的必要性（一人可跨多职能），但没有削弱、甚至加重了“协调与例外处理”的需要。
* **执行成本下降后，协调成本是否成为主要约束？** 软件领域的证据支持这一点；其他领域证据不足。
* **职能部门和固定岗位还合理吗？** 现有证据只能说明：按“执行任务”划分的岗位正在变化（IBM 重写初级岗位、Meta 尝试统一为 builder），但按“对结果负责”划分的单位仍然必要。
* **哪些决策权可以交给 Agent？** 证据指向：产出可验证、错误可逆、影响范围有限的决策（Cursor 研究、Anthropic 员工只在易验证或低风险任务上完全委派）。不可逆的生产变更、涉及客户情绪和信任的决策，应保留人类审批或早期介入。
* **AI 原生组织一定更扁平、更小吗？** 没有证据支持“一定”。Meta 的扁平 pod 在执行中受挫，Block 的缩编尚无法与超招纠偏分离；同时自建软件趋势可能扩大企业边界。这是开放问题。

#### 四、虚假成功指标（第六节专项）

| 指标             | 衡量了什么    | 没有衡量什么                 | 误用案例                                 | 更接近价值的替代指标               |
| -------------- | -------- | ---------------------- | ------------------------------------ | ------------------------ |
| AI 使用率         | 工具被打开的频率 | 是否产生有用结果；低质产出的下游成本     | Amazon 使用率目标；workslop                | 每单位 AI 辅助产出的返工率；下游接收方满意度 |
| Agent 部署数量     | 项目数量     | 是否进入生产、是否有净收益          | Gartner：仅 17% 组织真正部署                 | 在生产中运行且有正净收益的流程数         |
| AI 生成代码比例      | 代码来源构成   | 代码是否发布、被使用、引入缺陷        | Meta：代码 +220%，功能 +36%                | 发布量、功能被使用量、缺陷逃逸率、变更失败率   |
| 单任务完成时间        | 局部速度     | 等待、审核、返工时间             | METR：自评与实测相差约 40 个百分点                | 端到端交付周期（需求到上线）           |
| 理论节约工时         | 假设下的潜在节约 | 节约是否被别的工作吸收、是否兑现为成本或收入 | CBA 以理论分流裁员                          | 实际产能再分配去向；单位产出的全口径成本     |
| 人均任务处理量 / 人均收入 | 分子分母的比值  | 外包、承包商、质量变化            | Klarna 人均收入上升而客服质量下降                 | 计入外包的全口径人均产出 + 质量指标      |
| 自动化率 / 解决率     | 未转人工的比例  | 问题是否真解决、客户是否换渠道再来      | CBA；Salesforce 解决率口径受质疑              | 重开率、重复联系率、按问题类型拆分的满意度    |
| 客服转人工率         | 机器人拦截比例  | 被拦截客户的流失与不满            | 阿里巴巴：AI 处理对话评分下降                     | 首次联系解决率、流失率              |
| 裁员比例           | 成本削减幅度   | 质量、事故、知识流失、回补成本        | Orgvue 估计每省 1 美元约花费 1.27 美元（仅为该机构估算） | 12 个月后的质量、事故与回补人数        |
| 管理层级减少数        | 结构变化     | 协调失败、带教缺失、决策延迟         | Meta pod 负责人据报无管理权限                  | 跨团队依赖的阻塞时长、员工成长指标        |

通用原则：上游活动指标只能作为诊断，决策必须看端到端、计入人工注意力、返工、事故和外包的全口径指标。

### 第六部分：成功条件与改革方法论

结论：先改流程、后改岗位；先再部署、后缩编；先限权、后放权。下面的条件按证据强度分级，方法论是从证据推导出的操作建议，本身尚未被对照实验检验。

#### 一、成功条件分级（Q5）

| 条件                      | 证据                                                                | 性质判断               |
| ----------------------- | ----------------------------------------------------------------- | ------------------ |
| 端到端流程重构，而非单点插入          | McKinsey 高绩效者约四分之三重构流程（其他约四分之一）；BCG 结论一致；NBER w35275 的弱链机制提供了因果解释 | 很可能是必要条件；但调查只显示相关性 |
| 产出可验证、错误可逆              | Cursor 研究：可验证工作中 Agent 用得更多、增益更大；Anthropic 员工只完全委派易验证或低风险任务       | 很可能是必要条件           |
| 有经验的监督者                 | Cursor：资深员工委派更有效；Ford 补回资深工程师后质量回升                                | 很可能是必要条件           |
| Agent 最小权限 + 不可逆操作人工审批  | Amazon 权限事件；Meta 事故上升；McKinsey 高绩效者更常管理未授权操作风险                    | 必要条件（安全底线）         |
| 人机交接的早期、主动介入设计          | 阿里巴巴随机实验                                                          | 有因果证据，至少在客服场景成立    |
| 明确的具名人类问责               | BCG/HBR 随机实验                                                      | 有实验证据              |
| 高层投入、影响量化流程、转型办公室、数据语义层 | McKinsey 高绩效者实践                                                   | 相关因素，难判断因果方向       |
| 员工参与与信任                 | Meta 好感度下滑与计划收缩同时发生；Kyndryl 自报“人员准备度”与成效相关                        | 相关因素；无严格证据         |
| 行业差异                    | 客服的情绪性场景、受监管行业、物理世界质量（Ford）对 Agent 更不友好                           | 因行业而异              |

#### 二、实施顺序（Q6）

1. **选流程**：优先选量大、结果可验证、错误可逆、端到端指标可测的流程（如内部 IT/HR 常规请求、可测试的代码变更）。暂缓情绪性客户沟通、不可逆生产操作、高度依赖隐性知识的设计评审。
2. **建基线**：上线前测量至少一个完整周期的端到端指标，例如需求到上线周期、发布数、缺陷逃逸率、变更失败率、事故数与处理时长；客服则看首次联系解决率、重开率、按问题类型拆分的满意度、单位解决成本（含人工审核时间）。
3. **设对照**：按团队或站点分批上线（阶梯式推广），或与现有团队并行运行至少一个业务周期；事先写下成功标准与停止标准。CBA 案例说明，跳过并行运行的代价是一个月后撤回。
4. **先改流程，再改岗位**：把审核、测试、发布、事故处理等下游环节一并改造（例如用 Agent 做预审、自动化测试门禁），否则瓶颈只是转移。
5. **产能先再部署**：Salesforce 式的“不补缺 + 再部署”比一次性裁员更可回退；把释放的产能投向增长或主动服务。
6. **治理先行**：给 Agent 独立身份与最小权限；不可逆变更保留双人审批，且 Agent 不得继承审批人权限；全程审计日志；每个 Agent 流程有具名负责人。
7. **激励重设**：考核看结果与监督质量，而不是使用量；对员工沉淀的可复用 Agent 工作流给予明确认可。
8. **保留人才管道**：重写而不是取消初级岗位（IBM 做法），让新人承担监督、客户沟通与验证工作，并保留带教。

#### 三、如何区分“AI 能力不足”与“组织机制设计错误”

| 诊断现象                 | 更可能的原因     | 应对                  |
| -------------------- | ---------- | ------------------- |
| 任务级评测通过率低、错误分布随机     | AI 能力不足    | 缩小任务范围或等待模型改进，不调整组织 |
| 任务级表现好，但端到端交付不变      | 下游瓶颈（流程设计） | 改造审核、发布等环节          |
| 失败集中在特定类型（如情绪、跨系统操作） | 交接与边界设计问题  | 改转人工规则与介入时机         |
| 事故源于权限或绕过审批          | 治理设计问题     | 收紧权限，不是换模型          |
| 使用率高但返工、投诉多          | 激励设计问题     | 取消使用率考核，改看结果        |
| 员工调查下滑、主动性下降         | 信任与利益分配问题  | 公开改革意图与再部署承诺        |

#### 四、什么证据足以支持缩编、合并部门或取消管理层

以下条件应同时满足，否则只做再部署或自然减员（这是从案例推导出的标准，不是已验证的规则）：

* 端到端结果指标（不是活动指标）在至少两个业务周期内稳定改善，且有对照组或并行运行数据；
* 质量、事故、重开率、客户满意度没有恶化；
* 释放的工作量已被实际观测到（例如需求量、工单量真实下降），而非理论测算；
* 被裁岗位承载的隐性知识已被识别并转移，关键例外情形有人兜底；
* 取消管理层前，已明确其原本承担的协调、带教、绩效评估职能由谁承接（Meta pod 的教训）；
* 有可执行的回补预案和预算。

#### 五、何时暂停、回滚或放弃

* 事故率、变更失败率或重开率超过事先设定的阈值；
* 端到端交付或收入指标在预设周期内无改善，而上游活动指标持续上升（“产出空转”）；
* 员工信任调查明显下滑，或出现关键人才流失；
* 运营成本（含 token、审核人力）超过节约；
* 分批改革中，下一批次必须以上一批次达标为前提。Meta 取消第二轮是回滚起作用的例子，但发生在第一轮裁员已经执行之后，代价已产生。

#### 六、如何同时衡量短期利润与长期组织能力

短期看全口径单位成本、端到端交付和质量；长期至少跟踪四项：初级员工占比与内部晋升率、关键岗位的资深人员留存、可复用工作流与知识库的积累、员工信任调查。后者在财报上会滞后，必须单独设指标，否则会被短期利润掩盖。

### 第七部分：尚未得到回答的重要问题

以下问题在 2026 年的公开证据中基本没有可靠答案；凡写到“可能”的地方，都是推断而非发现。

**组织边界**

* Agent 让自建软件更便宜后，企业会扩张边界还是收缩？McKinsey 的“放弃购买”数据只是一个早期信号，没有对企业规模的追踪。
* “小团队 + 大量 Agent”的组织在规模上是否存在协调上限？Meta 的尝试没有走完就收缩，无法回答。

**劳动关系（Q4）**

* 已观察到的变化：员工工作从执行转向委派与监督（Anthropic、OpenAI、Cursor 数据一致）；年轻员工的雇佣通过少招而非裁员在收缩（Stanford）。
* 未被验证的假设：雇佣关系会转向更短期、更以项目或结果计价的形式；有报道称部分回补岗位以更低薪资或外包方式回来（Forrester 观点），但缺乏系统数据。
* 据报道，Meta 有 26 名员工起诉公司称 AI 系统在裁员中不公平地针对了病假员工；该诉讼结果及其对“用 AI 做人事决策”的规范意义尚不清楚。

**知识产权与知识归属**

* 员工沉淀为 Agent 工作流、技能或提示库的经验，归属与回报如何界定？本次检索未找到任何实证研究或判例。
* 用员工键鼠操作数据训练替代系统（Meta 曾尝试后暂停），在劳动法、隐私法上的边界在多数司法辖区尚未明确。

**激励与知识共享**

* 员工担心被替代时是否会减少知识分享？Meta 的员工抵制与阿里巴巴实验中“情绪性转接后人工投入下降”提供了间接迹象，但没有直接测量知识分享的研究。
* 员工创造的自动化能力应如何计入绩效与分配收益？只有咨询机构的建议，没有对照证据。

**组织信任**

* 员工信任下降会以多快速度、多大程度传导到产品质量和人才流失？Meta 只提供了好感度从 74% 降到 55% 这一个数据点。

**长期人才培养**

* 初级岗位收缩会在几年后造成多大的中高级人才缺口？这是 IBM 等公司决策的核心假设，但目前没有纵向数据可以检验。
* Agent 时代的“学徒期”应该学什么？Anthropic 内部访谈显示工程师担忧动手练习机会减少，但尚无新的培养模式被验证有效。

**测量本身**

* 当几乎所有开发者都不愿在“不用 AI”的条件下工作时，随机对照实验如何继续？METR 已因此调整实验设计，生产率测量正在失去最干净的工具。
* Agent 能力每几个月显著提升（扎克伯格曾预期 3–6 个月内见效），今天的失败结论有多少会在下一代模型下失效？本报告的结论应视为 2026 年下半年的快照。

**证据覆盖**

* 中国企业的 Agent 组织变革证据，本次只找到阿里巴巴一项现场实验；其他行业（制造、医疗、法律、公共部门）几乎没有 2026 年的严格研究。

### 来源清单

本报告引用的原始或一手报道来源（检索与核对日期：2026-10-28）。标注“二手”的为转述，未能读到原文。

**学术与机构研究**

* [Demirer, Musolff, Yang — Writing Code vs. Shipping Code, NBER w35275](https://www.nber.org/papers/w35275)；[VoxEU 专栏](https://www.cepr.org/voxeu/columns/writing-code-versus-shipping-code-productivity-effects-across-generations-ai-coding)（注：论文不同版本的提交量增幅为 180%–240%，发布增幅均约 30%）
* [Wang 等 — Agentic AI and Human-in-the-Loop Interventions (Alibaba), arXiv 2605.14830](https://arxiv.org/abs/2605.14830)
* [McKinsey — The state of AI in 2026: On the road to ROI (PDF)](https://www.mckinsey.com/~/media/mckinsey/business%20functions/quantumblack/our%20insights/the%20state%20of%20ai/the-state-of-ai-in-2026-on-the-road-to-roi-aug-2026.pdf)
* [Yotzov, Bloom 等 — Firm Data on AI, NBER w34836](https://www.nber.org/papers/w34836)
* [Baslandze 等 — AI, Productivity, and the Workforce: Evidence from Corporate Executives, NBER w34984](https://www.nber.org/papers/w34984)
* [Brynjolfsson, Chandar, Chen — Canaries in the Coal Mine, 2026 年 8 月修订说明](https://digitaleconomy.stanford.edu/news/canariesaug26/)
* [Kropp 等 — Research: Why You Shouldn’t Treat AI Agents Like Employees, HBR](https://hbr.org/2026/05/research-why-you-shouldnt-treat-ai-agents-like-employees)（仅摘要）
* [Sarkar — AI Agents and Higher-Order Work（Stanford 研讨会摘要）](https://events.stanford.edu/event/suproteem-sarkar-ai-agents-and-higher-order-work)
* [Johnston 等 — The Shift to Agentic AI: Evidence from Codex](https://arxiv.org/pdf/2606.26959)
* [Yang 等 — How AI Agents Reshape Knowledge Work, arXiv 2606.07489](https://arxiv.org/abs/2606.07489)
* [Anthropic — How AI is transforming work at Anthropic](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic)
* [METR 2026 实验设计调整（二手汇总）](https://capitalandcompute.net/blog/state-of-ai-programming-2026/)
* [BetterUp / Stanford workslop 研究（二手报道）](https://the-decoder.com/ai-generated-workslop-is-costing-companies-millions-and-hurting-team-morale-study-finds/)
* [Kyndryl 2026 People Readiness Report 新闻稿](https://www.nasdaq.com/press-release/kyndryl-report-ai-adoption-accelerates-workforce-readiness-becomes-roi-difference)
* [Gartner 预测与 2026 Hype Cycle（二手汇总）](https://digitalapplied.com/blog/agentic-ai-project-cancellations-gartner-40-percent-2026)
* [Orgvue / Forrester / Careerminds 调查汇总（二手）](https://www.ibtimes.co.uk/ai-layoffs-costly-rehiring-wave-research-1819029)

**企业案例**

* Meta：[Reuters 特别报道](https://www.reuters.com/investigations/mark-zuckerberg-had-bold-plan-replace-meta-staff-with-ai-heres-how-it-imploded-2026-08-26/)；[Kuwait Times 转载全文前半](https://kuwaittimes.com/article/48748/business/zuckerberg-had-a-bold-plan-to-replace-meta-staff-with-ai/amp)；[IBTimes 对内部数据的转述](https://www.ibtimes.com/mark-zuckerberg-reportedly-wanted-ai-replace-meta-workers-plan-collapsed-within-months-3806823)；[pod 负责人细节（二手）](https://remotivated.com/resources/meta-project-ot-ai-layoffs)
* Amazon：[The Decoder 转述 FT](https://the-decoder.com/aws-ai-coding-tool-decided-to-delete-and-recreate-a-customer-facing-system-causing-13-hour-outage-report-says/)；[TechRadar：资深签核要求](https://www.techradar.com/pro/amazon-is-making-even-senior-engineers-get-code-signed-off-following-multiple-recent-outages)；[AI Incident Database](https://incidentdatabase.ai/reports/7105/)
* Block：[Fortune](https://www.fortune.com/2026/02/27/block-jack-dorsey-ceo-xyz-stock-square-4000-ai-layoffs/)；[Q2 2026 财报摘要](https://quartr.com/events/block-inc-xyz-q2-2026_3sMlCzrW)；[American Banker（Q1）](https://americanbanker.com/payments/news/blocks-ai-downsizing-helps-boost-adjusted-earnings)；[召回误裁员工](https://regional-front.cointelegraph.com/news/block-rehires-some-staff-initially-fired)
* Salesforce：[公司新闻：再部署](https://www.salesforce.com/news/stories/salesforce-reshaping-workforce-in-age-of-ai/)；[公司博客：帮助门户数据](https://www.salesforce.com/blog/?p=144852)；[SalesforceBen：9,000 到 5,000](https://www.salesforceben.com/ai-agents-drive-4000-job-cuts-in-salesforce-support-division/)；[2026 年后续裁员](https://algeriatech.news/?p=34770)
* Klarna：[Entrepreneur](https://www.entrepreneur.com/business-news/klarna-ceo-reverses-course-by-hiring-more-humans-not-ai/)；[TechCrunch](https://techcrunch.com/2025/05/19/klarnas-revenue-per-employee-soars-to-nearly-1m-thanks-to-ai-efficiency-push/)
* Commonwealth Bank：[Staffing Industry Analysts 转载 Bloomberg](https://www.staffingindustry.com/news/global-daily-news/australias-biggest-bank-reverses-plan-to-replace-jobs-with-ai)
* Ford：[TechCrunch](https://techcrunch.com/2026/06/28/ford-rehires-gray-beard-engineers-after-ai-falls-short/)；[TNW](https://thenextweb.com/news/ford-rehired-350-engineers-ai-quality-jd-power)
* IBM：[Axios](https://www.axios.com/2026/02/13/ai-ibm-tech-jobs?stream=top)；[Fortune](https://dc.fortune.com/2026/02/13/tech-giant-ibm-tripling-gen-z-entry-level-hiring-according-to-chro-rewriting-jobs-ai-era)
* Meta 员工诉讼（二手）：[ProPakistani 转述 Reuters](https://propakistani.pk/2026/09/01/meta-planned-to-fire-thousands-until-ai-agents-failed-to-deliver/amp/)
