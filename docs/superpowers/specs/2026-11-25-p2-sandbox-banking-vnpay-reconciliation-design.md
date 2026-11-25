# Đặc tả Thiết kế Chi tiết — Trục 3 · Slice P2: `SandboxBankingGateway`, `VNPay Sandbox`, Sổ cái `payment_transactions`, Xử lý Lỗi Giao dịch, Webhook HMAC-SHA256 & Đối soát Ngân hàng

- **Ngày lập:** 2026-11-25
- **Mã tác vụ:** `CANTEEN-P2-SANDBOX-VNPAY-RECONCILIATION`
- **Trạng thái:** `APPROVED (2026-11-25)`
- **Lộ trình cha:** `docs/superpowers/specs/2026-11-25-truc3-production-readiness-roadmap.md` (`APPROVED 2026-11-25`)
- **Nền tảng hiện tại trên `main`:** `root @ 8bd8340`, `api @ 63e1d02`, `miniapp @ c7889be` (102 PHPUnit tests / 1012 assertions, 36 Vitest tests).
- **Quyết định đã chốt từ người dùng:** Ưu tiên kết hợp **`SandboxBankingGateway` + `VNPay Sandbox`**, đồng thời dọn 2 Minor còn lại của Slice P1.

---

## 1. Mục tiêu & Phạm vi

### 1.1. Trong phạm vi (In Scope)
1. **Sổ cái Giao dịch Thanh toán Bất biến (`payment_transactions`):**
   - Lưu vết 100% mọi nỗ lực `CHARGE` và `REFUND` (bao gồm cả giao dịch `SUCCEEDED`, `FAILED`, `TIMEOUT`, `PENDING`), đảm bảo có chứng từ kế toán và không mất vết lỗi ngay cả khi transaction đặt đơn bị rollback.
2. **`SandboxBankingGateway` (chuẩn Production-Ready cho Trích nợ Tự động Thẻ học sinh):**
   - Quản lý số dư & chế độ mô phỏng theo từng tài khoản ngân hàng liên kết trong bảng `sandbox_bank_accounts` (mặc định số dư `500.000 đ`; hỗ trợ quy ước hậu tố STK deterministic khi liên kết: `...0000` → số dư `0 đ`, `...8888` → `BANK_DECLINED`, `...9999` → `GATEWAY_TIMEOUT`, và Admin có thể chỉnh số dư/chế độ ngay trên trang chi tiết thẻ).
   - Xử lý tường minh các kịch bản lỗi thực tế:
     - **Không đủ số dư tài khoản ngân hàng:** `422 INSUFFICIENT_BANK_BALANCE` (rollback không trừ `spent_today`, ghi dòng `FAILED` vào `payment_transactions`).
     - **Ngân hàng từ chối giao dịch:** `502 BANK_DECLINED` (rollback không trừ `spent_today`, ghi dòng `FAILED`).
     - **Timeout từ cổng ngân hàng:** `504 GATEWAY_TIMEOUT` (rollback không trừ `spent_today`, ghi dòng `TIMEOUT` để đối soát/webhook xử lý tiếp).
   - Hoàn tiền (`refundAutoDebit`) khi hủy đơn: cộng lại số dư vào `sandbox_bank_accounts` và ghi dòng `REFUND` (`SUCCEEDED`) vào `payment_transactions`.
   - Hỗ trợ **Webhook Callback ký `HMAC-SHA256`** (`POST /api/v1/webhooks/banking/sandbox`) với kiểm tra `hash_equals`, cửa sổ timestamp `±300s` và bảng chống replay `gateway_webhook_events`.
