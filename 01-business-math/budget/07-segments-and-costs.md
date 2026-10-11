# 07 — Housing segments & hidden costs

2 module phụ trợ output. Cùng pattern: pure function, data-driven, Vietnamese-localized.

---

## A. Housing segments

`generateSuitableSegments(maxPrice)` ở `budget-calculator.ts:490`.

### 4 phân khúc cố định

| Segment      | Tên                  | Range (VND)         | Color    |
| ------------ | -------------------- | ------------------- | -------- |
| `social`     | Nhà ở xã hội         | 500M – 1.5 tỷ       | `#139a51` |
| `suburban`   | Căn hộ vùng ven      | 1.5 tỷ – 2.5 tỷ     | `#0284C7` |
| `mid-range`  | Căn hộ trung cấp     | 2.5 tỷ – 5 tỷ       | `#FFB300` |
| `high-end`   | Căn hộ cao cấp       | ≥ 5 tỷ              | `#139a51` (suitable) / `#F44336` (not) |

### Status logic — per segment

Mỗi segment có 3 status: `suitable` / `consider` / `not-suitable`.

**Social** (500M – 1.5tỷ):
- `maxPrice ≥ 500M` → `suitable` ("Phù hợp")
- else → `not-suitable` ("Chưa đạt")

**Suburban** (1.5 – 2.5tỷ):
- `1.5tỷ ≤ maxPrice ≤ 2.5tỷ` → `suitable` ("Phù hợp nhất")
- `maxPrice > 2.5tỷ` → `suitable` ("Phù hợp" — over-qualified)
- else → `consider` ("Cân nhắc kỹ")

**Mid-range** (2.5 – 5tỷ): same pattern with thresholds 2.5/5.

**High-end** (≥5tỷ):
- `maxPrice ≥ 5tỷ` → `suitable` ("Phù hợp")
- else → `not-suitable` ("Chưa phù hợp")

### `currentSegment`

`determineSuitableSegment(maxPrice)`:
- < 1.5tỷ → `"social"`
- < 2.5tỷ → `"suburban"`
- < 5tỷ → `"mid-range"`
- else → `"high-end"`

Đây là 1 trong 4 segments mà user "đang ở" — UI highlight nó.

### UI — `PriceSpectrum` component

Renders 4 segments dạng horizontal slider segments + marker tại vị trí `maxPrice`. User thấy ngay "tôi đang ở suburban, cách mid-range 600M nữa".

### Tại sao thresholds cứng?

VN real estate 2025-2026 — 4 mức này phản ánh sản phẩm thị trường thực:
- < 1.5tỷ = nhà ở xã hội (Hà Nội/HCMC ngoại ô + chính sách)
- 1.5-2.5tỷ = chung cư bình dân vùng ven
- 2.5-5tỷ = chung cư trung cấp nội đô
- ≥ 5tỷ = cao cấp / căn hộ rộng / nhà phố

Sẽ cần cập nhật khi market drift. **Single source**: chỉ ở `budget-calculator.ts` — đừng duplicate.

---

## B. Hidden costs

`calculateHiddenCosts(propertyPrice, monthlyInstallment, safeLoanAmount)` ở `budget-calculator.ts:305`.

### Always-included (3 items)

Áp dụng cho **cả** cash-only và loan purchases:

| Item                                       | Công thức                         |
| ------------------------------------------ | --------------------------------- |
| Thuế chuyển nhượng (2%)                    | `price × 0.02`                    |
| Phí đăng bộ / trước bạ (0.5%)              | `price × 0.005`                   |
| Phí công chứng hợp đồng                    | `price × 0.001 + 2_000_000`       |

Plus một flat:
| Item                  | Amount         |
| --------------------- | -------------- |
| Chi phí chuyển nhà    | `5_000_000` (5 triệu) |

### Loan-only (4 extra items)

Khi `safeLoanAmount > 0`:

| Item                          | Công thức                         |
| ----------------------------- | --------------------------------- |
| Phí thẩm định tài sản         | `3_000_000` (3 triệu — bank fee)  |
| Bảo hiểm cháy nổ (bắt buộc)   | `price × 0.001` (~0.1%)           |
| Quỹ dự phòng (6 tháng góp)    | `monthlyInstallment × 6`          |

Tổng = sum tất cả items. UI hiển thị từng item với icon + amount + name.

### Tại sao loại bỏ khi cash-only?

- Phí thẩm định = bank chỉ yêu cầu khi có vay.
- Bảo hiểm cháy nổ = bank đòi khi tài sản đang là collateral.
- Quỹ dự phòng = 6 tháng "lương" trả góp. Nếu installment = 0 thì quỹ = 0 — không hợp lý hiển thị.

### Items shape

```ts
interface HiddenCostItem {
  id: string;
  name: string;          // Tiếng Việt + % nếu liên quan
  amount: number;        // VND
  icon?: IconSymbolName; // SF Symbols name
}
```

Icons từ SF Symbols (iOS-native). Code hiện hard-coded:
- `doc.text.fill` (thuế)
- `list.bullet.clipboard.fill` (phí đăng bộ)
- `hammer.fill` (công chứng)
- `building.columns.fill` (thẩm định)
- `flame.fill` (cháy nổ)
- `banknote.fill` (quỹ dự phòng)
- `shippingbox.fill` (chuyển nhà)

### Hidden cost rationale

Vietnamese buyer thường **bỏ qua** mấy cost này khi tính ngân sách. Hiển thị chúng → user thấy "tôi cần thêm X triệu ngoài giá nhà" → quyết định realistic hơn.

Tổng thường ~3-5% giá nhà (cash-only) hoặc 4-7% (có vay) — không trivial.

→ [[04-budget-engine]] step 15-16
