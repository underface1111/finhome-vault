# 🏠 FinHome Ecosystem — Knowledge Hub & Architecture

> **Single Source of Truth** cho toàn bộ hệ sinh thái FinHome (Vietnamese Fintech Mua Nhà).  
> Vault này quản lý **Business Logic (Toán, Rules, Edge Cases)**, **Kiến trúc liên nền tảng (Web, App, Backend, AI)**, **Hợp đồng dữ liệu (API & Design System)** và **Khung thẩm định Quy hoạch BĐS**.

---

## 📌 BẢN ĐỒ ĐIỀU HÀNH HỆ SINH THÁI (MOC)

```
                            ┌─────────────────────────┐
                            │   01 - Business & Math  │
                            │   (Pure Logic & Moat)   │
                            └────────────┬────────────┘
                                         │
     ┌──────────────────┬────────────────┼────────────────┬──────────────────┐
     ▼                  ▼                ▼                ▼                  ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐
│  Web Portal  │ │  Mobile App  │ │   Backend    │ │  AI & Ops    │ │ Quy hoạch & Thẩm định│
│  (Next.js)   │ │ (Expo / RN)  │ │ (Serverless) │ │ (Automation) │ │  (Pháp lý & Bản đồ)  │
│ [[Web-Docs]] │ │ [[App-Docs]] │ │[[Backend-V.]]│ │ [[AI-Docs]]  │ │ [[04-quyhoach-bds]]  │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────────────┘
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

## 2. 🚀 Các Nền tảng trong Hệ sinh thái (Cross-Platform Hub)

Các thư mục dưới đây được đồng bộ 2 chiều trực tiếp (`symlink`) với từng repository:

### ⚙️ Backend Platform (`finhome_backend`) — [[02-platforms/Backend-Vault/00-Home|Mở Backend Hub MOC]]
- **Kiến trúc Serverless AWS**: [[02-platforms/Backend-Vault/01-Architecture/System Overview|AWS Serverless Topology]] · [[02-platforms/Backend-Vault/01-Architecture/Monorepo Structure|Monorepo Turbo Layout]]
- **Danh mục 33 Microservices**: [[02-platforms/Backend-Vault/03-Services/Services Catalog|33 Lambda Services Catalog]]
- **Cơ sở dữ liệu (Database)**: [[02-platforms/Backend-Vault/06-Database/Schema and Entities|PostgreSQL 11 Tables & ERD]] · [[02-platforms/Backend-Vault/06-Database/Migrations|Migrations Workflow]]
- **Hợp đồng API & Xác thực**: [[02-platforms/Backend-Vault/07-API/OpenAPI and Endpoints|OpenAPI & Endpoints]] · [[02-platforms/Backend-Vault/07-API/Cognito Authentication|Cognito Authentication]]
- **Phương pháp luận REFI & Compass**: [[02-platforms/Backend-Vault/02-Business-Domain/Compass Product|FinHome Compass Product]] · [[02-platforms/Backend-Vault/02-Business-Domain/REFI Methodology|REFI Methodology]]
- **Tài liệu dự án**: [[02-platforms/Backend-Docs/README|Backend Docs Folder]] · [[02-platforms/Backend-Docs/api-contracts|API Contracts Detail]]

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

## 3. 🗺️ Quy hoạch & Thẩm định BĐS (`04-quyhoach-bds/`)

Khung nghiệp vụ tra cứu quy hoạch, thẩm định pháp lý dự án và quỹ đất:

- **Khung quy trình thẩm định**: [[04-quyhoach-bds/README|FinHome – Quy hoạch & thẩm định dự án BĐS]]
  - **Flow 1**: Xác định vị trí và hồ sơ cần tra (số tờ, số thửa, tọa độ, bản đồ hiện trạng)
  - **Flow 2**: Đọc chức năng đất và chỉ tiêu (mật độ, tầng cao, hệ số sử dụng, chỉ giới xây dựng)
  - **Flow 3**: Đối chiếu hồ sơ dự án và khu vực (quy hoạch chi tiết 1/500, quyết định giao đất, GPXD)
  - **Flow 4**: Xác minh và lập kết luận (hồ sơ có thẩm quyền, phân loại rủi ro/giới hạn)
- **Nguồn dữ liệu thẩm định chính thức**: [[04-quyhoach-bds/sources/README|Official Sources & SHA256 Checksums]]
- **Mindmap kinh doanh**: [[04-quyhoach-bds/mindmap_kinh_doanh_website|Mindmap Kinh doanh Website & Khách hàng]]

---

## 4. 🛡️ Quy ước Bất biến (Core Invariants & Moat)

1. **Refusal Posture & Neutrality Moat**:
   - [[02-platforms/App-Docs/refusal-architecture|Refusal Architecture & Compliance]]
   - **Tuyệt đối KHÔNG**: Tra cứu CIC, không bán data cho ngân hàng, không ranking ngân hàng theo hoa hồng, không định danh PII sang bên thứ ba.
   - Luôn giữ vị thế: Trung lập 100%, đứng về phía người mua nhà.
2. **Quy tắc Code & Docs**:
   - Mọi công thức tài chính phải là **pure function**.
   - Code phải khớp doc; nếu lệch → **fix code, không fix doc** (Doc là chuẩn nghiệp vụ).
   - Đơn vị tiền tệ: Chuẩn VND (`1,2 tỷ` / `31 triệu` / `10,5%/năm`).
3. **Cập nhật ghi chú**:
   - Mọi chỉnh sửa trong các thư mục platform (`02-platforms/`, `04-quyhoach-bds/`) lập tức đồng bộ thẳng vào các nguồn tương ứng.
