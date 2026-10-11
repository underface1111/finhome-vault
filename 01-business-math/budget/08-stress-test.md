# 08 — Stress test

"Nếu lãi thả nổi tăng +N% thì sao?" — foresight tool, key cho **Compass readiness** (la bàn mua nhà).

## 2 engines song song

Code có 2 stress test với mục đích khác nhau:

### 1. `calculateStressTest()` — Budget single-point

File: `loan-comparison.ts:309`. Trả về **1 điểm** cho `stressIncrease` cho trước.

```ts
calculateStressTest(
  loanAmount, floatingRate, stressIncrease, term, netIncome, existingDebt
): {
  stressedRate,         // = floatingRate + stressIncrease
  stressedMonthly,      // PMT mới
  normalMonthly,        // PMT base
  stressedDTI,          // (newPI + existingDebt) / income × 100
  increase              // = stressedMonthly − normalMonthly
}
```

Dùng trong `calculateBudget()`: hard-coded `stressIncrease = 3` để emit `stressTest` field trong BudgetResult — compass dùng nó.

Format DTI ở đây = **percentage 0-100** (consistency với `loanAssumptions.dtiRatio`).

### 2. `buildRateStressPoints()` — ACM 3-point curve

File: `rate-stress.ts:42`. Trả về **3 điểm** `[+1, +2, +3]pp`.

```ts
buildRateStressPoints(input): RateStressPoint[]   // luôn 3 phần tử
```

Mỗi `RateStressPoint`:

```ts
{
  deltaPp: 1 | 2 | 3,
  stressedRate,         // base + deltaPp
  stressedMonthly,      // PI + nonPiHousingMonthly
  stressedDtiBack,      // RATIO 0-1 (không phải %)
  breaches              // boolean — stressedDtiBack > 0.5
}
```

Format DTI ở đây = **ratio 0-1** (consistency với `computeDti()` ACM convention).

`DTI_BREACH_CEILING = 0.5` — vượt qua coi là "vỡ kế hoạch".

## Tại sao 3 điểm?

UX research: user không hiểu "lãi tăng 1%" có nghĩa gì. Show 3 mức:
- **+1pp** → "tăng nhẹ, vẫn ổn"
- **+2pp** → "tăng vừa, cần xem lại"
- **+3pp** → "tăng mạnh, có thể default"

Visual: render 3 card / chart points. Card nào `breaches=true` → red warning.

Lý do **floating rate**, không phải preferential: ưu đãi 2 năm rồi hết. Risk thực sự là sau ưu đãi → cap đúng cái user sẽ sợ.

## Input shapes — phân biệt

**Budget single-point** (`calculateStressTest`):
- Không nhận `nonPiHousingMonthly` — chỉ tính PI vs DTI.
- DTI output dạng %.

**ACM 3-point** (`buildRateStressPoints`):
- Nhận `nonPiHousingMonthly` (thuế + bảo hiểm + HOA) → `stressedHousing = stressedPi + nonPi`.
- DTI output dạng ratio.

Khác biệt vì ACM thấy gánh nặng nhà ở **đầy đủ** (PITIA), không chỉ gốc+lãi.

## Math

```
stressedRate = baseRate + deltaPp
stressedPi = PMT(principal, stressedRate, term)   // same as core math
```

PMT recalculated lại với rate mới — giữ nguyên principal + term. → Trả góp tăng theo curve hơi convex.

Ví dụ: principal 2 tỷ, 20 năm, base 12% → PMT ~22M.
- +1pp (13%) → PMT ~23.4M (+6.4%)
- +2pp (14%) → PMT ~24.9M (+13.2%)
- +3pp (15%) → PMT ~26.3M (+19.5%)

→ Lãi tăng 25% (3/12), trả tăng 19.5% — không tỷ lệ tuyến tính. Loan dài hạn nhạy cảm hơn.

## Where it's surfaced

1. **BudgetResult.stressTest** — 1 dòng +3pp trong overview card.
2. **RateStressInline** component (BudgetResultScreen) — 3 dòng chi tiết từ `buildRateStressPoints()`.
3. **Compass step 4** — readiness check dùng `BudgetResult.stressTest.stressedDTI > 50` để gate next step.

## Edge cases

- `safeLoan === 0` → engine không emit `stressTest`. Component skip render.
- `netIncome === 0` → DTI = 0 (computeDti guards). Stress points vẫn build nhưng `breaches=false`.
- `baseFloatingRate` quá cao (vd 20%) → +3pp = 23%. Math vẫn chạy, UI hiển thị bình thường (no clamp).

## Refusal-safe

Cả 2 engines = **bank-agnostic**. Không có `bank_id`. Stress test trên `baseFloatingRate` (market rate), không phải lender-specific rate.

→ [[03-core-math]] section 8 · [[10-acm-engine]]
