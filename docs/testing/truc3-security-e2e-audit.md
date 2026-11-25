# Báo cáo Kiểm toán Bảo mật & Kiểm chứng E2E Toàn Hệ thống (Trục 3 · Slices P1–P4)

- **Ngày lập:** 2026-11-26
- **Phạm vi:** Toàn bộ 10/10 Module Căn tin Số (`FEATURES.md` §9) & 16/16 Nghiệp vụ Production-Ready (`FEATURES.md` §7) trên `api/` (Laravel 13 + PostgreSQL 17) và `miniapp/` (React 18 + TypeScript + ZaUI + `zmp-sdk`).

---

## 1. Ma trận Phủ Đầy 10/10 Module Căn tin Số (`FEATURES.md` §9)

| # | Module | Trạng thái | Bằng chứng Triển khai & Kiểm chứng |
|---|---|---|---|
| 1 | **Authentication Module** | **100% Đạt** | Zalo Phone Auth server-side (`POST /api/v1/auth/zalo-phone`), Demo Phone local (`POST /api/v1/auth/demo-phone`), Logout, Lazy Auth trên Mini App (`ZaloPhoneAuthTest`, `AuthApiTest`). |
| 2 | **Student Management** | **100% Đạt** | Admin Blade CRUD học sinh (`/admin/canteen/students`), gắn nhiều học sinh cho 1 phụ huynh, gắn tài khoản học sinh (`student_user_id`), chặn chuyển phụ huynh khi còn `BankLink` mở (`AdminStudentCrudTest`). |
| 3 | **Student Card Module** | **100% Đạt** | `CARD-01..07`, cấp phát thẻ mới/thay thế, khóa/mở/thu hồi tự động unlink & hủy OTP (`DigitalCanteenBankLinkTest`, `AdminCanteenManagementTest`). |
| 4 | **Banking Integration** | **100% Đạt** | `BankingGatewayInterface`, `SandboxBankingGateway` (số dư, chế độ lỗi, hoàn tiền), Webhook HMAC-SHA256 + anti-replay, `VnpayGatewayService` HMAC-SHA512 v2.1.0 (`SandboxBankingPaymentTest`, `BankingWebhookTest`, `VnpaySandboxTest`). |
| 5 | **OTP Verification** | **100% Đạt** | 6 chữ số bcrypt, TTL, cooldown, lockout, `SmsGatewayInterface` (`log`/`esms`/`twilio`) có rollback sạch khi SMS lỗi (`BankUnlinkOtpTest`, `SmsOtpGatewayTest`). |
| 6 | **Spending Limit** | **100% Đạt** | `LIMIT-01..13`, lazy daily reset `Asia/Ho_Chi_Minh`, audit log, Admin override, cảnh báo chạm ngưỡng thấp `<= 20%` (`SpendingLimitAndOrderPaymentTest`). |
| 7 | **Order Management** | **100% Đạt** | Server pricing, Idempotency-Key, Optimistic Locking, Tồn kho nguyên tử `stock_quantity` trừ & hoàn khi hủy, POS Quầy Căn tin `/admin/pos` kèm biên lai in (`ProductInventoryTest`, `StaffPosTerminalTest`, `OrderApiTest`). |
| 8 | **Payment Processing** | **100% Đạt** | Sổ cái bất biến `payment_transactions` (ghi bền vững cả `FAILED`/`TIMEOUT` ngoài rollback transaction + `gateway_txn_ref`), xử lý `INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`, VNPay IPN idempotent. |
| 9 | **Transaction History & Reconciliation** | **100% Đạt** | `ORDER-01..07`, `DETAIL-01..14`, `ReconciliationService` idempotent theo `(reconciliation_date, provider)` với 5 trạng thái khớp/lệch, CLI `canteen:reconcile`, Admin Blade `/admin/canteen/reconciliation` (`AdminReconciliationTest`). |
| 10 | **Notification Module** | **100% Đạt** | Bảng `canteen_notifications`, 7 sự kiện tự động sau commit / post-catch, `PushNotificationChannelInterface` (`log`/`zalo_oa`/`zalo_zns`) không làm lỗi luồng thanh toán, Mini App `/notifications` + badge `BottomNav`, Admin log `/admin/canteen/notifications` (`NotificationModuleTest`). |

---

## 2. Danh mục Kiểm toán Bảo mật (Security Controls Verification)

| Hạng mục Bảo mật | Biện pháp đã áp dụng | Trạng thái |
|---|---|---|
| **Mã hóa & Che mờ PII** | `account_number_encrypted`, `national_id_encrypted` mã hóa AES-256 (`encrypted` cast); OTP băm `bcrypt`; `DataMaskingService` che STK/CCCD/SĐT trên API, Blade, POS Lookup, Notification `meta` & Logs. | **PASS** |
| **RBAC 4 Vai trò & IDOR** | `admin`, `staff`, `parent` (alias `customer`), `student`. `student` bị chặn `403 PARENT_ROLE_REQUIRED` ở mọi route tài chính phụ huynh và `403` trên `/admin/*`; `staff` chỉ đọc ở các màn hình nhạy cảm; IDOR trả `404` trên Cards, Orders, Notifications, VNPay create-url. | **PASS** |
| **Chống Race & Deadlock** | Thứ tự khóa PostgreSQL thống nhất: `student_cards → spending_limits → sandbox_bank_accounts → products (id ASC) → orders → payment_transactions`. | **PASS** |
| **Webhook & VNPay Signing** | HMAC-SHA256 + timestamp `±300s` + `UNIQUE (provider, event_id)` anti-replay; VNPay HMAC-SHA512 + `vnp_TmnCode` + `vnp_Amount` + `vnp_TransactionStatus` + fail-closed khi production dùng khóa demo/rỗng. | **PASS** |
| **Chống XSS / Open Redirect / CSRF** | Blade escaping `{{ }}` & `escapeHtml()` thoát cả `"`/`'`; `isSafeHttpUrl()` cho VNPay; `isSafeInternalRoute()` cho Notification `action_url`; `@csrf` / `X-CSRF-TOKEN` trên mọi form/AJAX Admin. | **PASS** |
| **HTTP Security Headers & Rate Limiting** | `SecurityHeadersMiddleware` (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`); `throttle` trên auth, OTP, webhook và VNPay. | **PASS** |
| **Kiểm chứng E2E Liên hoàn** | `ProductionReadinessEndToEndTest::test_full_truc3_production_readiness_e2e_workflow_p1_to_p4` bao phủ trọn vẹn luồng P1→P2→P3→P4 + kiểm tra không lộ PII. | **PASS** |
