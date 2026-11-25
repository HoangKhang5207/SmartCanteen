# Trục 2 · Slice D1 — Hồ sơ Học sinh, Thẻ Thông minh & Liên kết Ngân hàng + OTP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans`, `superpowers:test-driven-development`, `frontend-ui-engineering`, `vercel-react-best-practices`, and `superpowers:verification-before-completion`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Hiện thực hóa trọn vẹn Slice D1 của phân hệ Căn tin Số (`FEATURES.md`, phương án `3A`): Hồ sơ Học sinh, Thẻ học sinh thông minh (`CARD-01..07`), Liên kết tài khoản ngân hàng & trích nợ tự động (`BANK-01..16`) và Xác thực OTP 6 chữ số (`OTP-01..07`) trên cả Backend `api/` và Frontend `miniapp/`.

**Architecture:** Mở rộng additive trên PostgreSQL 17 với 4 bảng mới (`students`, `student_cards`, `bank_links`, `otp_verifications`) cùng Partial Unique Index `WHERE status = 'ACTIVE'`, mã hóa AES-256 (`encrypted` cast) cho STK & CCCD, hash OTP (`Hash::make`), tách ranh giới tích hợp qua `BankingGatewayInterface` (`MockEpayBankingGateway`). Frontend Mini App mở rộng `BottomNav` thành 4 tab và thêm 3 màn hình (`/canteen`, `/canteen/link-bank`, `/canteen/verify-otp`) tuân thủ `DESIGN.md` và WCAG 2.1 AA, lưu draft form trong Jotai RAM (không dùng `localStorage`).

**Tech Stack:** Laravel 13 (PHP 8.4, Sanctum, PostgreSQL 17), React 18, TypeScript, ZaUI (`zmp-ui`), Jotai, TanStack Query, Tailwind CSS, PHPUnit 12, Vitest 2.

**Spec:** [2026-11-25-d1-student-card-bank-link-design.md](../specs/2026-11-25-d1-student-card-bank-link-design.md) (APPROVED 2026-11-25)

## Global Constraints

- 100% additive migrations — không phá vỡ 6 bảng và 53 PHPUnit tests + 18 Vitest tests hiện có.
- Không lưu STK, CCCD hay OTP dạng plaintext trong DB, log hay API response.
- `debug_otp` CHỈ được trả khi `app()->environment(['local', 'testing']) && config('services.banking.expose_debug_otp') === true`; tuyệt đối không xuất hiện trên `production`.
- Không phụ thuộc `localStorage` / `sessionStorage` / `cookie` trên Zalo Mini App.
- Không `git push`, không `git merge` vào `main`, làm việc trên nhánh `feat/d1-student-card-bank-link`.

## Review Focus

1. **Race condition liên kết trùng:** PostgreSQL Partial Unique Index `WHERE status = 'ACTIVE'` + `lockForUpdate()` trên `student_cards` phải ngăn tuyệt đối 2 liên kết `ACTIVE` cùng tồn tại trên 1 thẻ.
2. **Data leak protection:** Kiểm tra trực tiếp raw DB columns (`DB::table('bank_links')` / `otp_verifications`) xác nhận `account_number_encrypted`, `national_id_encrypted`, `code_hash` không chứa giá trị plaintext; API response tuyệt đối không có `account_number`, `national_id`, `code_hash` hay `debug_otp` khi tắt flag/production.
3. **OTP security edge cases:** Sai mã giảm `remaining_attempts`, sai lần thứ 5 khóa phiên (`429`), hết hạn 180s (`422` `OTP_EXPIRED`), gửi lại trong 30s (`429` `OTP_RESEND_COOLDOWN`), vượt quá 3 lần gửi lại (`429` `OTP_MAX_RESENDS_EXCEEDED`), và mã OTP cũ bị vô hiệu ngay khi gửi lại hoặc khởi tạo lại phiên mới (`OTP-06`).
4. **Cross-user authorization:** Mọi endpoint thao tác theo `cardId` hoặc `bankLinkId` thuộc người dùng khác phải trả `404 Not Found` (không tiết lộ tồn tại tài nguyên).
5. **Frontend UX & Accessibility:** `BottomNav` 4 tab không vỡ trên màn 320px, `OtpCodeInput` hỗ trợ gõ liên tục / Backspace / dán paste 6 số, nút xác thực chống double-submit, draft form giữ nguyên khi quay lại sửa SĐT và tự xóa khi liên kết xong.

---

### Task 1: RED Tests + Migrations, Enums, Models, Policies & Seeder (`api/`)

**Files:**
- Test: `api/tests/Unit/DataMaskingServiceTest.php`
- Test: `api/tests/Feature/Canteen/DigitalCanteenBankLinkTest.php`
- Create: `api/database/migrations/2026_11_25_100001_create_students_table.php`
- Create: `api/database/migrations/2026_11_25_100002_create_student_cards_table.php`
- Create: `api/database/migrations/2026_11_25_100003_create_bank_links_table.php`
- Create: `api/database/migrations/2026_11_25_100004_create_otp_verifications_table.php`
- Create: `api/app/Enums/StudentCardStatus.php`, `api/app/Enums/BankLinkStatus.php`, `api/app/Enums/OtpPurpose.php`
- Create: `api/app/Models/Student.php`, `api/app/Models/StudentCard.php`, `api/app/Models/BankLink.php`, `api/app/Models/OtpVerification.php`
- Modify: `api/app/Models/User.php`
- Create: `api/database/seeders/StudentCardSeeder.php`
- Modify: `api/database/seeders/DatabaseSeeder.php`

