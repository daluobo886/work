# 公式与参数解释

## 奖金 (monthlyBonusSmoothed)
- 定义：在参数表单的“平滑月度奖金”字段中录入的税前奖金，用于将年终奖/季度奖等平均到每个月。
- 用法：在计算当月税后收入时直接加到毛收入 `gross` 上，再扣除个人五险一金和有效税率后的个税，得出税后净收入 `net`。
- 代码位置：`calcAfterTaxIncome` 中先把奖金与工资合并，再计算个税与净收入。

## 折现率与折现因子
- 月化步骤：`monthlyRate = (1 + discountRateAnnual)^(timeStepMonths/12) - 1`，支持自定义时间步（默认为 1 个月）。
- 每期折现：`disc = 1 / (1 + monthlyRate)^(m-1)`，其中 `m` 为从 1 开始的期数。
- 等价说明：当 `timeStepMonths = 1` 时，上式与 `1 / (1 + discountRateAnnual)^((m-1)/12)` 数学等价；当前写法能兼容非常用的时间步长。

## 每月净现金流公式
`disposable = net(税后收入) + employerBenefit(雇主公积金) + lump(仲裁一次性补偿) − selfPay(失业续保) − oppCost(机会成本)`。

- `gross`：根据场景的全薪/降薪/待岗阶段决定毛收入；无收入月份为 0。
- `net`：由 `calcAfterTaxIncome` 计算，等于 `gross + monthlyBonusSmoothed − 个人五险一金 − 有效税`，如果当月毛收入为 0 则不计提五险一金。
- `employerBenefit`：当月有工资/补贴时计入的公积金入账额（公司 + 个人），避免个人公积金在净收入中扣除后未加回。
- `lump`：仲裁月份（`arbitrationDurationMonths`）计入一次性补偿，包含“法定 N 期望值 × 支持度 × 执行率”以及自动算出的降薪/待岗工资补回（同样乘以支持度与执行率），不再额外乘仲裁概率。
- `selfPay`：仅在仲裁等待期（无工资阶段）且选择续保时计入的自付社保固定支出，完结后不再扣。
- `oppCost`：事项未完结前逐月扣除的机会成本，等于 `expectedReemploymentIncome × unemploymentOpportunityCostFactor`，在完结后停止。

## 参数释义（按表单分组）
- **收入**：
  - `monthlyGrossSalary` 税前月薪，控制全薪/降薪/待岗阶段的起算基数。
  - `monthlyBonusSmoothed` 平滑奖金，直接并入毛收入后计税。
  - `salaryCutRatio` / `standbyRatio` 分别决定降薪与待岗阶段的支付比例。
- **五险一金**：
  - `socialBase` 为养老/医疗/失业的计提基数；个人部分只在有收入时扣除。
  - `housingBase` × (`housingEmployeeRate` + `housingEmployerRate`) 构成公积金入账，个人扣除的部分在 `employerBenefit` 中加回。
  - `keepSocialSecurityDuringUnemployment` 与 `selfSocialSecurityCostPerMonth` 控制仲裁等待期是否自行续保及月成本。
- **概率与贴现**：
  - `discountRateAnnual` 与 `timeStepMonths` 决定折现率；`riskAversionLambda` 用于指数效用和确定性等价计算。
- `p_*` 概率组描述仲裁获赔倍数的分布，自动加权得到“支持度”（期望倍数）；`p_exec_*` 描述执行率分布，自动加权得到期望执行率，两者均直接用于 DCF 无需手填。
- **法律/补偿**：
  - `N_years`、`N_baseSalary`、`N_legalCapMultiple` 决定法定 N 的基数，`hasTwoNCase` / `twoN_maxMultiple` 仅影响 2N 情形的封顶。
- `arbitrationDurationMonths` 为仲裁结算月（总期数=发薪期+仲裁等待期）。
- 降薪/待岗工资补回自动按“支持度（赔付档概率加权倍数）× 执行率（执行档概率加权）”计入仲裁月，无需手填支持度/执行率。
- **失业与机会成本**：
  - `expectedInJobMonths` 设定继续发薪的预期月数并参与期限截断（现按“发薪期 + 仲裁等待期”截断）。
  - `expectedReemploymentIncome` 和 `unemploymentOpportunityCostFactor` 决定机会成本扣减（完结即停）。