3. **Tích hợp Cổng Thanh toán `VNPay Sandbox` (chuẩn VNPay v2.1.0 `HMAC-SHA512`):**
   - Mở rộng `PaymentMethod::VNPAY` và `PaymentStatus::FAILED` / `PENDING_RECONCILE`.
   - 3 endpoints chuẩn Production VNPay:
     - `POST /api/v1/payments/vnpay/create-url` (`auth:sanctum`): sinh URL thanh toán VNPay có chữ ký `HMAC-SHA512` và tạo bản ghi `payment_transactions` (`PENDING`).
     - `GET /api/v1/payments/vnpay/ipn`: nhận IPN Callback, xác thực chữ ký `HMAC-SHA512`, đối chiếu số tiền (`vnp_Amount === amount * 100`), xử lý idempotent (`RspCode=02` khi đã thanh toán), cập nhật `payment_transactions` và `orders.payment_status`, trả JSON chuẩn `{ RspCode, Message }`.
     - `GET /api/v1/payments/vnpay/return`: xác thực chữ ký và trả trạng thái giao dịch cho client.
   - Hỗ trợ cấu hình `VNPAY_TMN_CODE` / `VNPAY_HASH_SECRET` sandbox thật **và** có fallback sandbox secret local (chỉ ở `local`/`testing`) để demo và kiểm thử tự động End-to-End 100% mà không phụ thuộc mạng ngoài.
4. **Đối soát Giao dịch Ngân hàng (`Reconciliation`):**
   - Bảng `reconciliation_batches` và `reconciliation_items`.
   - `ReconciliationService` đối chiếu sổ cái `payment_transactions` theo ngày `Asia/Ho_Chi_Minh` với sao kê từ `BankingGatewayInterface::fetchSettlementStatement($date)` → phân loại `MATCHED`, `MISSING_IN_BANK`, `MISSING_LOCAL`, `AMOUNT_MISMATCH`, `STATUS_MISMATCH`.
   - Artisan command `php artisan canteen:reconcile {date?} {--provider=sandbox_epay}`.
   - Màn hình Admin Blade **Đối soát & Giao dịch** (`/admin/canteen/reconciliation`, `role:staff,admin` xem; `role:admin` chạy đối soát & đánh dấu xử lý lệch) + khung chỉnh số dư/mode Sandbox trên `/admin/canteen/cards/{studentCard}` (`role:admin`, chỉ hiển thị khi dùng `sandbox` driver).
5. **Nâng cấp Mini App (`miniapp/`):**
   - Thêm lựa chọn **"Thanh toán VNPay (Sandbox)"** trên `CartPage` và nút thanh toán / hiển thị trạng thái `FAILED` / `PENDING_RECONCILE` trên `OrderDetailPage`.
   - Hiển thị thông báo lỗi nghiệp vụ rõ ràng trên `CanteenHubPage` & `CartPage` khi gặp `INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`.
6. **Dọn 2 Minor còn lại của Slice P1:**
   - Thêm `VITE_DEMO_PHONE_LOGIN=true` vào `miniapp/.env.example`.
   - Thêm test rollback SMS cho `resendOtp` và `initiateUnlink` trong `SmsOtpGatewayTest.php`.

### 1.2. Ngoài phạm vi (Out of Scope cho Slice P2)
- Không triển khai POS Staff Terminal, tồn kho sản phẩm, CRUD đa học sinh hay vai trò `student` (thuộc **Slice P3**).
- Không triển khai Notification Center / Zalo OA Push (thuộc **Slice P4**).
- Không giữ khóa DB khi gọi HTTP bên ngoài; không tự ý `git push` hay deploy.

---

## 2. Thiết kế Cơ sở Dữ liệu (4 Migrations Additive)

### 2.1. `2026_11_25_400001_create_sandbox_bank_accounts_table.php`
| Cột | Kiểu | Ràng buộc / Ghi chú |
|---|---|---|
| `id` | `bigIncrements` | PK |
| `provider_reference` | `VARCHAR(100)` | `UNIQUE` (liên kết 1-1 với `bank_links.provider_reference`) |
| `bank_code` | `VARCHAR(20)` | `MB`, `VCB`, ... |
| `account_number_masked` | `VARCHAR(32)` | Chỉ lưu bản che mờ |
| `balance` | `BIGINT UNSIGNED` | Số dư giả lập (VND), mặc định `500000` |
| `simulated_mode` | `VARCHAR(24)` | `NORMAL` \| `INSUFFICIENT_FUNDS` \| `BANK_DECLINED` \| `GATEWAY_TIMEOUT` |
| `created_at`, `updated_at` | `TIMESTAMP` | Timestamps |

