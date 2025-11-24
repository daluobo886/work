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
