# 12 家 AI 投资初创公司拆解：它们在做什么，各自有什么值得学

> 2026-11-26 · 历元技术笔记 · 约 9 分钟阅读

2026 年，“AI 做投资”的初创公司密集融资。我们把一份行业整理里提到的 12 家公司逐一查了官网、产品文档、YC 页面和公开报道，按它们解决的问题分成三类：**帮人构建交易策略**、**帮机构挖掘交易机会**、**AI 自己做投资**。

先说一个共同点：这 12 家里没有一家公开过经第三方审计的业绩。凡是文中提到的收益数字，都是公司自报，我们无法核实。

## 一、构建交易策略：把“一句话想法”变成可执行的策略

这一类主要面向个人交易者，核心是降低从想法到策略的门槛。

**Catalyst**（2026 年 10 月获 3000 万美元融资，红杉领投）
- 做什么：用户用一句话说出投资目标，AI 代理自己做研究、构建策略、盯事件、跨交易场所执行，品种包括现货代币、永续合约、代币化美股和预测市场。
- 值得学：资金不托管，通过非托管钱包执行，公司碰不到用户的钱；每一步操作都要用户批准，预算、条件、暂停都由用户掌控。
- 未公开：策略生成方式、回测的数据和成本模型、合规资质。

**Composer**（2026 年 6 月被 SoFi 收购）
- 做什么：无代码策略平台，覆盖美股、ETF 和加密货币。策略叫 Symphony，是一棵由条件分支、指标过滤、权重分配组成的“规则积木树”；自然语言会被翻译成这棵可编辑的树，用户确认后才运行。
- 值得学：策略结构透明、能直接改；回测和实盘用同一套调仓逻辑；回测默认计入监管费用和 1 个基点的滑点，并公开说明。
- 短板：只有日频，不能做日内；社区里的现成策略普遍有拟合历史的倾向。

**PandaAI**（重庆，2026 年 8 月前后累计融资数千万元人民币）
- 做什么：多个 AI 代理分工完成数据、因子、策略、回测、风控、执行，主攻 A 股和期货；底座开源，机构可以私有部署。
- 值得学：把 AI 交易分成 L0–L4 五个自主等级，用来定位产品路线；可视化工作流编排代理；因子库开源（panda_factor）。
- 注意：开源仓库里实盘接口和部分大模型功能当时仍标注“规划中/未实现”，宣传与代码有落差。

**Horizon Trade**（以色列，2026 年 7 月 200 万美元 pre-seed）
- 做什么：AI 代理把自然语言生成为策略代码，回测、压力测试后一键部署，宣称接入十几家券商和交易所。
- 值得学：回测和实盘用同一份策略代码，减少两者表现不一致；多券商接入，不锁定用户。
- 未公开：几乎没有技术文档，回测成本和防过拟合做法都没有说明。

**AnyAlpha**（YC 2026 秋季批）
- 做什么：面向预测市场和体育赛事（Kalshi、Polymarket）的交易代理：和 AI 讨论想法、搭成代理、回测后 24 小时盯盘自动下单。
- 备注：领域和传统股票研究距离较远；它自己的报道也提醒“估算的概率可能是错的，回测不等于实盘”。

## 二、挖掘交易机会：让 AI 替研究员干活

这一类面向对冲基金和资管机构，核心是把“出想法 → 写代码 → 回测 → 上线监控”的研究流程自动化。

**KelAI**（纽约，YC，2026 年 7 月约 500 万美元种子轮）
- 做什么：接入基金自己的数据、投资授权和风控参数，AI 代理走完整个量化研究流程；创始人曾任 WorldQuant 组合经理。
- 值得学：自建评测体系，由专家按机构标准给 AI 的研究打分；用“可验证奖励”训练 AI 的研究习惯——方法论漏洞、推理错误、计算错误都会扣分。也就是说，它用强化学习训练的是“怎么做研究”，而不是“怎么交易”。

**GIM**（Grace Investment Machine，2026 年 7 月 2000 万美元 A 轮，弘毅投资、IDG 等参与）
- 做什么：自称“智能体投资实验室”。核心论文 CogAlpha（ACL 2026）：7 层共 21 个代理，由大模型直接写可执行的 Python 因子代码，再像生物进化一样变异、交叉；用 IC、ICIR、RankIC 等指标筛选，只留排名靠前的因子。
- 值得学：因子不再局限于公式搜索，而且要求有经济含义、可以解释；论文对训练期、验证期、测试期做了严格切分。
- 注意：公开验证基本只有日频量价因子的论文，“多市场实盘”没有数据。

