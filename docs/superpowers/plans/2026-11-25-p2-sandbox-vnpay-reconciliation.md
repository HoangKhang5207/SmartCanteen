# Kế hoạch Triển khai — Trục 3 · Slice P2: `SandboxBankingGateway`, `VNPay Sandbox`, Sổ cái `payment_transactions`, Webhook HMAC-SHA256 & Đối soát Ngân hàng

- **Đặc tả đã phê duyệt:** `docs/superpowers/specs/2026-11-25-p2-sandbox-banking-vnpay-reconciliation-design.md`
- **Nhánh làm việc:** `feat/p2-sandbox-vnpay-reconciliation` (trên cả 3 repo: `root`, `api/`, `miniapp/`)

## Task 1 — Nhánh & Kế hoạch
- Đánh dấu đặc tả Slice P2 `APPROVED (2026-11-25)` và tạo nhánh `feat/p2-sandbox-vnpay-reconciliation` trên 3 repo.

## Task 2 — Migrations, Sổ cái `payment_transactions`, `SandboxBankingGateway` & Webhook HMAC-SHA256
1. 4 migrations additive:
   - `2026_11_25_400001_create_sandbox_bank_accounts_table.php`
   - `2026_11_25_400002_create_payment_transactions_table.php`
   - `2026_11_25_400003_create_reconciliation_tables.php`
   - `2026_11_25_400004_create_gateway_webhook_events_table.php`
2. Enums, Models & Exceptions:
   - Mở rộng `PaymentMethod` (`VNPAY`) & `PaymentStatus` (`FAILED`, `PENDING_RECONCILE`).
   - Models: `SandboxBankAccount`, `PaymentTransaction`, `ReconciliationBatch`, `ReconciliationItem`, `GatewayWebhookEvent`.
   - Exceptions: `InsufficientBankBalanceException` (`422`), `PaymentGatewayException` (`502`/`504`).
3. `BankingGatewayInterface` (bổ sung `providerName()`, `fetchSettlementStatement(string $date)`), `MockEpayBankingGateway`, `SandboxBankingGateway`, `AppServiceProvider`, `config/services.php`, `api/.env.example`.
4. `PaymentLedgerService` + tích hợp vào `SpendingLimitService`, `OrderService`, `OrderController`, `CanteenPosPurchaseController`, `StoreOrderRequest` (bắt lỗi thanh toán ngoài transaction để ghi bền vững bản ghi `FAILED`/`TIMEOUT` vào `payment_transactions`).
5. `BankingWebhookController` (`POST /api/v1/webhooks/banking/sandbox` verify `HMAC-SHA256`, timestamp `±300s`, chống replay).
6. Tests: `SandboxBankingPaymentTest`, `BankingWebhookTest`, + bổ sung test rollback SMS cho `resendOtp`/`initiateUnlink` trong `SmsOtpGatewayTest` (Slice P1 Minor).

## Task 3 — `VNPay Sandbox`, Đối soát Ngân hàng (`Reconciliation`) & Admin Blade
1. `VnpayGatewayService` (ký/xác thực `HMAC-SHA512` chuẩn VNPay v2.1.0, tạo URL, xử lý IPN idempotent, Return URL, fail-closed ở production).
2. `VnpayPaymentController` & `routes/api.php` (`POST /payments/vnpay/create-url`, `GET /payments/vnpay/ipn`, `GET /payments/vnpay/return`).
3. `ReconciliationService` + Artisan command `canteen:reconcile {date?} {--provider=sandbox_epay}`.
4. Admin Blade:
   - `Admin\ReconciliationController` (`index`, `run`, `resolveItem`), cập nhật `Admin\CanteenCardManagementController` (`updateSandboxAccount`), `routes/web.php`, `admin/layout.blade.php`, `admin/canteen/reconciliation/index.blade.php`, `admin/canteen/cards/show.blade.php`.
5. Tests: `VnpaySandboxTest`, `AdminReconciliationTest`.

## Task 4 — Frontend Mini App (`miniapp/`) + Slice P1 Minor
1. `miniapp/.env.example`: bổ sung `VITE_DEMO_PHONE_LOGIN=true`.
2. `miniapp/src/features/orders/{types,api}.ts`: thêm `VNPAY`, `FAILED`, `PENDING_RECONCILE`, `createVnpayPaymentUrl(orderId)`.
3. `miniapp/src/pages/cart.tsx`: thêm phương thức `VNPAY` khi đã đăng nhập.
4. `miniapp/src/pages/order-detail.tsx`: badge `FAILED`/`PENDING_RECONCILE` & nút mở cổng **VNPay Sandbox** cho đơn `VNPAY` chưa thanh toán/thất bại.
5. Unit test `miniapp/src/features/orders/payment.test.ts`.

## Task 5 — Tài liệu, Kiểm chứng & Code Review Độc lập
- Cập nhật `docs/api-contract.md`, `docs/database.md`, `docs/PROJECT_STATE.md`.
- Chạy Docker `sc-p2` (Pint, PHPUnit, migrations/seeders) & Frontend (`npm test`, `tsc`, `vite build`).
- Gọi `Slice P2 Code Reviewer` rà soát độc lập toàn bộ diff của 3 repo.

## Task 6 — Khắc phục Review, Kiểm chứng cuối, Commit & Handoff
- Sửa các phát hiện Critical/Important, chạy lại cổng kiểm chứng sau lần sửa cuối, dọn Docker `sc-p2`, commit sạch trên 3 repo và viết Báo cáo Mục 17.
