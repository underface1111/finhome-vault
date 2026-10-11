# 09 — Loan comparison

Khi user có `safeLoanAmount`, tab "Xem gói vay phù hợp" → so sánh **5 gói vay** từ `BANK_REGISTRY`.

## Module: `loan-comparison.ts`

3 export chính:
1. `computeLoanComparison(loanAmount, term, basePackages?)` → `LoanPackage[]` enriched.
2. `sortLoanPackages(packages, key)` → pure sort.
3. `calculateStressTest(...)` → [[08-stress-test]] single-point.

## Bank registry — single source

`features/budget/utils/bank-registry.ts` — 5 banks (VIB, VCB, VPBank, BIDV, ACB).

> **Strategy note**: Techcombank cố ý loại trừ. Neutrality moat positions against OneHousing × Techcombank vertical-integration play. See cascade execution brief §4.

Each entry:

```ts
{
  id, bankName, packageName,
  preferentialInterestRate, preferentialYears,
  floatingInterestRate, maxLoanTerm,
  prepaymentPenalty, processingFee, appraisalFee,
  insuranceRequired, limitation,
  eligibility: { maxDTI, maxLTV }  // legacy, dùng cho bank-overlay
}
```

`BANK_PACKAGES` view = drops `eligibility` → consumed bởi loan-comparison.

## Pipeline visual

```mermaid
flowchart TD
  IN([loanAmount + loanTermYears]) --> FILTER{maxLoanTerm ≥ term?}
  FILTER -->|some pass| ELIG[eligiblePackages]
  FILTER -->|none pass| FB[Fallback: sort by maxLoanTerm desc]
  FB --> ELIG

  ELIG --> CALC[For each pkg:<br/>monthlyPref = PMT pref<br/>monthlyFloat = realistic<br/>totalInterest = 2-phase<br/>totalCost = interest + fees]

  CALC --> SORT[Sort by totalCost asc]
  SORT --> EXTR[Find extremes:<br/>lowestCost, lowestPrefRate,<br/>longestPrefYears]

  EXTR --> BADGE[Assign badges<br/>no collision]
  BADGE --> SAV[monthlySavingsVsWorst]
  SAV --> OUT([LoanPackage[] sorted])

  style ELIG fill:#E6F5EC
  style OUT fill:#139A51,color:#fff
```

## Computation pipeline

```
INPUT  loanAmount, loanTermYears, packages
  │
  ├─ 1. Filter eligible:
  │      packages where pkg.maxLoanTerm ≥ loanTermYears
  │      (fallback: nếu empty → sort by maxLoanTerm desc)
  │
  ├─ 2. For each eligible pkg:
  │      monthlyPreferential = PMT(loan, prefRate, term)
  │      monthlyFloating     = calculateRealisticFloatingPayment(...)
  │      totalInterest       = calculateTotalInterest(...) — 2-phase annuity
  │      fees                = processingFee + appraisalFee
  │      totalCost           = totalInterest + fees
  │
  ├─ 3. Sort by totalCost ascending
  │
  ├─ 4. Find extremes:
  │      lowestCost  = computed[0]
  │      highestCost = computed[last]
  │      lowestPrefRate    = sort by preferentialInterestRate asc → [0]
  │      longestPrefYears  = sort by preferentialYears desc → [0]
  │
  ├─ 5. monthlySavingsVsWorst = worstMonthlyFloating − pkg.monthlyFloating
  │
  └─ 6. Assign badges (no collision):
       lowestCost.id        → "Chi phí thấp nhất" / lowest-cost
       lowestPrefRate.id    → "Lãi ưu đãi tốt nhất" / best-rate
       longestPrefYears.id  → "Ưu đãi lâu nhất" / most-suitable
       else                 → "Phù hợp" / most-suitable
       (track assigned IDs — mỗi pkg chỉ 1 badge)
       │
       ▼
OUTPUT  LoanPackage[]  (sorted by totalCost asc, with computed + badges)
```

## Fair comparison principle

