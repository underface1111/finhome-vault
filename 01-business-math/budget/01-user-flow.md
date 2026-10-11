# 01 — User flow

## Screen flow visual

Visual file: [[diagrams/user-flow.excalidraw|🎨 user-flow (Excalidraw)]]

```mermaid
flowchart LR
  TOOL([Tab Công cụ]) --> BS[BudgetScreen<br/>4 sliders + DTI picker]
  ONB([Onboarding]) --> BO[BudgetOnboardingScreen]
  COMP([Compass step 3]) --> BS

  BO --> BS
  BS -->|estimateBudgetPreview live| BS
  BS -->|Tính ngay| BR[BudgetResultScreen<br/>overview + PITIA + stress + segments]
  BR -->|Bookmark| SAVE[(savedResults[]<br/>cap 20)]
  BR -->|Xem gói vay| LP[LoanPackagesScreen<br/>sort by cost/rate/monthly]
  LP -->|Tap card| LPD[LoanPackageDetailScreen<br/>repayment schedule]
  BR -->|consultation| HND[DecisionRecord<br/>handoff — buyer-controlled]

  style BS fill:#E6F5EC
  style BR fill:#139A51,color:#fff
  style LP fill:#0284C7,color:#fff
```

## Entry points

User vào budget feature qua 3 ngả:

1. **Tab "Công cụ" → "Bạn mua được nhà giá bao nhiêu?"** (screenshot trong PRD)
2. **Onboarding lần đầu** → `BudgetOnboardingScreen` (route `/budget-intro`)
3. **La bàn mua nhà (Compass)** wizard step 3 pre-fills input rồi tính budget

## Routes

Tất cả route ở `app/(budget)/` đều là **thin wrappers** — re-export từ `@/features/budget`.

| Route                          | Screen                       | Vai trò                              |
| ------------------------------ | ---------------------------- | ------------------------------------ |
| `/budget`                      | `BudgetScreen`               | Form input (sliders)                 |
| `/budget-intro`                | `BudgetOnboardingScreen`     | First-time onboarding                |
| `/budget-result`               | `BudgetResultScreen`         | Output dashboard                     |
| `/loan-packages`               | `LoanPackagesScreen`         | So sánh + sort gói vay               |
| `/loan-package-detail`         | `LoanPackageDetailScreen`    | Detail 1 gói (1368 LOC — large)      |
| `/consultation-registration`   | `DecisionRecordScreen`       | Buyer-controlled handoff (non-PII)   |

## Happy path

```
BudgetScreen (input)
  ├─ 4 sliders: thu nhập / chi phí / nợ / vốn tự có
  ├─ "Mức trả nợ an toàn" picker (36% / 55% / custom)
  ├─ Loan structure: lãi suất + kỳ hạn (default 10.5%/20y)
  ├─ Live preview (estimateBudgetPreview) — cập nhật mỗi keystroke
  └─ CTA "Tính ngay"
      ↓ calculateBudget() — pure function, instant
      ↓ store result vào Zustand
      ↓ router.push('/budget-result')

BudgetResultScreen (output)
  ├─ Overview: maxAffordablePrice + safeLoan + monthlyInstallment
  ├─ DTI section (4-tier: ≤30 / ≤36 / ≤43 / >43)
  ├─ PITIA breakdown (gốc+lãi / thuế / bảo hiểm / quản lý)
  ├─ Cash flow card
  ├─ Hidden costs (3-7 items tuỳ có vay hay không)
  ├─ Price spectrum (4 phân khúc social/suburban/mid/high)
  ├─ Rate stress inline (+1/+2/+3pp)
  ├─ What-if scenarios
  ├─ Perspective comparison (compare 36 vs 55 vs custom)
  └─ CTA "Xem gói vay phù hợp"
      ↓ router.push('/loan-packages')

LoanPackagesScreen
  ├─ Sort chip: chi phí / lãi suất / trả góp ←  [[09-loan-comparison|3 published rankings]]
  ├─ List card: badge + bank + rates + monthly + totalCost
  └─ Tap card → /loan-package-detail
```

## State persistence

`budget-storage` (AsyncStorage, user-scoped):
- `budgetInput` — pre-fill khi quay lại
- `budgetResult` — last result
- `savedResults` — up to 20, displayable ở tab "Đã lưu"
- `dtiMode` + `customDTI` — perspective đã chọn

Save flow: tap bookmark trên BudgetResultScreen → `toggleSaveCurrentResult()` → toast.

## Edge cases trong flow

- **Custom DTI ≥ 60%** → Alert confirm bắt buộc trước khi tính (compliance — AI Law 134/2025).
- **Slider out of range** → `safeSetX()` clamp về `[MIN, MAX]`.
- **No saved input** → pre-fill bằng defaults (70M thu nhập, 15M chi phí, 0 nợ, 1 tỷ vốn).
- **Refresh button** → Alert confirm rồi reset toàn bộ về default.

## Analytics events

PII-minimized (PDPL): raw VND được bucket trước khi gửi PostHog. Xem `bucketIncomeOrExpenses`, `bucketMonthlyDebt`, `bucketDownPayment` trong `@/features/shared`.

| Event                    | When                              |
| ------------------------ | --------------------------------- |
| `calculate_budget`       | Mỗi lần tính                      |
| `first_budget_calculated`| 1 lần / device (activation NEW5)  |
| `activated`              | 1 lần / device                    |
| `perspective_selected`   | Đổi DTI mode                      |
| `risk_ack_shown/resolved`| Custom DTI vùng nguy hiểm         |

→ [[02-data-contracts]] · [[04-budget-engine]]
