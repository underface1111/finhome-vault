# 02 — Data contracts

## `BudgetInput` — what goes in

```ts
interface BudgetInput {
  netIncome: number;          // Thu nhập ròng/tháng (VND)
  monthlyExpenses?: number;   // Chi phí sinh hoạt/tháng — affordability check
  monthlyDebt: number;        // Trả nợ/tháng (CHƯA tính khoản vay mới)
  downPayment: number;        // Vốn tự có — VND tuyệt đối, không phải %
  interestRate?: number;      // Lãi suất user chọn (%/năm); default 12% (thả nổi)
  loanTerm?: number;          // Kỳ hạn (năm); default 20
  targetDTI?: number;         // "Mức trả nợ an toàn" %; default 36
}
```

**Lưu ý**:
- `monthlyExpenses` là **optional** vì income-based cap **không** dùng nó để giới hạn vay. Nó chỉ feed vào affordability check (`disposableAfterPayment < 0` → warning).
- `interestRate` ở đây là **lãi suất thả nổi** (worst-case), không phải lãi ưu đãi.
- `targetDTI` được `clampTargetDTI()` về `[36, 80]` nếu out of bounds.

## `BudgetResult` — what comes out

Output shape rất giàu — phục vụ nhiều card trên BudgetResultScreen.

```ts
interface BudgetResult {
  maxAffordablePrice: number;    // = safeLoan + downPayment
  safeLoanAmount: number;        // min(loanFromIncome, loanFromLTV)
  simulatedLoanAmount?: number;  // dùng khi safeLoan=0 nhưng có vốn — preview gói vay
  monthlyInstallment: number;    // = monthlyFloating (worst case)
  downPayment: number;
  minDownPaymentRequired: number; // 20% của maxPrice
  affordabilityWarning?: boolean; // disposable < 0
  belowMarketFloor?: boolean;     // maxPrice < 500M (no VN property)
  lifeEventBufferLow?: boolean;   // 0 ≤ disposable < income × 20%
  loanAssumptions: { /* 12 fields */ };
  insight: { title, description, loanPackageButtonText? };
  hiddenCosts: { total, items[] };
  suitableSegments: { currentSegment, segments[] };
  limitingFactor: { type: "income"|"downpayment", title, description, loanFromIncome, loanFromLTV };
  stressTest?: { stressedRate, stressedMonthly, normalMonthly, stressedDTI, increase };
}
```

### `loanAssumptions` chi tiết

| Field                  | Ý nghĩa                                          |
| ---------------------- | ------------------------------------------------ |
| `preferentialRate`     | Lãi ưu đãi (default 8.5%)                        |
| `preferentialYears`    | 2 năm                                            |
| `floatingRate`         | Lãi thả nổi (user input hoặc default 12%)        |
| `weightedAverageRate`  | TB có trọng số trên loanTerm                     |
| `loanTerm`             | Năm                                              |
| `ltv`                  | % loan-to-value                                  |
| `targetDTI`            | Ceiling user chọn                                |
| `dtiRatio`             | Tỷ lệ THỰC TẾ (debt+install)/income              |
| `monthlyPreferential`  | PMT giai đoạn ưu đãi                             |
| `monthlyFloating`      | PMT giai đoạn thả nổi (= monthlyInstallment)     |
| `totalInterestPaid`    | Tổng lãi 2-phase                                 |
| `totalAmountPaid`      | Gốc + lãi                                        |

## Constants — single source of truth

Tất cả ở `features/budget/utils/budget-calculator.ts`:

```ts
DEFAULT_PREFERENTIAL_RATE = 8.5   // %/năm — ưu đãi 2 năm đầu (2025-2026 VN market)
DEFAULT_PREFERENTIAL_YEARS = 2
DEFAULT_FLOATING_RATE = 12.0      // %/năm — worst-case cap
DEFAULT_LOAN_TERM = 20            // năm

DTI_PRESETS = { global: 36, asia: 55 }
CUSTOM_DTI_MIN = 36
CUSTOM_DTI_MAX = 80
TARGET_DTI = 36                   // = DTI_PRESETS.global
MAX_DTI = 43                      // back-end ceiling ngân hàng VN

MIN_DOWN_PAYMENT_RATIO = 0.2      // 20% min
MAX_LTV_RATIO = 0.8               // 80% max LTV — SBV regulation

MARKET_FLOOR_PRICE = 500_000_000  // VN real estate floor — below = unfindable
LIFE_EVENT_BUFFER_RATIO = 0.2     // disposable ≥ 20% income for life shocks
```

UI-level constants ở `features/budget/constants/budget-defaults.ts`:

```ts
DEFAULT_NET_INCOME = 70_000_000        // 70 triệu
DEFAULT_MONTHLY_EXPENSES = 15_000_000  // 15 triệu
DEFAULT_MONTHLY_DEBT = 0
DEFAULT_DOWN_PAYMENT = 1_000_000_000   // 1 tỷ
DEFAULT_INTEREST_RATE = 10.5           // user-displayed default (khác 12% của engine)
DEFAULT_LOAN_TERM_YEARS = 20

// Slider bounds
MIN_NET_INCOME = 5M       MAX_NET_INCOME = 200M
MIN_MONTHLY_EXPENSES = 0  MAX_MONTHLY_EXPENSES = 100M
MIN_MONTHLY_DEBT = 0      MAX_MONTHLY_DEBT = 100M
MIN_DOWN_PAYMENT = 50M    MAX_DOWN_PAYMENT = 5_000M
MIN_INTEREST_RATE = 5.0   MAX_INTEREST_RATE = 20.0
MIN_LOAN_TERM_YEARS = 5   MAX_LOAN_TERM_YEARS = 35
```

> **Note**: `DEFAULT_INTEREST_RATE` UI = 10.5% nhưng `DEFAULT_FLOATING_RATE` engine = 12%. User mặc định nhập 10.5% (lạc quan hơn). Engine fallback 12% chỉ khi user **không** truyền `interestRate` (vd Compass, what-if, tests).

## `HousingSegment` enum

```ts
type HousingSegment = "social" | "suburban" | "mid-range" | "high-end";
```

Threshold: <1.5tỷ social · 1.5-2.5tỷ suburban · 2.5-5tỷ mid · ≥5tỷ high.

Xem [[07-segments-and-costs]].

## ACM types (separate engine, neutrality posture)

Định nghĩa ở `features/budget/types/acm.types.ts`. Refusal at type level:
**NO `bank_id` / `lender_name` / `counterparty_id` anywhere.**

- `TargetPriceBand extends ConfidenceBand { midpoint }` — band, không bao giờ là parcel point.
- `RateStressPoint { deltaPp: 1|2|3, stressedRate, stressedMonthly, stressedDtiBack, breaches }`
- `PitiaBreakdown { principalInterest, propertyTax, insurance, hoaOrMgmt, total }`
- `AssumptionsSnapshot` — captured at compute time, CDR embeds VERBATIM (A2 decision).
- `AffordabilityState` — full surface engine spits out.

Xem [[10-acm-engine]].

## Persistence shape

`SavedBudgetResult { id, savedAt, input, result }` — store giữ tối đa 20 entries. Migrate logic trong store handle các saved cũ thiếu `input`.

→ [[03-core-math]] · [[04-budget-engine]] · [[11-state-and-api]]