- **管理人约束**：
  - `managerMaxCost` 与 `managerMaxMonthsPay` 用于判断场景是否满足“可行”限制。

## 场景表与参数映射
- **全薪月 / 降薪比 / 降薪月 / 待岗月 / 待岗支付** → 决定 `gross` 的阶段收入，进入税后计算；差额自动计入“降薪/待岗补回”，再乘支持度与执行率。
- **仲裁月** → 决定一次性补偿 `lump` 的折现时点；不再单独设置“仲裁概率”或额外 N。
- **支持度** → 由赔付档概率自动加权成期望倍数并用于仲裁补偿与工资补回。
- **执行率** → 由执行档概率自动加权成期望执行率并用于仲裁补偿与工资补回。

## 关键公式的代码锚点
- `calcAfterTaxIncome`：奖金并入毛收入后再扣五险一金与有效税率，得到 `net`。
- `buildTimeline`：按场景阶段生成 `gross`，计算 `net`、`employerBenefit`、`selfPay`、`oppCost`、仲裁补偿 `lump`（含工资补回），并应用折现因子得到 `discounted`，最终形成月度现金流 `disposable`。

## S1–S6 分场景公式（完结即停）
公共符号：
- `baseN = 12 × 30,300 = 363,600`（工龄按 12 年封顶）。
- 赔付倍数期望 `E[mult] = 0.865`（2N/1N/0.8N/0.5N/0 的概率加权）。
- 自付社保 `selfPay = 2,000`，机会成本 `opp = 15,000 × 0.2 = 3,000`，直至终结月前逐月扣除。
- 月折现因子 `disc_m = 1 / (1 + monthlyRate)^(m-1)`，`monthlyRate = (1+20%)^(1/12)-1`。
- 仲裁期望赔付 `P(s) = baseN × win_s × 0.865 × exec_s`，降薪/待岗补回 `M(s) = 补差 × win_s × exec_s`，均在仲裁终结月一次性计入。

### S1：3 个月全薪主动离职
- **发薪阶段**（1–3 月）：每月现金流 `cash_m = net(全薪) + 公积金 − opp`；第 3 月为终结月，之后不再扣成本。
- **NPV**：仅对 1–3 月折现求和。

### S2：4 个月全薪主动离职
- **发薪阶段**（1–4 月）：全薪入账并扣机会成本；第 4 月终结，之后无等待期与扣减。
- **NPV**：仅对 1–4 月折现求和。

### S3：2 个月降薪 + 仲裁
- **发薪阶段**（1–2 月）：按 50% 降薪发放，降薪差额在仲裁结算月补回并乘支持度×执行率；每月现金流含机会成本扣减。
- **仲裁等待**（3–10 月，共 8 期）：工资停发，仅扣 `selfPay + opp`。第 10 月计入 `P(S3) + 降薪补回`。

### S4：2 个月待岗 + 仲裁
- **发薪阶段**（1–2 月）：按 6.5% 待岗补贴发放，差额在仲裁月补回并乘支持度×执行率；每月现金流含机会成本。
- **仲裁等待**（3–10 月，共 8 期）：仅扣 `selfPay + opp`。第 10 月计入 `P(S4) + 待岗补回`。

### S5：6 个月降薪 + 6 个月待岗
- **发薪阶段**（1–6 月）：按 50% 降薪发放；7–12 月按 6.5% 待岗，均扣机会成本。无仲裁，终结月为第 12 月。
- **NPV**：对 1–12 月现金流折现求和。

### S6：12 个月待岗
- **发薪阶段**（1–12 月）：全程按 6.5% 待岗支付，同时扣机会成本；第 12 月为终结月，无仲裁等待与后续扣减。
- **NPV**：对 1–12 月折现求和。

> “总期数” = 发薪期长度 + 仲裁等待长度；终结月份之后不再扣社保续保和机会成本，且续保与正常缴纳不会在同一期同时出现。
