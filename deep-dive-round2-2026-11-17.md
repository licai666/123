# 破茧情报站 · 第二轮深挖清单（2026-11-17）

六路（G–L），全部亲开原站核实。规矩同第一轮：宁缺毋滥、查不到写未核实、不收盘点文。本轮入站 22 条（站内 ids 490–511），另有 7 条与站内已有条目重复（GEX 三个实现、Cboe 数据接口、SAVNG 复现、optionsdecay、OptionOmega 的 MEIC 访谈），其新深度并入本清单，不重复入站。

## G 路 · GEX 实现到底能不能跑

1. **Gex-Multi（itsfabtrading/gex-multi）——先跑它。** 两步装好：pip install -r requirements.txt 后一条命令出活，输出文本表＋JSON＋SQLite 历史＋PNG；含 flip、0DTE flip、双墙、magnet、1 日预期波幅带。引擎代码原样内嵌自 GMestreM/gex_data（2024 年建、7 forks，三套里被真实世界碰得最多）。坑：issues 0 条＝零社区验证；作者自曝到期日当天盘前链还是昨收、现价是昨收价；作者明说这些位「不是已验证的 edge」。第一步：钉死 HEAD（de5a273）只跑 SPX，对一眼夜航社现有 GEX 标签语义；SQLite 历史表可作状态变量落盘格式参考。
2. **GEX Dashboard（Darthreign/gex-dashboard）——第二阶段对照。** 281 commits、有 tests 和 CI、自带 MCP server（get_gex_by_strike 等工具，AI 可直接问数）；可选 tastytrade dxFeed 去延迟（作者实测同刻 0DTE 成交量是 Cboe 的 3–6 倍、OI 逐合约一致）。坑：仓库自带 .mcp.json 是 Windows 相对路径，Mac/Linux 不改绝对路径 MCP 会静默失败；它是带调度器＋Parquet 的常驻应用，运维重量大；文档主体法语。
3. **Traders Edge MCP——只拆零件。** zero_dte_exposure 一次出齐 0DTE 的 GEX/flip/双墙/max-pain/预期波幅，snapshot_log 把日内状态落 SQLite——这两份字段清单可作夜航社 GEX 状态包模板。它的提交史是延迟链踩坑录：B23「CDN 滚动前 prev_day_close 差一个交易日」、B21「延迟链里的 spot 从未被校验」——自算时现价必须独立校验，不能信链里带的。本体绑 Robinhood 登录态，不装。
4. **Cboe _SPX.json 字段逐字核实。** 每张合约字段：option、bid/ask（及 size）、iv、open_interest、volume、delta、gamma、vega、theta、rho、theo、prev_day_close 等。亲见坑：远离平值合约的 iv 是垃圾值（400 put 的 iv＝8.111、600 的 6.96，gamma 全 0），接口自带 gamma 不进计算；只取 open_interest 和平值附近的 iv，gamma 用 BSM 重算。
5. **存疑一条**：pikki622/gex-analytics 的 README 说 Cboe 免费端点只供指数链、ETF 链返回 403 要 DataShop 订阅，与 gex-multi、gex-dashboard「SPY/QQQ 可直接跑」的说法矛盾，三方打架无法只靠文档裁定；夜航社只做 SPX 不受影响，记入待核实。

**总判断**：先跑 Gex-Multi（一人命令、吐 JSON＋SQLite、常量集中可冻结版本）；首跑后拿 flip/双墙与 gex-dashboard 或 GexVision 连对几个交易日，一致了才许进判断链。

## H 路 · 止损在 IBKR 上怎么落地