### 2.2. `2026_11_25_400002_create_payment_transactions_table.php`
| Cột | Kiểu | Ràng buộc / Ghi chú |
|---|---|---|
| `id` | `bigIncrements` | PK |
| `order_id` | `BIGINT UNSIGNED NULL` | FK → `orders.id` (`restrictOnDelete`) |
| `student_card_id` | `BIGINT UNSIGNED NULL` | FK → `student_cards.id` (`restrictOnDelete`) |
| `bank_link_id` | `BIGINT UNSIGNED NULL` | FK → `bank_links.id` (`restrictOnDelete`) |
| `user_id` | `BIGINT UNSIGNED` | FK → `users.id` (`restrictOnDelete`) |
| `provider` | `VARCHAR(32)` | `sandbox_epay` \| `mock_epay` \| `vnpay` |
| `type` | `VARCHAR(16)` | `CHARGE` \| `REFUND` |
| `amount` | `INTEGER UNSIGNED` | Số tiền giao dịch (VND) |
| `status` | `VARCHAR(24)` | `PENDING` \| `SUCCEEDED` \| `FAILED` \| `TIMEOUT` |
| `failure_code` | `VARCHAR(64) NULL` | `INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`, `VNPAY_FAILED`, ... |
| `failure_message` | `VARCHAR(255) NULL` | Thông điệp lỗi không chứa PII |
| `idempotency_key` | `VARCHAR(120) NULL` | Khóa chống trùng từ yêu cầu gốc |
| `provider_reference` | `VARCHAR(120) NULL` | Mã giao dịch từ cổng thanh toán (`EPAY-CHG-...`, `vnp_TransactionNo`) |
| `gateway_txn_ref` | `VARCHAR(120) NULL` | `UNIQUE` — mã tham chiếu gửi sang cổng (vd `VNP-...` / `order.code`) |
| `response_meta` | `JSONB NULL` | Metadata kỹ thuật đã lọc sạch PII |
| `processed_at` | `TIMESTAMP NULL` | Thời điểm hoàn tất/thất bại |
| `created_at`, `updated_at` | `TIMESTAMP` | Index `(provider, created_at)`, `(order_id, type)`, `(student_card_id, created_at)` |

### 2.3. `2026_11_25_400003_create_reconciliation_tables.php`
- **`reconciliation_batches`:** `id`, `reconciliation_date DATE`, `provider VARCHAR(32)`, `status VARCHAR(24)` (`COMPLETED` | `HAS_DISCREPANCIES`), `total_local_count`, `total_provider_count`, `matched_count`, `discrepancy_count`, `total_local_amount BIGINT`, `total_provider_amount BIGINT`, `run_by BIGINT UNSIGNED NULL` (FK `users.id`), `completed_at TIMESTAMP`, `timestamps`. Index `(reconciliation_date, provider)`.
- **`reconciliation_items`:** `id`, `batch_id` (FK `reconciliation_batches.id` `cascadeOnDelete`), `payment_transaction_id` (FK `payment_transactions.id` nullable `restrictOnDelete`), `provider_reference VARCHAR(120) NULL`, `order_code VARCHAR(64) NULL`, `local_amount INTEGER NULL`, `provider_amount INTEGER NULL`, `status VARCHAR(32)` (`MATCHED` | `MISSING_IN_BANK` | `MISSING_LOCAL` | `AMOUNT_MISMATCH` | `STATUS_MISMATCH`), `note VARCHAR(255) NULL`, `resolved_at TIMESTAMP NULL`, `resolved_by BIGINT UNSIGNED NULL` (FK `users.id`), `timestamps`.

### 2.4. `2026_11_25_400004_create_gateway_webhook_events_table.php`
- **`gateway_webhook_events`:** `id`, `provider VARCHAR(32)`, `event_id VARCHAR(120)`, `event_type VARCHAR(64)`, `signature_valid BOOLEAN`, `payload_hash VARCHAR(64)`, `processed_at TIMESTAMP NULL`, `timestamps`. `UNIQUE (provider, event_id)`.

---

## 3. Thiết kế Backend (`api/`)

