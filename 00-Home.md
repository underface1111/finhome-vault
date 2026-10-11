# 🏠 FinHome Ecosystem — Knowledge Hub & Architecture

> **Single Source of Truth** cho toàn bộ hệ sinh thái FinHome (Vietnamese Fintech Mua Nhà).  
> Vault này quản lý **Business Logic (Toán, Rules, Edge Cases)**, **Kiến trúc liên nền tảng (Web, App, AI)** và **Hợp đồng dữ liệu (API & Design System)**.

---

## 📌 BẢN ĐỒ ĐIỀU HÀNH HỆ SINH THÁI (MOC)

```
                       ┌─────────────────────────┐
                       │   01 - Business & Math  │
                       │   (Pure Logic & Moat)   │
                       └────────────┬────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
│  Web Platform      │   │  Mobile Platform   │   │  AI & Operations   │
│  (Next.js Portal)  │   │  (React Native)    │   │  (Agentic Workflows│
│  [[Web-Docs/]]     │   │  [[App-Docs/]]     │   │  [[AI-Docs/]]      │
└────────────────────┘   └────────────────────┘   └────────────────────┘
```

---

## 1. 🧮 Cốt lõi Nghiệp vụ & Toán tài chính (`01-business-math/`)

Nơi định nghĩa các thuật toán, công thức tính toán và edge cases chuẩn xác của FinHome:

- **💰 Budget Engine ("Bạn mua được nhà giá bao nhiêu?")**:
  - [[01-business-math/budget/01-user-flow|01. User Flow]] — Trải nghiệm người dùng & các điểm chạm
  - [[01-business-math/budget/02-data-contracts|02. Data Contracts]] — Input/Output & Validation rules
  - [[01-business-math/budget/03-core-math|03. Core Math]] — Công thức tính toán DTI, LTV, EMI, lãi suất
  - [[01-business-math/budget/04-budget-engine|04. Budget Engine]] — Pipeline xử lý ngân sách 2 ràng buộc (Thu nhập vs Vốn tự có)
  - [[01-business-math/budget/05-dti-perspectives|05. DTI Perspectives]] — Góc nhìn an toàn vs thực tế ngân hàng VN
  - [[01-business-math/budget/06-insights-and-limits|06. Insights & Hard Limits]] — Cảnh báo rủi ro & giới hạn cho vay
  - [[01-business-math/budget/07-segments-and-costs|07. Segments & Hidden Costs]] — Chi phí phát sinh (thuế, phí trước bạ, bảo hiểm)
  - [[01-business-math/budget/08-stress-test|08. Stress Test]] — Kịch bản lãi suất tăng +2% đến +4%
  - [[01-business-math/budget/09-loan-comparison|09. Loan Comparison]] — So sánh gói vay (Dư nợ giảm dần vs Niên kim)
  - [[01-business-math/budget/10-acm-engine|10. ACM Engine]] — Affordability Calculation Model
  - [[01-business-math/budget/11-state-and-api|11. State & API]] — Tích hợp frontend state & backend API
  - [[01-business-math/budget/12-tests-as-spec|12. Tests as Spec]] — Test suites làm chứng minh chuẩn toán
  - [[01-business-math/budget/README|📄 Tổng quan Feature Budget]]

---

## 2. 📱 Nền tảng Nổi bật (Cross-Platform Docs)

Các thư mục tài liệu này được đồng bộ trực tiếp (`symlink`) với thư mục `docs/` của từng repo source code:

### 🌐 Web Portal (`finhome_website`) — [[02-platforms/Web-Docs/README|Mở Web Docs]]
- **Calculator Suite**: [[02-platforms/Web-Docs/calculator-suite-status|Trạng thái bộ công cụ tính toán]]
- **Finance Conformance**: [[02-platforms/Web-Docs/finance-core-conformance|Chuẩn hóa công thức tài chính Web]]
- **Security & Acceptance**: [[02-platforms/Web-Docs/an-security-review-2026-10-05|Security Review]] · [[02-platforms/Web-Docs/an-ai-acceptance-cases-2026-10-05|AI Acceptance Cases]]
- **Web Navigation & UX**: [[02-platforms/Web-Docs/navigation-map-2026-09-28|Navigation Map]]

### 📱 Mobile App (`finhome_reactnative`) — [[02-platforms/App-Docs/README|Mở App Docs]]
- **Knowledge Base tổng**: [[02-platforms/App-Docs/finhome-knowledge-base|FinHome Knowledge Base (Core Reference)]]
- **Kiến trúc hệ thống**: [[02-platforms/App-Docs/system-architecture|System Architecture]] · [[02-platforms/App-Docs/application-flow|Application Flow]]
- **Design System Glass**: [[02-platforms/App-Docs/design-system-glass|Design System Glass Spec]] · [[02-platforms/App-Docs/figma-token-spec|Figma Tokens]]
- **Roadmap & Release**: [[02-platforms/App-Docs/project-roadmap|Project Roadmap]] · [[02-platforms/App-Docs/ci-testflight-runner|CI TestFlight Runner]]

### 🤖 AI & Operation (`finhome_ai_operation`) — [[02-platforms/AI-Docs/architecture|Mở AI Docs]]
- **Kiến trúc AI Operation**: [[02-platforms/AI-Docs/architecture|AI Operation Architecture & Flow]]
- **Automation Superpowers**: [[02-platforms/AI-Docs/superpowers/specs-superpower|Specs Superpower]]

---

## 3. 🛡️ Quy ước Bất biến (Core Invariants & Moat)

1. **Refusal Posture & Neutrality Moat**:
   - [[02-platforms/App-Docs/refusal-architecture|Refusal Architecture & Compliance]]
   - **Tuyệt đối KHÔNG**: Tra cứu CIC, không bán data cho ngân hàng, không ranking ngân hàng theo hoa hồng, không định danh PII sang bên thứ ba.
   - Luôn giữ vị thế: Trung lập 100%, đứng về phía người mua nhà.
2. **Quy tắc Code & Docs**:
   - Mọi công thức tài chính phải là **pure function**.
   - Code phải khớp doc; nếu lệch → **fix code, không fix doc** (Doc là chuẩn nghiệp vụ).
   - Đơn vị tiền tệ: Chuẩn VND (`1,2 tỷ` / `31 triệu` / `10,5%/năm`).
3. **Cập nhật ghi chú**:
   - Khi chỉnh sửa tài liệu trong `02-platforms/`, Git của các repo tương ứng (`finhome_website`, `finhome_reactnative`, `finhome_ai_operation`) sẽ tự động nhận thay đổi.
