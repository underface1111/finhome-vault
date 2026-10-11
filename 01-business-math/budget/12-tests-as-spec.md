# 12 — Tests as executable spec

7 test suites, **561 tests pass** (`npx jest features/budget/utils/__tests__`). Run time ~2.5s — fast feedback loop.

> Tests = the ground truth. Khi doc và code lệch, tests cho biết behavior thực tế. Khi sửa logic, sửa test cùng commit.

## Suite map

| Suite                                | LOC   | Vai trò                                        |
| ------------------------------------ | ----- | ---------------------------------------------- |
| `budget-calculator.test.ts`          | 830   | Core engine — constants, dual constraint, DTI, LTV, 2-phase, limiting factor, insight tiers, hidden costs, segments |
| `budget-production-readiness.test.ts`| 909   | 5 VN personas × full pipeline + limiting-factor messaging |
| `user-journey-integration.test.ts`   | 967   | End-to-end Budget → Loan Packages → Detail data linkage  |
| `loan-comparison.test.ts`            | 326   | Bank comparison + repayment schedule + stress test + sort |
| `interest-consistency.test.ts`       | 66    | Cross-module: budget-calc vs loan-comparison khớp |
| `safety-flags.test.ts`               | 178   | 3-issue fix (belowMarketFloor + lifeEventBufferLow + chip reconciliation) |
| `what-if.test.ts`                    | 71    | Scenario simulation utilities                  |
| `budget-analytics.test.ts`           | 80    | PII bucketing helpers                          |

Total **3,427 LOC test** · **578 tests passing**. Test:code ratio > 1:1 — high confidence.

## 3-issue fix (06-2026) — verified by `safety-flags.test.ts`

### Issue 1 — chip ↔ banner conflict (chip xanh trong khi banner cam)

**Fix**: `getSafetyChip()` accepts engine flags từ BudgetResult, override targetDTI-derived tone. Priority order tested.

| Engine flag             | Chip label                        | Tone     |
| ----------------------- | --------------------------------- | -------- |
| `isBlocked`             | "Chưa đủ điều kiện"               | danger   |
| `belowMarketFloor`      | "Dưới ngưỡng"                     | danger   |
| `affordabilityWarning`  | "Vượt khả năng"                   | risky    |
| `lifeEventBufferLow`    | "Đệm mỏng"                        | caution  |

> Labels kept ≤ 18 chars by test invariant — chip phải glanceable + fit cạnh title trên màn hình hẹp. Long-form explanation chuyển sang banner/insight body.
| (none) — fallback       | từ targetDTI ladder               | safe/caution/risky/danger |

### Issue 2 — market floor missing

**Fix**: thêm `MARKET_FLOOR_PRICE = 500_000_000` const. Engine flag `belowMarketFloor` khi `safeLoan > 0 AND maxPrice < 500M`. Insight override → "Chưa đạt ngưỡng giá thị trường" với gợi ý tăng vốn `(MARKET_FLOOR − maxPrice) / 5`.

Verified scenario: income 10M + down 50M → maxPrice 250M → `belowMarketFloor: true`, insight override, chip "danger".

### Issue 3 — life event buffer thin

**Fix**: thêm `LIFE_EVENT_BUFFER_RATIO = 0.2`. Engine flag `lifeEventBufferLow` khi `0 ≤ disposable < income × 20%`. Mutually exclusive với `affordabilityWarning` (disposable < 0).

Verified scenarios:
- Asia 55% perspective (30M income → install 16.5M → disposable 1.5M < 6M threshold) → **fires**.
- Default UI persona (70M income → install 25.2M → disposable 30M > 14M threshold) → **does NOT fire** (defaults stay green).

## What tests verify (key invariants)

### Constants — frozen contract
```ts
DEFAULT_PREFERENTIAL_RATE === 8.5
DEFAULT_FLOATING_RATE === 12.0
DEFAULT_PREFERENTIAL_YEARS === 2
DEFAULT_LOAN_TERM === 20
DTI_PRESETS.global === 36, DTI_PRESETS.asia === 55
CUSTOM_DTI_MIN === 36, CUSTOM_DTI_MAX === 80
MAX_DTI === 43
TARGET_DTI === DTI_PRESETS.global
MIN_DOWN_PAYMENT_RATIO === 0.2
MAX_LTV_RATIO === 0.8
MIN_DOWN_PAYMENT_RATIO + MAX_LTV_RATIO === 1.0   ← invariant check
```

### Core math — verified properties

| Invariant                                                   | Test                                  |
| ----------------------------------------------------------- | ------------------------------------- |
| `safeLoanAmount = min(loanFromIncome, loanFromLTV)`         | budget-calc:216-223                   |
| `maxAffordablePrice = safeLoan + downPayment`               | budget-calc:198-203                   |
| `safeLoan ≤ loanFromIncome` AND `safeLoan ≤ loanFromLTV`    | budget-calc:204-215                   |
| `LTV ≤ 80%` (MAX_LTV_RATIO)                                 | budget-calc:257-262                   |
| `LTV = safeLoan / maxPrice × 100`                           | budget-calc:263-269                   |
| `monthlyInstallment === monthlyFloating` (worst case)       | budget-calc:229-234                   |
| `monthlyPreferential < monthlyFloating`                     | budget-calc:235-241                   |
| `dtiRatio = (debt + installment) / income × 100`            | budget-calc:247-255                   |
| `weightedRate ∈ (prefRate, floatRate)`                      | budget-calc:281-289                   |
| `totalAmountPaid > safeLoan` (principal + interest)         | budget-calc:290-295                   |
| `totalInterestPaid = totalAmountPaid − safeLoan`            | budget-calc:296-303                   |