### 3.1. Thứ tự Khóa Hàng PostgreSQL Chuẩn (Deadlock-Free)
- Mọi luồng thanh toán/hoàn tiền giữ nghiêm thứ tự khóa hàng chuẩn mở rộng:
  `student_cards → spending_limits → sandbox_bank_accounts → products → orders → payment_transactions`
- Khi `OrderService::createOrder()` hoặc `createPosPurchase()` gặp `InsufficientBankBalanceException` hoặc `PaymentGatewayException`:
  1. `DB::transaction()` tự động **ROLLBACK** toàn bộ (hạn mức `spent_today` không bị trừ, số dư sandbox không bị trừ, đơn hàng không được tạo).
  2. Khối `catch` **bên ngoài** transaction gọi `PaymentLedgerService::recordFailedAttempt(...)` để ghi bền vững bản ghi `payment_transactions` (`status = FAILED` hoặc `TIMEOUT`, `failure_code`, `failure_message`), sau đó ném lại exception để Controller trả đúng HTTP code (`422` / `502` / `504`) kèm `error_code`.

### 3.2. `SandboxBankingGateway` vs `MockEpayBankingGateway`
- Binding qua `BANKING_GATEWAY_DRIVER` (`sandbox` | `mock`, mặc định `sandbox` trong `.env.example`, giữ `mock` tương thích ngược):
  - `SandboxBankingGateway`: lưu & khóa số dư trên `sandbox_bank_accounts`, kiểm tra `simulated_mode`, trừ/hoàn tiền thật trên số dư sandbox, và cung cấp `fetchSettlementStatement(string $date)` trả sao kê ngày `Asia/Ho_Chi_Minh`.
  - Hỗ trợ `BankingGatewayInterface::providerName(): string` (`sandbox_epay` / `mock_epay`) và `fetchSettlementStatement(string $date): array`.
- **Cấu hình Webhook Sandbox (`config/services.php`):**
  - `banking.webhook_secret = env('BANKING_WEBHOOK_SECRET', '')` (fail-closed `503` nếu rỗng).
  - `banking.webhook_tolerance_seconds = 300` (5 phút chống replay).
  - Endpoint `POST /api/v1/webhooks/banking/sandbox` (`throttle:60,1`):
    - Kiểm tra header `X-Gateway-Timestamp` & `X-Gateway-Signature` (`hash_hmac('sha256', "{$timestamp}.{$rawBody}", $secret)` bằng `hash_equals`).
    - Kiểm tra `gateway_webhook_events` (`UNIQUE (provider, event_id)`): nếu đã xử lý trả `200 OK` `{ "status": "already_processed" }` (idempotent).
    - Hỗ trợ sự kiện `CHARGE_SETTLED`, `CHARGE_FAILED`, `REFUND_SETTLED` cập nhật trạng thái `payment_transactions` & `orders.payment_status` tương ứng trong `DB::transaction()`.

### 3.3. `VnpayGatewayService` & Endpoints VNPay Sandbox
- Cấu hình trong `config/services.php` & `api/.env.example`:
  ```php
  'vnpay' => [
      'tmn_code' => env('VNPAY_TMN_CODE', ''),
      'hash_secret' => env('VNPAY_HASH_SECRET', ''),
      'payment_url' => env('VNPAY_PAYMENT_URL', 'https://sandbox.vnpayment.vn/paymentv2/vpcpay.html'),
      'return_url' => env('VNPAY_RETURN_URL', 'http://localhost:8080/api/v1/payments/vnpay/return'),
      'sandbox_simulator_enabled' => filter_var(env('VNPAY_SANDBOX_SIMULATOR_ENABLED', false), FILTER_VALIDATE_BOOL),
  ],
  ```
- **Thuật toán chữ ký chuẩn VNPay v2.1.0:**
  - Loại bỏ `vnp_SecureHash` & `vnp_SecureHashType`, bỏ các tham số rỗng, sắp xếp tăng dần theo key (`ksort`), mã hóa `urlencode($key) . '=' . urlencode($value)` nối bằng `&`, tính `hash_hmac('sha512', $hashData, $hashSecret)`.