1. **IBKR 官方：组合止损是 IB 模拟触发。** 官方 Trigger Methods 页有独立 COMBO 行：组合止损单由 IB 服务器模拟监控，默认按组合 bid/ask 触发；官方警告 triggerMethod 与 secType 不兼容时「订单可能永不触发」。Order Types 页明说模拟单可能因行情错误/缺失、数据过滤（IB 忽略偏离 bid-ask 的 last sale）、停牌而不成交或误成交。未核实：组合 bid/ask 如何由各腿合成，官方没写明；也没有找到 SPX 指数期权专属订单限制的官方页。
2. **IBKR 官方：组合定价与 bracket 硬限制。** 超过 2 腿的组合只能整单一个净价、且必须 Guaranteed（不能 Non-Guaranteed）；API 给父单挂止损子单只能走 slOrderType＝"PRESET"，参数镜像 TWS Presets 的全局设置——逐笔不同止损价只能自己下独立 STP/STP LMT 组合单＋OCA 自管。
3. **止损 overshoot 机制（optionsdecay 分钟级回放，非实盘）。** 权利金 $1,035 的铁鹰，50% 止损位 $517.50，实际退出 −$565（54.6%）：不是订单滑点（退出成交还比报价好 $0.125），缺口在「每分钟检查一次」的间隔里——最后一分钟组合 delta 从 −0.12 涨到 −0.37、实际 $47/点。姊妹篇：止损位 $362.50 实际 −$395（54.5%）；另一单进场 14 分钟被止损、SPX 全程没碰短腿仍亏 53%。对照：tastytrade 官方文档说价差止损限价按各腿 NBBO 合成触发、不是 mid——触发口径各家不同，不能跨券商套用。
4. **Chambless 2026 访谈（OptionOmega 整理，卖课方自报、未审计，仅量级参考；站内已有此条）**：MEIC 平均滑点从 2025 年约 $0.05/笔涨到 2026-06 的约 $0.13–0.15/笔（约 3 倍），她把滑点预先计入止损位；试过限价 bot 结论是几秒延迟反而成交更差，坚持市价止损；按「全天全止损」定仓、上限约账户 2%，坏日子会穿。
5. **个人实盘日志 Foolish Trader Day 124（自报未审计，入站 id 510）**：SPX 0DTE 铁鹰 30 笔 25 胜 +$2,675（15 宽 8 战全胜），唯一亏的一笔是 10 宽持有到结算 −$307；NDX 20 笔 −$6,649；周四下午 3 分钟连开四笔方向单 −$1,758 全损；全年佣金 $3,295 约为毛利一半。大亏来自不止损和赌气单，不是止损滑点。

**总判断**：Paper 退出设计第一步不是选止损倍数，是先实测「券商挂单 vs 自监控」两条路的触发口径、延迟和 overshoot 分布；止损预算公式要把检查间隔写进去（止损位＋间隔内可移动点数×当前每点美元数）；Paper 里对同一批铁鹰并行记录「按规则止损」与「影子持有到结算」两条曲线。

## I 路 · 带真数字的严肃证据（二线）

1. **Cboe 官方绩效表（入站 id 495）**：PUT（月度卖 ATM put）1986–2019 年化 9.7%、波动 9.9%（标普 10.4%/14.9%）；CNDR 铁鹰 2006–2019 最大回撤 −13.7%，全表最浅（PUT −32.7%、标普 −51.0%）；PUTVM（至 2026-10）年化 9.0%、夏普 0.65。口径：指数不计成本和税，上市前为回测假设绩效。结论：被动卖方的价值在回撤不在收益；PUTVM「高波动切去跟踪 CNDR」可借作状态开关参照。
2. **SAVNG 独立复现 tasty 规矩（站内已有，不重复入站）**：2010–2026、45 DTE 16Δ 宽跨、真实买卖价＋$0.65 佣金：50% 止盈胜率 85%（tasty 说法成立）但每笔收益 0.92%＜持有到期 1.26%——止盈只抬胜率、压期望；高 IV Rank 进场每笔只赚 0.31%、其余 1.1%，IV Rank 过滤方向是反的；铁鹰翼宽 2.5% 每笔 −3.2%、5% −0.4%、10% +2.4%，窄翼扣真实价差后亏钱；换个股标的每笔 −0.28%。
3. **Li & Miller 学术论文（EFMA 2025，入站 id 494）**：2010–2019 标普期权、按买卖价差成交、Cboe 保证金为资金基数：月度卖 5% 虚值宽跨 +3.40%/月，周度宽跨波动降到 4.54%；但 2020 样本跨式/防崩跨式/铁蝶（加翼）全部为负、亏损集中在 5 周内；拆腿：卖 put 长期为正、卖 call 长期为负。会议论文未正式发表。
4. **tastytrade Market Measures 2017（入站 id 501，成本口径未披露）**：SPY 2005–2017、只在 IV Rank＜10 买方：16Δ 宽跨按 50% 管理胜率 33%、平均亏；ATM 跨式按 25% 管理胜率 50%、平均亏。镜像证据：低 IV 买方＋提前管理在 tasty 自家 12 年数据里系统性亏钱。
5. **2manyoptions 0DTE 铁鹰全网格（入站 id 499，自报、厂商发表）**：2023–2026、100ms 报价、21,060 配置、1,578 万笔：扣佣金前平均每笔 −$2.08，只有 44% 配置为正；最差 10% 交易日平均 −$229/笔、其余 90% 是 +$26；年际衰减 2023 +$2.25 → 2026 −$6.23；全包约 $1.60/张/腿，一张铁鹰开仓费约 $6.40。
6. **如实报缺**：本轮没找到「审计或券商对账级、多年的卖方实盘记录（非卖课方）」新条目，不凑数。

