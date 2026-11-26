# Danh mục Kịch bản Kiểm thử Toàn Hệ thống (Test Cases) — Smart Canteen & Căn tin Số (`docs/test-cases.md`)

Tài liệu này tổng hợp toàn bộ các kịch bản kiểm thử tự động trên **Backend PHPUnit (PostgreSQL 17)** và **Frontend Vitest**, bao phủ đầy đủ từ **MVP (S1–S4)** đến **Căn tin Số (D1–D3)** và **Production-Readiness (P1–P4 + Final Polish)**.

---

## 1. Nhóm Thực đơn, Tồn kho & CORS (`S1 + P3`)

| Mã TC | Kịch bản Kiểm thử | Kết quả Mong đợi | Test Suite | Trạng thái |
|---|---|---|---|---|
| **TC-01** | `MenuSeeder` & `DemoShowcaseSeeder` idempotent | Tạo đủ 6 danh mục, 30 món, 4 học sinh, 4 thẻ, 2 đơn lịch sử hôm qua, 1 đợt đối soát; chạy lại lần 2 không trùng bản ghi & thống kê hôm nay vẫn = `0` | `MenuSeederTest`, `DemoShowcaseSeederTest` | **PASS** |
| **TC-02** | Giá nguyên VND, `restrictOnDelete` & CORS `https://h5.zdn.vn` | Giá kiểu `int`; chặn xóa danh mục có món; preflight `OPTIONS` từ Zalo được chấp nhận, origin lạ bị chặn | `SchemaTest`, `CorsTest` | **PASS** |
| **TC-03** | Lọc danh mục, tìm kiếm tiếng Việt, phân trang & ẩn danh mục tắt | Chỉ trả món thuộc danh mục `is_active = true`; tìm kiếm không phân biệt hoa thường; `category_id` sai trả `422` | `MenuApiTest` | **PASS** |
| **TC-04** | Trừ/hoàn tồn kho nguyên tử & chặn oversell (`INSUFFICIENT_STOCK`) | Đặt vượt `stock_quantity` trả `422 INSUFFICIENT_STOCK` kèm `available_stock`; hủy đơn tự động hoàn kho; form Admin chống ghi đè kho đồng thời (`original_stock_quantity`) | `ProductInventoryTest` | **PASS** |

---

## 2. Nhóm Xác thực SĐT Zalo, Lazy Auth & RBAC 4 Vai trò (`S2 + P1 + P3`)

| Mã TC | Kịch bản Kiểm thử | Kết quả Mong đợi | Test Suite | Trạng thái |
|---|---|---|---|---|
| **TC-05** | Đăng nhập Zalo OAuth (`appsecret_proof`) & Zalo Phone (`POST /auth/zalo-phone`) | Server xác minh `access_token` và đổi `phone_token` qua `graph.zalo.me`, chuẩn hóa SĐT VN (`0xxxxxxxxx`), chống `ACCOUNT_CONFLICT` (`409`), chặn Demo ở production (`403`) | `AuthApiTest`, `ZaloPhoneAuthTest`, `phone.test.ts` | **PASS** |
| **TC-06** | Logout thu hồi Sanctum token | Token bị thu hồi ngay lập tức; request kế tiếp trả `401 Unauthorized` | `ZaloPhoneAuthTest` | **PASS** |
| **TC-07** | RBAC 4 vai trò (`admin`, `staff`, `parent`/`customer`, `student`) & IDOR | `student` xem/mua bằng thẻ mình nhưng bị chặn `403 PARENT_ROLE_REQUIRED` ở mọi API tài chính phụ huynh; `staff` không được sửa thực đơn/học sinh/đối soát (`403`); IDOR trả `404` | `FourRoleRbacTest`, `AdminWebPortalTest`, `RbacAndStatusLogSchemaTest` | **PASS** |

---

## 3. Nhóm Đặt đơn, Idempotency, State Machine & POS Quầy Căn tin (`S2–S4 + D2 + P3`)

| Mã TC | Kịch bản Kiểm thử | Kết quả Mong đợi | Test Suite | Trạng thái |
|---|---|---|---|---|
| **TC-08** | Server-side pricing, bảo toàn giá lịch sử & `Idempotency-Key` | Bỏ qua giá client gửi; retry cùng key + payload trả `200`, đổi payload trả `409`; đổi giá sản phẩm sau đó không đổi đơn cũ | `OrderApiTest`, `cart-logic.test.ts` | **PASS** |
| **TC-09** | State Machine đơn hàng, Optimistic Locking (`version`) & Audit Log | Chuyển sai bước trả `422`; `version` cũ trả `409 Conflict`; mỗi bước hợp lệ ghi `order_status_logs` | `AdminOrderApiTest`, `OrderSchemaTest` | **PASS** |
| **TC-10** | POS Staff Terminal (`/admin/pos`) & Biên lai in | Tra cứu thẻ theo mã thẻ/mã HS không lộ STK/CCCD đầy đủ; tạo đơn `#POS-...` trừ kho + hạn mức + ngân hàng; idempotent; hiển thị biên lai in; chặn `parent`/`student` (`403`) | `StaffPosTerminalTest` | **PASS** |

---

## 4. Nhóm Thẻ Học sinh, Liên kết / Hủy Liên kết Ngân hàng, SMS OTP & Hạn mức (`D1–D3 + P1`)