- **Endpoints:**
  | Method | Endpoint | Auth | Chức năng |
  |---|---|---|---|
  | `POST` | `/api/v1/payments/vnpay/create-url` | `auth:sanctum`, `throttle:20,1` | Nhận `{ order_id }` (chỉ chủ đơn có `payment_method = VNPAY` và `payment_status in [UNPAID, FAILED]`), tạo dòng `payment_transactions` (`PENDING`), trả `{ payment_url, txn_ref, amount, expires_at }`. |
  | `GET` | `/api/v1/payments/vnpay/ipn` | Checksum `HMAC-SHA512` | Verify chữ ký (`97` nếu sai), tìm `payment_transactions` theo `vnp_TxnRef` (`01` nếu không tồn tại), đối chiếu `vnp_Amount === amount * 100` (`04` nếu lệch), nếu đã `SUCCEEDED` trả `02`, ngược lại cập nhật `SUCCEEDED` (`vnp_ResponseCode === '00'`) hoặc `FAILED` + đồng bộ `orders.payment_status` (`PAID`/`FAILED`), trả `00`. |
  | `GET` | `/api/v1/payments/vnpay/return` | Checksum `HMAC-SHA512` | Verify chữ ký và trả JSON kết quả thanh toán rõ ràng cho trình duyệt/Mini App. |
- **Fail-closed:** Nếu `VNPAY_TMN_CODE` hoặc `VNPAY_HASH_SECRET` rỗng: ở `production` luôn trả `503 VNPAY_CONFIG_MISSING`; ở `local`/`testing` chỉ cho dùng credential sandbox nội bộ khi `VNPAY_SANDBOX_SIMULATOR_ENABLED=true`.

### 3.4. Đối soát Giao dịch (`ReconciliationService`) & Tuyến Admin Blade
- `ReconciliationService::runForDate(string $date, string $provider = 'sandbox_epay', ?User $actor = null, ?array $overrideStatement = null): ReconciliationBatch`:
  - Lấy toàn bộ `payment_transactions` trong ngày `Asia/Ho_Chi_Minh` thuộc `$provider` (type `CHARGE`, status `SUCCEEDED`/`TIMEOUT`/`FAILED`).
  - Đối chiếu theo `provider_reference` với sao kê từ `$gateway->fetchSettlementStatement($date)` (hoặc `$overrideStatement` khi test kịch bản lệch).
  - Tạo `ReconciliationBatch` + danh sách `ReconciliationItem` (`MATCHED`, `MISSING_IN_BANK`, `MISSING_LOCAL`, `AMOUNT_MISMATCH`, `STATUS_MISMATCH`) trong `DB::transaction()`.
- **Route Admin Blade mới (`/admin/canteen/...`):**
  - `GET /admin/canteen/reconciliation` (`role:staff,admin`): Xem các phiên đối soát, chi tiết lệch, và bảng sổ cái `payment_transactions` (lọc theo trạng thái/provider).
  - `POST /admin/canteen/reconciliation/run` (`role:admin`): Chạy đối soát theo ngày chọn.
  - `PATCH /admin/canteen/reconciliation/items/{item}/resolve` (`role:admin`): Xác nhận xử lý lệch kèm ghi chú.
  - `PUT /admin/canteen/cards/{studentCard}/sandbox-account` (`role:admin`): Chỉnh số dư (`balance`) & chế độ lỗi (`simulated_mode`) của tài khoản Sandbox gắn với thẻ để phục vụ kiểm thử trực quan trên Local/Dev.

---

## 4. Thiết kế Frontend Mini App (`miniapp/`)

1. **`CartPage` (`miniapp/src/pages/cart.tsx`):**
   - Bổ sung lựa chọn phương thức thanh toán thứ 3: **`Thanh toán VNPay (Sandbox)`** (`VNPAY`) hiển thị ngay khi người dùng đã đăng nhập (kể cả khi chưa liên kết thẻ học sinh!).
   - Khi đặt đơn bằng `VNPAY`, điều hướng sang `OrderDetailPage` với nút **"Thanh toán qua VNPay Sandbox"** gọi `POST /api/v1/payments/vnpay/create-url`.
