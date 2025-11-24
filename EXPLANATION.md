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
`disposable = net(税后收入) + employerBenefit(雇主公积金) + lump(仲裁一次性补偿) − selfPay(失业续保) − oppCost(失业机会成本)`。

- `gross`：根据场景的全薪/降薪/待岗阶段决定毛收入；无收入月份为 0。
- `net`：由 `calcAfterTaxIncome` 计算，等于 `gross + monthlyBonusSmoothed − 个人五险一金 − 有效税`，如果当月毛收入为 0 则不计提五险一金。
- `employerBenefit`：当月有工资/补贴时计入的公积金入账额（公司 + 个人），避免个人公积金在净收入中扣除后未加回。
- `lump`：仲裁月份（`arbitrationDurationMonths`）按期望赔付×仲裁概率计入一次性补偿。
- `selfPay`：仅在无工资且选择续保时计入的自付社保固定支出。
- `oppCost`：仅在无工资月份扣除的机会成本，等于 `expectedReemploymentIncome × unemploymentOpportunityCostFactor`。

## 参数释义（按表单分组）
- **收入**：
  - `monthlyGrossSalary` 税前月薪，控制全薪/降薪/待岗阶段的起算基数。
  - `monthlyBonusSmoothed` 平滑奖金，直接并入毛收入后计税。
  - `salaryCutRatio` / `standbyRatio` 分别决定降薪与待岗阶段的支付比例。
- **五险一金**：
  - `socialBase` 为养老/医疗/失业的计提基数；个人部分只在有收入时扣除。
  - `housingBase` × (`housingEmployeeRate` + `housingEmployerRate`) 构成公积金入账，个人扣除的部分在 `employerBenefit` 中加回。
  - `keepSocialSecurityDuringUnemployment` 与 `selfSocialSecurityCostPerMonth` 控制失业期是否自行续保及月成本。
- **概率与贴现**：
  - `discountRateAnnual` 与 `timeStepMonths` 决定折现率；`riskAversionLambda` 用于指数效用和确定性等价计算。
  - `p_*` 概率组描述仲裁获赔倍数的分布；`p_exec_*` 描述执行率分布，若场景未指定执行率则使用该期望值。
- **法律/补偿**：
  - `N_years`、`N_baseSalary`、`N_legalCapMultiple` 决定法定 N 的基数，`hasTwoNCase` / `twoN_maxMultiple` 仅影响 2N 情形的封顶。
  - `arbitrationExtraN` 为额外赔付倍数，`arbitrationDurationMonths` 为仲裁到账期，`probUseArbitration` 为提起仲裁的概率。
  - `arbitrationWinFactor` 表示证据强度/支持度，`executionSuccessRate` 为场景层面的执行成功率（优先级高于全局）。
- **失业与机会成本**：
  - `expectedInJobMonths` 设定继续发薪的预期月数并参与期限截断。
  - `expectedReemploymentIncome` 和 `unemploymentOpportunityCostFactor` 决定失业期的机会成本扣减。
- **管理人约束**：
  - `managerMaxCost` 与 `managerMaxMonthsPay` 用于判断场景是否满足“可行”限制。

## 场景表与参数映射
- **全薪月 / 降薪比 / 降薪月 / 待岗月 / 待岗支付** → 决定 `gross` 的阶段收入，进入税后计算。
- **额外N / 仲裁月 / 仲裁概率** → 影响仲裁一次性补偿 `lump` 的期望值与发生时间。
- **支持度** → 映射为 `arbitrationWinFactor`，调节仲裁胜率或证据强度。
- **执行率** → 使用场景内的 `executionSuccessRate`，若未填则退回全局执行概率期望 `expectedExecutionFactor`。

## 关键公式的代码锚点
- `calcAfterTaxIncome`：奖金并入毛收入后再扣五险一金与有效税率，得到 `net`。
- `buildTimeline`：按场景阶段生成 `gross`，计算 `net`、`employerBenefit`、`selfPay`、`oppCost`、仲裁补偿 `lump`，并应用折现因子得到 `discounted`，最终形成月度现金流 `disposable`。

## S1–S6 分场景公式（仅说明，不改代码）
以下均按“完结即停”梳理：发薪阶段按全薪/降薪/待岗发放；一旦进入仲裁，工资/补贴停发，仅扣续保和机会成本，直到仲裁月一次性计入期望赔付，之后不再计成本。额外 N 视为对降薪的补差，不在输入里重复填写。