**EquiLibre Technologies**（布拉格，2026 年 6–7 月 A 轮，Creandum 领投，估值超过 5 亿美元）
- 做什么：创始人是 DeepMind 出身、做过德州扑克 AI 的团队，把不完全信息博弈和自我对弈的思路用于交易代理；信号交给 Tower Research 执行，自称在标普 500、纳斯达克上实盘。
- 备注：方法完全不公开，业绩为自述，需要大量算力和专业执行伙伴。

## 三、自主投资：AI 自己做投资

这一类都是自营或对冲基金形态，用自己的钱实盘交易，没有一家给个人用户做工具。

**Abundance**（2026 年 4 月 1 亿美元种子轮，Instacart 联合创始人创办）
- 做什么：部署“数千个”代理，覆盖扫信息、提想法、做研究、选股、定仓位、下单。
- 备注：创始人只说“跑赢多个指数”，没有给出任何数字。

**Prodigy Research**（YC）
- 做什么：自训“量化金融基础模型”，做 delta 中性的自营交易。
- 备注：自报约 3 个月实盘 +140%、“从未有过亏损周”，未经审计，资金规模和杠杆都没有披露，可信度存疑。

**Spectre Intelligence**（YC）
- 做什么：按行业分组（半导体、生物科技），组内多个 AI 交易员相互竞争，策略不断新生、不断被淘汰；实盘产生的决策日志和真实盈亏反过来当训练数据。
- 值得学：用“新生与淘汰”的机制对抗策略失效。具体怎么打分、怎么在代理之间分配资金、淘汰门槛是什么，均未公开。

**Standard Signal**（YC）
- 做什么：用市场结果当奖励，训练金融推理模型；模型同时学“什么时候不交易”；市场中性。
- 值得学：持仓上限、回撤上限、敞口约束由基础设施强制执行，模型绕不过去；每个决策从信号到成交都有可实时审计的日志。
- 备注：自称实盘夏普大于 3，未公开审计；创始人一人团队管理外部资金，关键人风险高。

## 四、几条共同规律

1. **自然语言只是入口**：做得好的都会先把想法落成一种可检查的中间形式（规则树、工作流、代码），再由人确认后执行。
2. **研究流程自动化是机构端的主线**：KelAI、GIM 都在让代理自动出想法、写代码、回测、筛选，但都没有公开严格的多重检验校正和样本外规程。
3. **风控与模型分离**：做得最认真的一家（Standard Signal）把风控放在模型之外由系统硬执行；面向个人的产品则把预算、止损、暂停交给用户。
4. **实盘反馈当训练数据**：Spectre、Standard Signal 都把实盘结果回灌给模型，形成“交易产生数据、数据改进模型”的循环。
5. **业绩普遍不透明**：12 家都没有经审计的公开业绩，只有短期自报数字。对使用者来说，能否看到可核对的前向记录，比宣传里的收益率重要得多。

## 主要来源

- [Catalyst](https://catalyst.app/) · [红杉投资 Catalyst](https://sequoiacap.com/article/partnering-with-catalyst-turning-ideas-into-trades)
- [Composer 回测说明](https://help.composer.trade/article/67-backtest-basics) · [Composer 滑点与费用](https://help.composer.trade/article/78-slippage-and-fees)
- [PandaAI panda_factor](https://github.com/PandaAI-Tech/panda_factor) · [36氪报道](https://www.36kr.com/p/3923186227320455)
- [Horizon Trade 融资](https://dealroom.co/news/139499-horizon-trade-raises-2m-pre-seed-to-open-systematic-trading-to-everyone/)
- [AnyAlpha · YC](https://www.ycombinator.com/companies/anyalpha)
- [KelAI · YC](https://www.ycombinator.com/companies/kelai)
- [GIM 融资新闻稿](https://tools.prnewswire.com/en-us/live/20813/release/20260709EN01146) · [CogAlpha 论文](https://arxiv.org/html/2511.18850v1)
- [EquiLibre · tech.eu](https://tech.eu/2026/07/01/ex-deepmind-researchers-land-record-creandum-funding-to-scale-ai-agents-for-nasdaq/)
- [Abundance · Hedgeweek](https://www.hedgeweek.com/instacart-co-founder-launches-ai-driven-hedge-fund/)
- [Prodigy Research · YC](https://www.ycombinator.com/companies/prodigy-research)
- [Spectre Intelligence · YC](https://www.ycombinator.com/companies/spectre-intelligence)
- [Standard Signal · YC](https://www.ycombinator.com/launches/QdC-standard-signal-hedge-fund-where-ai-researches-and-executes-every-trade)

---

<sub>本文为行业与技术观察，信息来自各公司官网与公开报道，截至 2026-11-26；文中收益数字均为公司自报、未经审计。本文不构成任何投资建议，也不代表对任何公司或产品的推荐。文中提及的公司名称及商标归各自所有者所有。</sub>