2. **`OrderDetailPage` (`miniapp/src/pages/order-detail.tsx`) & `features/orders/types.ts`:**
   - Mở rộng `PaymentMethod` thêm `"VNPAY"` và `PaymentStatus` thêm `"FAILED" | "PENDING_RECONCILE"`.
   - Hiển thị badge màu chuẩn WCAG 2.1 AA cho `FAILED` (`bg-red-100 text-red-800`) và `PENDING_RECONCILE` (`bg-amber-100 text-amber-800`).
   - Với đơn `VNPAY` đang `UNPAID` hoặc `FAILED` (và đơn chưa `CANCELLED`): hiển thị nút **"Mở cổng thanh toán VNPay Sandbox"** tạo URL thanh toán có chữ ký HMAC-SHA512.
3. **Hiển thị lỗi tài khoản không đủ tiền / timeout rõ ràng trên `CanteenHubPage` & `CartPage`:**
   - Lỗi từ API (`INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`) hiển thị bằng banner `role="alert"` tiếng Việt rõ ràng kèm gợi ý hành động.
4. **Dọn Minor Slice P1:** Bổ sung `VITE_DEMO_PHONE_LOGIN=true` vào `miniapp/.env.example`.

---

## 5. Kế hoạch Kiểm chứng & Tiêu chí Chấp nhận Slice P2

| Hạng mục | Test Suite | Tiêu chí đạt |
|---|---|---|
| `SandboxBankingGateway` & Xử lý lỗi giao dịch | `api/tests/Feature/Canteen/SandboxBankingPaymentTest.php` | Đủ tiền trừ `balance` + `spent_today` + ghi `payment_transactions (SUCCEEDED)`; không đủ tiền trả `422 INSUFFICIENT_BANK_BALANCE` không trừ hạn mức & ghi `FAILED`; `BANK_DECLINED` trả `502` & ghi `FAILED`; `GATEWAY_TIMEOUT` trả `504` & ghi `TIMEOUT`; hủy đơn hoàn số dư sandbox + ghi dòng `REFUND` |
| Webhook HMAC-SHA256 & Anti-replay | `api/tests/Feature/Canteen/BankingWebhookTest.php` | Chữ ký hợp lệ xử lý idempotent (`200`), replay cùng `event_id` trả `already_processed`, chữ ký sai trả `401`, timestamp quá hạn `>300s` trả `401`, thiếu `BANKING_WEBHOOK_SECRET` fail-closed `503` |
| `VNPay Sandbox` (`HMAC-SHA512`) | `api/tests/Feature/Payment/VnpaySandboxTest.php` | Tạo URL có `vnp_SecureHash` hợp lệ, IPN `00` chuyển `PAID` & `SUCCEEDED`, IPN trùng trả `RspCode=02`, IPN sai checksum trả `RspCode=97`, IPN lệch số tiền trả `RspCode=04`, IPN mã lỗi `51` chuyển `FAILED`, IDOR `404` khi tạo URL cho đơn người khác |
| Đối soát Ngân hàng & Admin Blade | `api/tests/Feature/Admin/AdminReconciliationTest.php` | Chạy đối soát phát hiện đủ `MATCHED`, `MISSING_IN_BANK`, `MISSING_LOCAL`, `AMOUNT_MISMATCH`, `STATUS_MISMATCH`; RBAC (`staff` xem `200`, mutate `403`; `admin` chạy đối soát, resolve lệch, cập nhật số dư/mode Sandbox); Artisan `canteen:reconcile` PASS |
| Slice P1 Minor (SMS Rollback) | Cập nhật `api/tests/Feature/Canteen/SmsOtpGatewayTest.php` | Kiểm chứng rollback sạch khi SMS lỗi ở cả `resendOtp` (giữ nguyên `resend_count`) và `initiateUnlink` |
| Hồi quy toàn hệ thống | `vendor/bin/pint --test`, `php artisan test --compact`, `php artisan migrate --force --seed`, `npm test`, `npx tsc --noEmit`, `npm run build` | 100% test cũ (102 Backend + 36 Frontend) + test mới đều PASS trên PostgreSQL 17 |