## J 路 · 考卷落地的字段级细节

1. **PIXIU/FinBen 任务注册（入站 id 505）**：任务＝一个 HF 数据集（统一 JSON：query/answer/text，分类加 choices＋gold）＋一个继承预置基类的类（一行 DATASET_PATH）＋TASK_REGISTRY 里一行。夜航社照搬：scenarios.jsonl＋SCENARIO_REGISTRY＋runner 点名跑，增删场景不改代码。
2. **InvestorBench 出分口径（入站 id 506）**：四指标 CR/SR/AV/MDD、一资产一表、Buy & Hold 基线行；最值得抄的是每模型跑 5 次、报告「中位数轨迹」的成绩，治 LLM 随机翻号。它无成本、无 no-trade 统计，不能整表照搬。
3. **SWE-bench 冻结包（入站 id 508）**：instance 一行 JSON：instance_id、base_commit（冻结时点状态哈希）、problem_statement（唯一给作答者的部分）；金标 patch 与 test_patch 同包但不发；评分预先冻结为 FAIL_TO_PASS＋PASS_TO_PASS 两张名单、二值判定、无 LLM 裁判；每题钉一个定版 Docker 环境。夜航社场景包照此：base_state 快照＋prompt_inputs＋answer_key 分文件＋scoring_checks 两栏（MUST_PASS / MUST_NOT_BREAK）。
4. **KTD-Fin（2026，入站 id 507）**：十维面板含行为族——弃权率（重试后仍零下单的步数占比）、parse-failure 率、现金闲置率；成本写死在执行器（最低费、买 5bp/卖 15bp、次日开盘、T+1，无效动作只记录不修正）；四级掩码（真名/代码盲/日期盲/双盲）测记忆泄漏，去匿名攻击认证 top-1 还原 ≤3%。
5. **LiveOption 成绩单补遗（入站 id 511）**：episode 表六列 n / Mean% [95% CI] / Med% / Win% / NT% / PF；CI 用 10,000 次 bootstrap；比较一律配对＋sign-flip＋Holm 校正；no-trade episode 留在分布里计 0；成本敏感性是固定动作：滑点从 $0.0001 换实测半价差 $0.0050 后 0DTE 正 mean 全翻负（−0.216%→−0.452%，p=0.013）。

## K 路 · 对账落地二线（本轮最实的一路）

1. **ibflex 必须用新端点版本（入站 id 490）**：IBKR 关了旧 Flex 端点（全 1001），旧版还有硬编码 URL 导致无限重试死循环；修复 PR 2026-05-23 已并入 master（新 AccountManagement 端点、真 token 实测通过）。作者自述这是跟 IBKR 改接口的军备竞赛，建议直接跑 git master 不等 PyPI。
2. **Flex 下载纪律（入站 id 493）**：限额 1 次/秒、10 次/分钟/token；SendRequest 失败重发会触发未公开的 1025 token 锁定（实测约 14 小时、重试还延长）；1001 在 GetStatement 阶段＝还没生成好、继续轮询，在 SendRequest 阶段＝生成失败、绝不能重发；token 默认 6 小时过期，生成时必须把过期设为 1 年；数据每日收盘后才更新，多拉无益。落地：SendRequest 只发一次、只轮询同一 ReferenceCode、请求阶段错误一律 fail-fast 等下一轮。
3. **OptionEAE 入账规则（入站 id 492）**：官方字段表已核实、明确含指数现金结算。两个静默算错点：无价值到期会同时出现在 Trades（带完整 Realized P&L 的平仓行）和 OptionEAE，两边相加＝双计；被指派的期权 Realized P/L 记 0（权利金折进股票成本基），再单独记一次又是反向双计。规则定死：到期/结算只认 OptionEAE，Trades 同合约同日的到期类平仓行只交叉核对不入账。
4. **可借的骨架（入站 id 491、496）**：decaf 的「定期拉＋SQLite 落地累积」对冲 Flex 365 天窗口（硬要求：SELL 必须带 Closed Lots 的 Lot 明细与 openDateTime，缺了报错）；open-renta 的 OptionEAE 重建算法最完整——（标的、类型、行权价、到期日、乘数）同质键＋多空双 FIFO 队列合并两段事件流。另有 ibkr-flex-cli（TS）可作通用解析参考。三家全是税务/记账导向，没有一个做多腿往返配对。
5. **TradeTally v2.11.0（入站 id 497）**：修复「按券商订单号分组期权腿」——旧版会把不相干订单的腿串成一组；官方文档明说导入 Conid 是 CRITICAL，没 Conid 期权行可能错组或被拒。用它先升 ≥v2.11.0、Trades 段选 Executions 并勾 Conid，拿 SPXW 四腿铁鹰＋一笔单腿验分组。

