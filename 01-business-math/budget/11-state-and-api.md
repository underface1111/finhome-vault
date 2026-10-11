# 11 — State & API layer

## Zustand store — `budget-store.ts`

```ts
useBudgetStore = create<BudgetStore>()(
  persist(
    (set, get) => ({ ... }),
    {
      name: "budget-storage",
      storage: userScopedStorage,    // ← KEY: user-scoped, không cross-account
      partialize: state => ({ savedResults, budgetResult, budgetInput, dtiMode, customDTI }),
      merge: migrationFn,
    }
  )
)
```

### State

| Field             | Type                  | Purpose                              |
| ----------------- | --------------------- | ------------------------------------ |
| `budgetInput`     | `BudgetInput \| null` | Last input — pre-fill khi quay lại   |
| `budgetResult`    | `BudgetResult \| null`| Last result                          |
| `savedResults`    | `SavedBudgetResult[]` | Up to 20, displayable tab "Đã lưu"   |
| `dtiMode`         | `"global"\|"asia"\|"custom"` | Default `"global"` (36%)      |
| `customDTI`       | `number`              | Default `36`                         |

### Actions

| Action                          | Behavior                              |
| ------------------------------- | ------------------------------------- |
| `setBudgetInput(input)`         | Pre-fill source                       |
| `setBudgetResult(result)`       | Sau khi calculateBudget()             |
| `setDtiMode(mode)`              | Đổi perspective                       |
| `setCustomDTI(pct)`             | Update slider custom                  |
| `clearBudgetResult()`           | Reset cả input + result               |
| `toggleSaveCurrentResult()`     | → boolean (true=saved, false=unsaved) |
| `removeSavedResult(id)`         | Xoá 1 entry                           |
| `clearAllSavedResults()`        | Xoá all                               |
| `undoRemoveSavedResult(entry, idx)` | Undo trong toast                  |

### Save dedup logic

`toggleSaveCurrentResult()` so sánh entry mới với existing dựa trên 3 fields:
```ts
saved.result.maxAffordablePrice === current.maxAffordablePrice
&& saved.result.safeLoanAmount === current.safeLoanAmount
&& saved.result.downPayment === current.downPayment
```

Match → remove (toggle off). No match → append, cap at `MAX_SAVED_RESULTS = 20` (shift oldest nếu đầy).

ID = `Date.now().toString()` — đủ unique cho local persistence (không cần UUID).

### User-scoped storage

```ts
import { userScopedStorage } from "@/features/shared/services/user-storage";
```

Keys scoped theo user ID. Logout/switch account → storage keys khác → data không leak giữa accounts.

### Migration (merge fn)

Saved results cũ thiếu `input` field (added later). Migration:
```ts
merge: (persisted, current) => {
  if (stored.savedResults) {
    stored.savedResults = stored.savedResults.map(s => ({
      ...s,
      input: s.input ?? { netIncome: 0, monthlyDebt: 0, downPayment: s.result.downPayment }
    }));
  }
  return { ...current, ...stored };
}
```

→ Backward compat không cần force-migrate, chỉ heal at read time.

## Selectors pattern

Tất cả consumers PHẢI dùng selector (single field) để tránh unnecessary re-render:

```ts
// ✅ GOOD
const budgetResult = useBudgetStore(s => s.budgetResult);

// ❌ BAD — re-renders mỗi action
const { budgetResult, savedResults } = useBudgetStore();
```

Trong `BudgetResultScreen`: derived state qua selector functions (`isSaved` selector evaluates `.some()` mỗi render — ok vì state shallow change).

## API layer

### Endpoints — `budget-api.ts`

```ts
API_ENDPOINTS.budget = {
  calculate: 'POST /budget/calculate',  // server-side mirror (chưa dùng production)
  save:      'POST /budget/save',
  list:      'GET  /budget/list',
}

loanApi.get('/loan-packages')           // separate API client cho loan service
```

### React Query hooks

| Hook                       | Type       | Purpose                              |
| -------------------------- | ---------- | ------------------------------------ |
| `useCalculateBudget()`     | Mutation   | POST `/budget/calculate`             |
| `useSaveBudget()`          | Mutation   | POST `/budget/save` + invalidate list|
| `useListBudgets(params?)`  | Query      | GET `/budget/list` (2min stale)      |
| `useLoanPackagesQuery()`   | Query      | `loanApi.get('/loan-packages')` (10min stale, 30min gc) |

### Why `calculateBudget` mutation EXISTS but UI bypasses it

UI flow:
```
BudgetScreen "Tính ngay" → calculateBudget(input) ← PURE FN  → setBudgetResult()
                          (utils/budget-calculator.ts)
```

KHÔNG gọi `useCalculateBudget()`. Server endpoint:
- Reserved cho server-side mirror (cross-device sync).
- Logged for analytics / fraud detection (chưa enabled).
- Backup tính toán nếu cần off-device computation.

→ Hiện UI hoạt động hoàn toàn offline. Network cần chỉ cho save + list (server-side persistence) + loan-packages (catalog).

### `getLoanPackages()` response shape

```ts
ApiLoanPackage {
  id, bankId, bankName, bankLogo, packageName, statusBadge,
  preferentialInterestRate, preferentialYears, floatingInterestRate,
  maxLoanTerm,  // ← MONTHS (360 = 30 năm), KHÁC static registry dùng YEARS
  maxLtv, prepaymentPenalty, processingFee, appraisalFee,
  insuranceRequired, limitation,
}
```

> **Unit mismatch alert**: API `maxLoanTerm` dùng **months**, static `BANK_PACKAGES` dùng **years**. Khi swap source cần convert.

### Removed — `/loan-leads`

> Strategy phase-02 / disqualifier #3. CI route-audit guard enforces no POST `/loan-leads`. Replacement: buyer-mint decision record (`features/handoff`).

## Provider chain & React Query config

```
app/_layout.tsx:
  Sentry → PostHogProvider → QueryProvider → AppProvider → ThemeProvider → Stack
```

`QueryProvider` ở `@/features/shared` — wraps `QueryClient` with defaults. Budget feature không override defaults.

## Error handling pattern

API services return **defaulted shape** thay vì throw:

```ts
export async function calculateBudget(data) {
  const response = await api.post(...);
  return response.data.data || {  // ← fallback shape
    maxAffordablePrice: 0,
    safeLoanAmount: 0,
    ...
  };
}
```

Trade-off: UI luôn có data hợp lệ (defensive) nhưng có thể silently mask errors. Mutations expose `error` qua React Query — UI mới có thể show toast.

## Loading states

Compute is **instant** (pure fn). UI fake loading 300ms cho UX:

```ts
setIsCalculating(true);
const result = calculateBudget(input);
setBudgetInput(input); setBudgetResult(result);
// ... posthog tracking ...
setTimeout(() => {
  setIsCalculating(false);
  router.push("/budget-result");
}, 300);
```

Loading state dùng cho button → "Tính ngay" → "Đang tính..." → push next screen. Nếu instant, user bị shock visual.

→ [[04-budget-engine]] · [[01-user-flow]] state persistence section