公共符号：
- `baseN = 12 × 30,300 = 363,600`（工龄按 12 年封顶）。
- 赔付倍数期望 `E[mult] = 0.865`（2N/1N/0.8N/0.5N/0 的概率加权）。
- 自付社保 `selfPay = 2,000`，机会成本 `opp = 15,000 × 0.2 = 3,000`，仅在无工资月份扣除。
- 月折现因子 `disc_m = 1 / (1 + monthlyRate)^(m-1)`，`monthlyRate = (1+20%)^(1/12)-1`。
- 月净收入 `net(g) = g - 社保公积金个人缴 + 奖金(0) - 20% 个税`，个人缴按 30,300 为基数；有工资/补贴时加回雇主+个人公积金 `HF = 30,300 × (12%+12%) = 7,272`。
- 仲裁期望赔付 `P(s) = baseN × (1+extraN_s) × win_s × 0.865 × exec_s`，实际入账 `P(s) × probUseArb_s`，在该场景的仲裁终结月计入。

### S1：3 月全薪主动离职
- **发薪阶段**（1–8 月）：1–3 月全薪；4–5 月按 50% 降薪；6–8 月按 6.5% 待岗，每月现金流 `cash_m = net(gross_m) + HF`。
- **仲裁等待**（9–14 月，共 6 期）：工资停发，`cash_m = -selfPay - opp = -5,000`。第 14 月加一次性 `P(S1) × 0.2`，其中 `P(S1) = 363,600 × 1.0 × 0.9 × 0.865 × 0.625 ≈ 176,914`。
- **NPV**：把 14 期 `cash_m` 分别乘 `disc_m` 后求和。

### S2：4 月全薪主动离职
- **发薪阶段**（1–8 月）：1–4 月全薪；5–6 月 50% 降薪；7–8 月 6.5% 待岗，`cash_m = net(gross_m) + HF`。
- **仲裁等待**（9–16 月，共 8 期）：`cash_m = -5,000`。第 16 月加 `P(S2) × 0.8`，`P(S2) = 363,600 × 1.8 × 1.0 × 0.865 × 0.5 ≈ 283,063`。
- **NPV**：16 期折现求和。

### S3：2 月降薪 + 仲裁
- **发薪阶段**（1–13 月）：1–2 月全薪；3–5 月 50% 降薪；6–13 月 6.5% 待岗，`cash_m = net(gross_m) + HF`。
- **仲裁等待**（14–23 月，共 10 期）：`cash_m = -5,000`。第 23 月加 `P(S3) × 0.7`，`P(S3) = 363,600 × 1.4 × 0.8 × 0.865 × 0.5 ≈ 176,128`。
- **NPV**：23 期折现求和。

### S4：全薪 5 月后和平离职
- **发薪阶段**（1–5 月）：全薪，`cash_m = net(30,300) + HF`；第 5 月为终结月，之后不再扣任何成本。
- **仲裁（极低概率）**：若触发 0.05 概率仲裁，则在第 5 月计入一次性 `P(S4) × 0.05`，`P(S4) = 363,600 × 1.0 × 0.5 × 0.865 × 0.75 ≈ 117,943`。
- **NPV**：仅对 1–5 月（含可能的一次性赔付）折现求和。

### S5：6 月降薪 + 6 月待岗
- **发薪阶段**（1–12 月）：1–4 月全薪；5–10 月 50% 降薪；11–12 月 6.5% 待岗，`cash_m = net(gross_m) + HF`。
- **仲裁等待**（13–19 月，共 7 期）：`cash_m = -5,000`。第 19 月加 `P(S5) × 0.6`，`P(S5) = 363,600 × 1.5 × 0.85 × 0.865 × 0.5 ≈ 200,503`。
- **NPV**：19 期折现求和。

### S6：12 个月待岗 + 1N 和解
- **发薪阶段**（1–5 月）：1–3 月全薪；第 4 月 50% 降薪；第 5 月 6.5% 待岗，`cash_m = net(gross_m) + HF`。
- **仲裁等待**（6–9 月，共 4 期）：`cash_m = -5,000`。第 9 月加 `P(S6) × 0.3`，`P(S6) = 363,600 × 1.1 × 0.6 × 0.865 × 0.65 ≈ 134,927`。
- **NPV**：9 期折现求和。

> “总期数” = 发薪期长度 + 仲裁等待长度；终结月份之后不再扣社保续保和机会成本，符合“完结即停”的假设。
