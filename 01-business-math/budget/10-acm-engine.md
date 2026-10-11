# 10 — ACM engine (Affordability & Capacity Model)

Engine song song với budget-calculator. Mục đích: surface **PITIA breakdown** + **rate-stress curve** + **assumptions snapshot** trên BudgetResultScreen. Khác biệt key: refusal-safe ở **type level**.

## Vì sao có ACM riêng?

`calculateBudget()` (W1 logic) trả 1 mức vay an toàn + 1 stressTest điểm. Đủ cho budget answer.

ACM (W2 logic) thêm:
- **PITIA itemized** (P+I, thuế, bảo hiểm, HOA riêng từng dòng).
- **3-point stress curve** (+1/+2/+3pp) thay vì 1 điểm.
- **AssumptionsSnapshot** — CDR (decision-record handoff) embed VERBATIM.
- **TargetPriceBand** — band + display-only midpoint, không bao giờ parcel point.

Architecture decision (A2): CDR (`features/handoff`) export VERBATIM full ACM snapshot — không lossy subset. Buyer carry decision record của họ, FinHome route nothing.

## Modules

| File                                          | Purpose                              |
| --------------------------------------------- | ------------------------------------ |
| `types/acm.types.ts`                          | Refusal-safe types                   |
| `utils/pitia-calculator.ts`                   | computePitia / computeDti / computeLtv |
| `utils/rate-stress.ts`                        | buildRateStressPoints (3-point curve) |
| `utils/assumptions-snapshot.ts`               | buildAssumptionsSnapshot              |
| `services/acm-api.ts`                         | computeAffordability (LOCAL stub)    |
| `hooks/use-acm-query.ts`                      | useAffordability mutation            |

## `ACM_DEFAULTS` — bank-agnostic VN defaults

```ts
{
  preferentialRate: 7.5,      // %/năm (KHÁC budget engine 8.5)
  preferentialYears: 2,
  floatingRate: 11,           // KHÁC budget engine 12
  loanTerm: 20,
  propertyTaxRate: 0.03,      // %/năm trên propertyValue
  insuranceRate: 0.55,        // %/năm trên loanPrincipal
  hoaPerSqmMonthly: 12_000,   // VND/m²/tháng (chung cư)
  targetDti: 0.45,            // 45% — RATIO 0-1
  ltvCap: 0.7,                // 70% — RATIO 0-1
}
```

Khác defaults với budget engine — ACM dùng số liệu "trung lập thị trường" thay vì worst-case caps. Budget engine bi quan hơn (12% floating) cho safe sizing; ACM honest hơn (11%) cho display.

## PITIA breakdown

```ts
PITIA.total = principalInterest + propertyTax + insurance + hoaOrMgmt
```

Trong đó:
- **principalInterest** = `monthlyAnnuityPayment(loanPrincipal, preferentialRate, loanTerm)`
  - Dùng `preferentialRate` (baseline) — rate-stress xử lý kịch bản thả nổi riêng.
- **propertyTax** = `(propertyValue × propertyTaxRate% / 100) / 12`
  - Thuế nhà đất phi nông nghiệp VN ~ 0.03%/năm.
- **insurance** = `(loanPrincipal × insuranceRate% / 100) / 12`
  - Bảo hiểm khoản vay ~ 0.55%/năm trên dư nợ gốc.
  - **Lưu ý approximation**: insurance tính trên `loanPrincipal` ban đầu (không decline) → over-estimate sang phase 2. Acceptable cho "ước tính khả năng chi trả".
- **hoaOrMgmt** = `areaSqm × hoaPerSqmMonthly` (chỉ áp dụng chung cư; nhà phố `areaSqm=undefined`)

## DTI front + back

ACM differentiate 2 DTI:

```ts
computeDti(housingPayment, otherMonthlyDebts, monthlyIncome) {
  dtiFront = housingPayment / income           // RATIO 0-1
  dtiBack  = (housingPayment + otherDebts) / income
}
```

- **`dtiFront`** = housing burden alone (chỉ PITIA).
- **`dtiBack`** = total debt burden (PITIA + nợ khác).

`income <= 0` → return `{0, 0}` (no NaN).

US/VN convention: front-end ≤ 28%, back-end ≤ 36% (budget engine threshold). Bank VN duyệt tới ~43-50% back-end.

## `computeAffordability()` — orchestrator

```ts
async computeAffordability(input: AffordabilityInput): AffordabilityState
```

LOCAL stub (file `services/acm-api.ts`). Backend TODO:
```
POST /fds/acm/affordability
→ returns AffordabilityState superset + assumptionsSnapshot
→ server rejects bank_id / bare price point → 400
→ no CIC pull
```

Local logic:
```
1. propertyValue = input.targetPrice.midpoint
2. loanPrincipal = max(0, propertyValue − savings)
3. pitiaBreakdown = computePitia({ loanPrincipal, propertyValue, ...rates })
4. nonPiHousingMonthly = pitia.total − pitia.principalInterest
5. { dtiFront, dtiBack } = computeDti(pitia.total, debts, income)
6. ltv = computeLtv(loanPrincipal, propertyValue)
7. stressResults = buildRateStressPoints({ loanPrincipal, baseFloatingRate, term, income, debts, nonPiHousingMonthly })
8. return AffordabilityState
```

> `midpoint` được dùng ONLY làm working value cho engine; surface UI luôn render band.

## Refusal posture — type-level enforcement

> "Refusal at the type level: NO bank_id / lender_name / counterparty_id anywhere."

```ts
// ❌ Would not type-check:
interface AffordabilityInput {
  bank_id?: string;          // FORBIDDEN
  preferredLender?: string;  // FORBIDDEN
}
```

Reason: ACM engine assumptions = **market-rate**, không gắn 1 lender cụ thể. Nếu cần thông tin về bank, route qua bank-overlay (flag-gated, separate feature `loan-fit`).

## `TargetPriceBand` — band, never point

```ts
interface TargetPriceBand extends ConfidenceBand {
  midpoint: number;  // display-only, rendered LIGHTER than band, never parcel point
}
```

`ConfidenceBand` ở `@/features/shared` carries low/high + granularity ("city" | "district" | "ward"). Midpoint là **single number** cho engine math, nhưng UI rule: render band first, midpoint chỉ là dim marker.

Refusal at this layer: bare price point (vd `targetPrice: 3_000_000_000` thuần) sẽ fail type check — phải pass band shape.

## `AssumptionsSnapshot` — verbatim CDR embed

```ts
{
  rateFeedVersion, preferentialRate, preferentialYears,
  floatingRate, loanTerm, targetDti, ltvCap,
  granularity, computedAt: ISO_timestamp
}
```

`buildAssumptionsSnapshot()` fills defaults từ `ACM_DEFAULTS` cho fields không truyền. `computedAt` accept param → keep fn testable.

Architecture decision A2: CDR embeds **full shape**, không subset. → User có decision record có thể audit "tôi quyết định mua dựa trên assumptions gì". Lender đối tác có thể verify rate feed version.

## `RateStressPoint[]` — always 3

`buildRateStressPoints(input)` returns deterministic 3 points cho `[1, 2, 3]pp`. Stable layout cho UI (always 3 rows, không vary).

Xem [[08-stress-test]] for math.

## Hook surface

```ts
useAffordability(): useMutation<AffordabilityState, Error, AffordabilityInput>
```

Mutation pattern (not query) vì input có thể không stable enough cho query key. Trigger: user submit ACM form → mutate → state update.

→ [[02-data-contracts]] ACM types section · [[08-stress-test]]