> "Comparing packages with different effective terms is financially invalid because shorter terms have lower total interest but much higher monthly payments."

→ Tất cả gói **bắt buộc cùng** `effectiveTerm = loanTermYears` (user's term). Lọc gói nào không support nổi term đó.

Fallback khi tất cả gói đều quá ngắn: sort by `maxLoanTerm` desc → dùng best available. Không silent drop.

## Buyer-selectable sort (K3 Z1.5)

```ts
LOAN_SORT_OPTIONS = [
  { key: "totalCost",  label: "Chi phí thấp nhất" },
  { key: "rate",       label: "Lãi suất thấp nhất" },
  { key: "monthly",    label: "Trả góp thấp nhất" },
]
```

> **Published ranking parameters**: every sort key is an OBJECTIVE cost dimension computed identically for every bank — no paid placement, no FinHome-favoured ordering. The buyer chooses how their options are ranked.

```ts
sortLoanPackages(packages, key) → LoanPackage[]  // pure, returns COPY ascending
```

Metric per key:
- `totalCost` → `p.totalCost ?? 0`
- `rate` → `p.preferentialInterestRate`
- `monthly` → `p.monthlyFloating` (KHÔNG `monthlyPreferential`)

### Tại sao `monthly` rank theo floating, không preferential?

> "Mọi package đều dùng cùng loan amount + term, nên preferential monthly là strict function of preferential rate. Ranking theo nó = no-op vs rank theo rate."

Floating monthly **phụ thuộc vào floating rate của bank** → distinct, honest ordering "lowest installment after teaser period".

## Repayment schedule generation

`generateRepaymentSchedule(loanAmount, prefRate, prefYears, floatRate, term)` → 1 point/year cho chart.

Logic:
- Năm 0 → prefYears-1: flat line tại `monthlyPref`.
- Năm prefYears (transition): emit 2 entries — pref entry + float entry (vertical step).
- Năm prefYears+1 → term: rising line dùng `FLOAT_RATE_DRIFT = 0.3%/year` simulation.
  - `currentRate = floatingRate + yearsIntoFloat × 0.3`
  - `balanceNow` = `calculateRemainingBalance(balanceAtPrefEnd, floatingRate, ...)`
  - `payment = PMT(balanceNow, currentRate, remainingYears)`

`FLOAT_RATE_DRIFT = 0.3` — conservative annual increase từ VN market avg. Mục đích **không** predict thực tế — illustrate trend "ngay cả khi rate ổn định, payment vẫn drift do recalculation".

## 4 badge types

| Type             | Logic                                     | Display              |
| ---------------- | ----------------------------------------- | -------------------- |
| `lowest-cost`    | Min `totalCost`                           | "Chi phí thấp nhất"  |
| `best-rate`      | Min `preferentialInterestRate`            | "Lãi ưu đãi tốt nhất"|
| `most-suitable`  | Max `preferentialYears` HOẶC fallback     | "Ưu đãi lâu nhất" / "Phù hợp" |

Collision rule: 1 pkg = 1 badge. Priority: `lowest-cost` > `best-rate` > `most-suitable`.

## Removed (history) — `/loan-leads`

```ts
// ── Loan-lead (PII-push) API removed (strategy phase-02 / disqualifier #3).
// The buyer-controlled replacement lives in features/handoff (decision-record
// export). FinHome must never push buyer PII to lenders — do not reintroduce a
// POST to /loan-leads (the CI route-audit guard enforces this).
```

→ Buyer-controlled handoff replaces lead-push. `/consultation-registration` route mounts `DecisionRecordScreen` từ `features/handoff` (không phải budget).

## API option — `getLoanPackages()`

`features/budget/services/budget-api.ts:141`. Fetch live data từ `loanApi.get('/loan-packages')`. Static `BANK_REGISTRY` là default, API là override (production roadmap).

Hook: `useLoanPackagesQuery()` — 10min staleTime, 30min gcTime.

→ [[02-data-contracts]] LoanPackage type · [[08-stress-test]]