- [ ] **Step 1:** Viết trước `DataMaskingServiceTest` và `DigitalCanteenBankLinkTest`, chạy để xác nhận RED đúng nguyên nhân thiếu feature.
- [ ] **Step 2:** Tạo 4 migrations PostgreSQL (kèm Partial Unique Index `bank_links_one_active_per_card_idx`), 3 Enums, 4 Models (có `encrypted` cast & `#[Hidden]`), quan hệ trên `User`, và `StudentCardSeeder` idempotent.

---

### Task 2: Banking Gateway Adapter, Domain Services & Config (`api/`)

**Files:**
- Modify: `api/config/services.php`, `api/.env.example`, `api/.env.docker.example`, `api/app/Providers/AppServiceProvider.php`
- Create: `api/app/Contracts/BankingGatewayInterface.php`
- Create: `api/app/Services/Banking/MockEpayBankingGateway.php`
- Create: `api/app/Services/DataMaskingService.php`
- Create: `api/app/Services/StudentCardService.php`
- Create: `api/app/Services/BankLinkingService.php`
- Create: `api/app/Exceptions/BankLinkConflictException.php`, `api/app/Exceptions/OtpVerificationException.php`

- [ ] **Step 1:** Triển khai `DataMaskingService` và xác minh `DataMaskingServiceTest` chuyển từ RED → GREEN.
- [ ] **Step 2:** Triển khai `BankingGatewayInterface` + `MockEpayBankingGateway`, cấu hình `config/services.php` (`banking`), bind trong `AppServiceProvider`.
- [ ] **Step 3:** Triển khai `StudentCardService` và `BankLinkingService` (`initiateLink`, `verifyOtp`, `resendOtp` trong `DB::transaction` + `lockForUpdate()`).

---

### Task 3: Canteen REST API Controllers, Requests, Resources & Routes (`api/`)

**Files:**
- Create: `api/app/Http/Requests/InitiateBankLinkRequest.php`, `api/app/Http/Requests/VerifyBankLinkOtpRequest.php`
- Create: `api/app/Http/Resources/StudentResource.php`, `api/app/Http/Resources/BankLinkResource.php`, `api/app/Http/Resources/StudentCardResource.php`
- Create: `api/app/Http/Controllers/Api/V1/CanteenBankController.php`, `api/app/Http/Controllers/Api/V1/CanteenCardController.php`, `api/app/Http/Controllers/Api/V1/CanteenBankLinkController.php`
- Modify: `api/routes/api.php`

- [ ] **Step 1:** Tạo FormRequests (validate động độ dài STK theo `bank_code`, CCCD 12 số, SĐT VN, email tùy chọn, `agreed_terms`), Resources (chỉ trả dữ liệu đã mask), Controllers và đăng ký routes `/api/v1/canteen/*`.
- [ ] **Step 2:** Chạy toàn bộ PHPUnit suite (`php artisan test --compact`) & `vendor/bin/pint --test` trong Docker xác nhận 100% GREEN (bao gồm 53 test cũ + toàn bộ test mới Slice D1).
- [ ] **Step 3:** Commit trên nhánh `feat/d1-student-card-bank-link` của sub-repo `api/`.

---

### Task 4: Frontend Mini App — Canteen Domain, Validation (TDD) & 3 Màn hình UI (`miniapp/`)

**Files:**
- Modify: `miniapp/src/lib/api-client.ts` (bổ sung additive `errorCode` & `meta` trên `ApiError`)
- Create: `miniapp/src/features/canteen/types.ts`
- Create: `miniapp/src/features/canteen/validation.ts`
- Test: `miniapp/src/features/canteen/validation.test.ts`
- Create: `miniapp/src/features/canteen/api.ts`
- Create: `miniapp/src/features/canteen/store.ts`
- Create: `miniapp/src/components/student-card-view.tsx`
- Create: `miniapp/src/components/otp-code-input.tsx`
- Create: `miniapp/src/pages/canteen.tsx`
- Create: `miniapp/src/pages/bank-link-form.tsx`
- Create: `miniapp/src/pages/bank-link-otp.tsx`
- Modify: `miniapp/src/components/bottom-nav.tsx` (4 tab)
- Modify: `miniapp/src/components/layout.tsx` (3 route mới)

- [ ] **Step 1:** Viết trước `miniapp/src/features/canteen/validation.test.ts` (RED), chạy `npm test` xác nhận fail.
- [ ] **Step 2:** Triển khai `validation.ts`, `types.ts`, `api.ts`, `store.ts`, mở rộng `ApiError`, chạy `npm test` xác nhận GREEN.
- [ ] **Step 3:** Xây dựng `StudentCardView`, `OtpCodeInput`, `CanteenHubPage`, `BankLinkFormPage`, `BankLinkOtpPage`, cập nhật `BottomNav` & `Layout`.
- [ ] **Step 4:** Chạy `npm test`, `npx tsc --noEmit`, `npm run build` xác nhận 100% GREEN.
- [ ] **Step 5:** Commit trên nhánh `feat/d1-student-card-bank-link` của sub-repo `miniapp/`.

---

### Task 5: Cập nhật Tài liệu, Code Review Độc lập & Kiểm chứng Toàn diện

**Files:**
- Modify: `docs/api-contract.md`, `docs/database.md`, `docs/PROJECT_STATE.md`

- [ ] **Step 1:** Cập nhật tài liệu API contract, Database ERD và `PROJECT_STATE.md` trên repo cha.
- [ ] **Step 2:** Dispatch Fresh-context Code Reviewer rà soát toàn bộ diff của cả 3 repo theo **Review Focus**.
- [ ] **Step 3:** Khắc phục mọi vấn đề Critical/Important (nếu có), chạy lại toàn bộ Quality Gates (`pint --test`, `php artisan test`, `vitest`, `tsc`, `vite build`, Docker build smoke test), dọn sạch Docker test stack và bàn giao báo cáo Mục 17.
