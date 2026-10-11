# 💰 Budget — "Bạn mua được nhà giá bao nhiêu?"

Tool tính toán **giá nhà tối đa người dùng có thể mua được** dựa trên thu nhập, chi phí, nợ hiện tại, vốn tự có. Đầu ra: giá trần, khoản vay an toàn, trả góp/tháng, phân khúc phù hợp, stress test, gói vay so sánh.

## Câu hỏi feature trả lời

1. **Tôi mua được nhà giá bao nhiêu?** — `maxAffordablePrice`
2. **Tôi nên vay bao nhiêu?** — `safeLoanAmount`
3. **Trả góp/tháng là bao nhiêu?** — `monthlyInstallment` (worst-case = lãi thả nổi)
4. **Yếu tố nào giới hạn tôi?** — `limitingFactor` (thu nhập hay vốn tự có)
5. **Nếu lãi tăng +3% thì sao?** — `stressTest`
6. **Phân khúc nào phù hợp?** — `suitableSegments`
7. **Gói vay nào tối ưu?** — màn LoanPackages (sort theo cost / rate / monthly)

## Triết lý design

- **Client-first**: toàn bộ logic là pure function, chạy offline, không cần backend. API hooks tồn tại để sync sau khi user save.
- **Worst-case sizing**: dùng lãi **thả nổi** (không phải ưu đãi) để cap khoản vay → tránh sốc khi hết ưu đãi.
- **Dual constraint**: vay tối đa = `min(income-cap, LTV-cap)`. Cả 2 đều phải pass.
- **Refusal-safe**: không có bank_id trong type cốt lõi. Bank list là static registry, comparable nhưng không "preferred".
- **Transparency**: giải thích **why** (limiting factor, insight tier, hidden costs breakdown), không chỉ một con số.

## Index — đọc theo thứ tự

1. [[01-user-flow]] — màn hình, route, user journey
2. [[02-data-contracts]] — `BudgetInput`, `BudgetResult`, constants
3. [[03-core-math]] — PMT, PV, remaining balance, two-phase
4. [[04-budget-engine]] — `calculateBudget()` pipeline đầu-cuối
5. [[05-dti-perspectives]] — "Mức trả nợ an toàn" 36/55/custom
6. [[06-insights-and-limits]] — 4-tier insight + limiting factor + edge cases
7. [[07-segments-and-costs]] — housing segments + hidden costs
8. [[08-stress-test]] — +3pp foresight cho compass-readiness
9. [[09-loan-comparison]] — so sánh gói vay ngân hàng
10. [[10-acm-engine]] — PITIA / DTI front+back / refusal posture
11. [[11-state-and-api]] — Zustand store + React Query + endpoints
12. [[12-tests-as-spec]] — 561 tests, verified invariants

## Visual diagrams

Mỗi note có Mermaid inline (render trực tiếp trong Obsidian). Excalidraw files cho heavyweight views:

- 🎨 [[diagrams/budget-pipeline.excalidraw|budget-pipeline]] — calculateBudget end-to-end
- 🎨 [[diagrams/dual-constraint.excalidraw|dual-constraint]] — `min(loanFromIncome, loanFromLTV)`
- 🎨 [[diagrams/insight-decision-tree.excalidraw|insight-decision-tree]] — 4 tiers + 3 special cases
- 🎨 [[diagrams/user-flow.excalidraw|user-flow]] — screen journey

Cần Obsidian Excalidraw plugin để edit. Nếu không có plugin, mở `.excalidraw` file ở excalidraw.com.

## Key files trong code

| Vai trò                | File                                                  |
| ---------------------- | ----------------------------------------------------- |
| Core engine            | `features/budget/utils/budget-calculator.ts` (816 LOC) |
| Loan math (PMT/PV)     | `features/budget/utils/loan-math.ts`                  |
| 2-phase loan compare   | `features/budget/utils/loan-comparison.ts`            |
| ACM PITIA engine       | `features/budget/utils/pitia-calculator.ts`           |
| Rate-stress engine     | `features/budget/utils/rate-stress.ts`                |
| Bank registry          | `features/budget/utils/bank-registry.ts`              |
| Types                  | `features/budget/types/{budget,acm}.types.ts`         |
| Store                  | `features/budget/stores/budget-store.ts`              |
| API                    | `features/budget/services/{budget,acm}-api.ts`        |
| Main input UI          | `features/budget/screens/BudgetScreen.tsx`            |
| Result UI              | `features/budget/screens/BudgetResultScreen.tsx`      |
| Loan packages UI       | `features/budget/screens/LoanPackagesScreen.tsx`      |
| Routes                 | `app/(budget)/*.tsx` — thin wrappers                  |