| Mã TC | Kịch bản Kiểm thử | Kết quả Mong đợi | Test Suite | Trạng thái |
|---|---|---|---|---|
| **TC-11** | Liên kết ngân hàng AES-256 + OTP 6 số (`BANK-01..15`, `OTP-01..07`) | Mã hóa STK/CCCD, che mờ khi trả API (`DataMaskingServiceTest`); Partial Unique Index tối đa 1 `ACTIVE`/thẻ; cooldown `30s`, max `5` lần sai (`429`), max `3` lần gửi lại | `DigitalCanteenBankLinkTest`, `DataMaskingServiceTest`, `validation.test.ts` | **PASS** |
| **TC-12** | Gửi SMS OTP qua `SmsGatewayInterface` (`log`/`esms`/`twilio`) & rollback khi lỗi | Khi cổng SMS lỗi HTTP 500: rollback sạch `bank_links`/`otp_verifications` và trả `503 SMS_DELIVERY_FAILED` | `SmsOtpGatewayTest` | **PASS** |
| **TC-13** | Hủy liên kết ngân hàng qua OTP (`UNLINK-01..07`, `OTP-08`) | OTP `BANK_UNLINK` xác thực thành công chuyển `UNLINKED` nhưng bảo toàn hạn mức ngày; nhập sai quá 5 lần chỉ khóa phiên OTP, không phá hỏng `ACTIVE` | `BankUnlinkOtpTest` | **PASS** |
| **TC-14** | Hạn mức chi tiêu hằng ngày, lazy reset & hoàn hạn mức khi hủy (`LIMIT-01..13`) | Kiểm tra bội số `1000`, `0..5_000_000`; reset `spent_today` sang ngày mới `Asia/Ho_Chi_Minh`; hủy đơn hoàn hạn mức + kho + số dư ngân hàng; ghi `spending_limit_logs` | `SpendingLimitAndOrderPaymentTest`, `spending-limit.test.ts` | **PASS** |
| **TC-15** | Quản trị Blade Học sinh & Thẻ (`/admin/canteen/students`, `/cards`) | CRUD Học sinh, cấp thẻ thay thế tự thu hồi thẻ cũ, khóa/mở/thu hồi thẻ tự ngắt ngân hàng, chặn chuyển phụ huynh khi còn `BankLink` mở | `AdminStudentCrudTest`, `AdminCanteenManagementTest` | **PASS** |

---

## 5. Nhóm Thanh toán Sandbox, VNPay, Webhook, Đối soát & Thông báo (`P2 + P4`)

| Mã TC | Kịch bản Kiểm thử | Kết quả Mong đợi | Test Suite | Trạng thái |
|---|---|---|---|---|
| **TC-16** | `SandboxBankingGateway` & Sổ cái bất biến `payment_transactions` | Đủ tiền trừ số dư + ghi `SUCCEEDED`; thiếu tiền trả `422 INSUFFICIENT_BANK_BALANCE`, từ chối `502 BANK_DECLINED`, timeout `504 GATEWAY_TIMEOUT` — đều rollback đơn/kho/hạn mức nhưng ghi bền vững `FAILED`/`TIMEOUT` vào sổ cái | `SandboxBankingPaymentTest` | **PASS** |
| **TC-17** | Webhook Ngân hàng Sandbox (`HMAC-SHA256` + Anti-replay) | Chữ ký sai/timestamp quá `±300s` trả `401`; sự kiện lặp `event_id` idempotent không xử lý 2 lần; xử lý đúng `CHARGE_SETTLED`, `CHARGE_FAILED`, `REFUND_SETTLED` | `BankingWebhookTest` | **PASS** |
| **TC-18** | Cổng thanh toán `VNPay Sandbox` (`HMAC-SHA512`) | Tạo URL thanh toán có `vnp_SecureHash`; IPN xác thực chữ ký + `vnp_Amount` + `vnp_TmnCode`, idempotent; fail-closed khi production dùng khóa demo | `VnpaySandboxTest`, `payment.test.ts` | **PASS** |
| **TC-19** | Đối soát Ngân hàng & CLI `canteen:reconcile` | Phát hiện đủ 5 trạng thái (`MATCHED`, `AMOUNT_MISMATCH`, `MISSING_LOCAL`, `MISSING_IN_BANK`, `STATUS_MISMATCH`); Admin xác nhận xử lý lệch giữ nguyên `note` gốc | `AdminReconciliationTest` | **PASS** |
| **TC-20** | Module Thông báo In-App + Push Zalo OA/ZNS + `SecurityHeadersMiddleware` | 7 sự kiện tự phát sau commit; lọc PII trong `meta`; chặn open redirect `action_url`; lỗi HTTP 500 từ Zalo OA/ZNS không làm lỗi thanh toán; kiểm tra HTTP security headers & index lọc Admin | `NotificationModuleTest`, `helpers.test.ts` | **PASS** |
| **TC-21** | Kiểm thử E2E Liên hoàn toàn hệ thống (MVP 8 bước + D1–D3 4 luồng + Trục 3 P1→P4) | Xác minh toàn bộ vòng đời từ đăng nhập SĐT → liên kết ngân hàng → đặt hạn mức → mua Online/POS → hủy đơn hoàn tiền/kho → đối soát → thông báo không lộ PII | `EndToEndDemoWorkflowTest`, `DigitalCanteenEndToEndFlowsTest`, `ProductionReadinessEndToEndTest` | **PASS** |