### DTI perspective behavior

| Property                                                          | Test                |
| ----------------------------------------------------------------- | ------------------- |
| Higher ceiling → larger budget (monotonic) when income-limited    | budget-calc:65-72   |
| Picked % == displayed ratio at income-limited cap (no debt)       | budget-calc:73-79   |
| Out-of-range ceilings clamped to `[36, 80]`                       | budget-calc:80-90   |
| Omitting `targetDTI` defaults to global 36 (backward compat)      | budget-calc:91-99   |
| `estimateBudgetPreview` honors targetDTI (matches full calc)      | budget-calc:100-110 |

### Income-based ceiling

| Property                                                          | Test                |
| ----------------------------------------------------------------- | ------------------- |
| Living expenses do NOT shrink borrowing cap (income-based design) | budget-calc:112-128 |
| `affordabilityWarning` raised when payment > disposable           | budget-calc:130-140 |
| No warning for comfortable budget                                 | budget-calc:141-155 |

### Cross-module consistency

`interest-consistency.test.ts` verifies:
- 2-phase total interest in `calculateBudget` matches `loan-comparison` per-package math.
- Same `(loanAmount, prefRate, prefYears, floatRate, term)` → same `totalInterestPaid`.

→ Phòng drift khi sửa 1 file. Critical invariant cho user trust (Budget tổng chi phí phải khớp khi user chọn bank tương đương).

`user-journey-integration.test.ts` verifies:
- `safeLoanAmount` từ BudgetResult → đầu vào `computeLoanComparison` → mỗi gói vay show monthly khớp với BudgetResult's `monthlyInstallment` cho tương đương rate.
- DTI consistency cross-screens.

### Production readiness — VN personas

5 user personas test trong `budget-production-readiness.test.ts`:

| Persona                                  | Profile                              | Expected verdict        |
| ---------------------------------------- | ------------------------------------ | ----------------------- |
| Nhân viên văn phòng (first-buyer)        | 20M income, 200M savings             | safeLoan modest         |
| Vợ chồng trẻ có nợ                       | 40M combined, 5M debt, 500M saved    | DTI tight               |
| Nhà đầu tư cá nhân                       | 100M income, 0 debt, 3 tỷ down       | LTV-limited (cash rich) |
| Thu nhập tối thiểu                       | 5M income, 50M savings               | safeLoan ≈ 0            |
| Stressed (high debt)                     | 20M income, 8M debt                  | Zero-loan special case  |

Mỗi persona test xuyên suốt pipeline → đảm bảo edge cases real-world không crash + insight/limiting factor copy meaningful.

### Limiting factor messaging

3 nhánh tested (`budget-production-readiness.test.ts`):
- Income-limited
- Downpayment-limited
- With existing debt → must suggest debt reduction
- No debt → must NOT mention debt reduction
- Edge: DTI exactly at threshold

→ [[06-insights-and-limits]] copy decisions xác thực bằng test.

### Loan comparison invariants

| Property                                          | Test          |
| ------------------------------------------------- | ------------- |
| Sort by `totalCost` ascending                     | loan-comp     |
| Each pkg has exactly 1 badge (no collision)       | loan-comp     |
| All packages tested at same `effectiveTerm`       | loan-comp     |
| Real-world scenarios (small, medium, large loan)  | loan-comp     |
| `generateRepaymentSchedule` emits 2 entries at transition year | loan-comp |
| `BANK_LOAN_PACKAGES` derives from `BANK_PACKAGES` (no drift)   | loan-comp |

### Stress test invariants
- `stressedRate = floatingRate + stressIncrease`
- `stressedMonthly > normalMonthly` (rate up → payment up)
- DTI ratio increases with stress
- `existingMonthlyDebt` defaults to 0 (backward compat)

### Analytics bucketing (PDPL compliance)

`budget-analytics.test.ts` ensures `bucketIncomeOrExpenses`, `bucketMonthlyDebt`, `bucketDownPayment` map raw VND vào coarse ranges trước khi gửi PostHog. Test prevents accidental raw-PII leak.

## Coverage gaps (no tests yet)

- `risk-copy.ts` 4-tone ladder transitions (`getCustomRiskCopy` thresholds 45/55/70).
- `loan-insights.ts:generateInsight` per-tier copy assertion (only integration-tested).
- `assumptions-snapshot.ts` defaults filling.
- `acm-api.ts:computeAffordability` orchestration (logic covered by `pitia-calculator` + `rate-stress` indirectly).
- UI screens (`*.tsx`) — pure unit tests cover utils only. UI need RTL/Detox separately.

## How to run

```bash
npx jest features/budget/utils/__tests__              # all 561
npx jest features/budget/utils/__tests__/budget-calculator   # single suite
npx jest --watch features/budget                      # watch mode
```

→ [[04-budget-engine]] · [[09-loan-comparison]] · [[06-insights-and-limits]]