**总判断**：对账落地顺序＝先按第 2 条写对下载纪律（别锁 token）→按第 3 条定死 OptionEAE 入账规则（别双计）→ibflex 按第 1 条核对版本即可用；其余工具只当参考实现或人看的账本。

## L 路 · 中文深水区（公开可读）

1. **雪球《A股期权日报》（入站 id 498）**：每日 16:00 更新，8 只 ETF 期权逐日给 IV30、IVR 一年分位、IV−HV、skew、PCR、持仓墙、max pain、逐只 GEX 与双墙。规矩写死：IVR≥50 才卖、GEX 只当环境过滤器不改进场规矩、卖方分档止盈（近月 50%、远月 30/20/10%）、每条想法带失效条件。标的是 A 股 ETF 不是 SPX，但整套日报框架可直接移植成夜航社的 SPX 每日环境判断模板。
2. **《期权墙 · SPX 日内实战入门》（入站 id 503）**：中文里极少见的把 GEX 读图写成完整日内流程的书：盘前 OI GEX 标双墙与 Flip，10:00–15:00 切 Volume GEX，尾盘 30–15 分钟 OI 权重回升读 Pin；GVP＝GEX＋VWAP＋价格行为三者同向才动手；出场三档（50% 减半→触墙再减→破 Flip/VWAP 全走）；事件日等 15–30 分钟。墙位统计是二手引用 SpotGamma 2019–2024（Call Wall 83% 日高未破、Put Wall 89%），无自有回测，只有买方篇。
3. **0DTE 评分体系设计稿（Substack 中文，入站 id 502）**：100 分七组权重（行情 20/方向 20/催化 15/波动定价 15/流动性 10/时间与 Gamma 10/风险收益 10 且有否决权）；许可门写死（＜60 禁做、60–69 观察/模拟、70–79 标准单位、80+ A 级）；风险单位有算例（$10k 账户、单笔风险 0.5%＝$50、25% 止损→仓位上限 $200）；三重止损＋Daily Loss Limit＋Giveback＋Brier 校准。作者自述是训练营设计稿、无成绩——正好和夜航社四组判断格式逐项对照。
4. **vnpy_optionmaster（入站 id 504）——纠正第一轮结论**：第一轮说「组合级 Greeks 风控无成熟开源成品」，英文圈确实没有，但这个中文仓库（73 stars、MIT、有 tests）有现成实现：组合持仓 Greeks 风控、Delta 自动对冲、波动率曲面跟踪、情景压力测试。面向国内 CTP 不是 SPX，借结构要自己换数据源。
5. **B站「小白学期权」合集（入站 id 500）**：三段成体系的中文课程（Greeks 逐个讲、20 多集策略含铁鹰搭建与风控和 0DTE 专集、波动率专题），适合当中文术语与策略结构对照教材；止步教学层、无回测与参数表。
6. **如实报空**：YouTube 中文频道没挖到过关的（两个候选都是营销导流）；知乎专栏没挖到能打开全文且方法具体的长文。

## 本轮三条总修正

- 「组合 Greeks 无开源成品」要改口：英文圈没有，中文 vnpy 有参考实现（id 504）。
- 对账的最大风险不在解析库，在调用纪律和入账规则：token 锁 14 小时、到期双计——这两条 Paper 成绩会静默失真，先于任何策略问题。
- 止损的真实亏损＝止损位＋监控间隔的 gamma 溢出＋滑点（三年涨约 3 倍到 $0.13–0.15/笔）；IBKR 组合止损是模拟触发，这三件事决定退出模块先测什么。
