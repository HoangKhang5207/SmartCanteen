# Kế hoạch Triển khai Chi tiết — Trục 3 · Slice P4 (`CANTEEN-P4-NOTIFICATIONS-SECURITY-E2E`)

- **Ngày lập:** 2026-11-26
- **Đặc tả nguồn:** `docs/superpowers/specs/2026-11-26-p4-notifications-security-e2e-design.md` (`APPROVED 2026-11-26`)
- **Nhánh làm việc (3 repo):** `feat/p4-notifications-security-e2e`

---

## Thứ tự Triển khai (6 Giai đoạn Gia tăng)

### Giai đoạn 1 — Schema, Enums, Push Channel Strategy & `NotificationService` (`api/`)
1. Migration `2026_11_26_600001_create_canteen_notifications_table.php` (`canteen_notifications` với indexes `(user_id, read_at, id)` và `(user_id, type, created_at)`).
2. Enum `NotificationType`, `NotificationSeverity`, Model `CanteenNotification`, Resource `CanteenNotificationResource`.
3. Interface `PushNotificationChannelInterface` + 3 driver (`LogPushNotificationChannel`, `ZaloOaPushChannel`, `ZaloZnsPushChannel`) + cấu hình `config/services.php` & `api/.env.example` + binding động trong `AppServiceProvider`.
4. `NotificationService`:
   - `dispatch(...)` và `dispatchAfterCommit(...)` bọc `try/catch` toàn diện — lỗi push tuyệt đối không ảnh hưởng/rollback giao dịch tài chính.
   - Các phương thức nghiệp vụ có ngữ nghĩa rõ ràng (`notifyPurchaseSuccess`, `notifyPaymentFailed`, `notifyOrderRefunded`, `notifyOrderStatusUpdated`, `notifyBankLinkChanged`, `notifySpendingLimitUpdated`), tự động phát kèm `LOW_SPENDING_LIMIT` khi còn `<= 20%` hạn mức ngày và kiểm tra chặt không chứa PII thô.

### Giai đoạn 2 — Gắn Domain Hooks, REST API, Admin Log, Security Middleware & 3 P3 Backend Minors (`api/`)
1. Gắn `NotificationService` (qua `dispatchAfterCommit` / post-catch) vào: `OrderService`, `VnpayGatewayService`, `BankLinkingService`, `SpendingLimitService`, `StudentCardService`.
2. Customer/Parent/Student REST API `NotificationController` (`GET /api/v1/notifications`, `GET /api/v1/notifications/unread-count`, `PATCH /api/v1/notifications/{id}/read`, `POST /api/v1/notifications/read-all`, IDOR 404).
3. Admin Blade `Admin/NotificationLogController` (`GET /admin/canteen/notifications`, `role:staff,admin`) + Blade `admin/canteen/notifications/index.blade.php` + nav link ở `admin/layout.blade.php`.
4. `SecurityHeadersMiddleware` đăng ký global tại `api/bootstrap/app.php` (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`).
5. Xử lý P3 Backend Minors:
   - `ProductManagementController::update()` + `admin/products/form.blade.php`: khóa `Product::lockForUpdate()` & chỉ cập nhật `stock_quantity` khi khác `original_stock_quantity`.
   - `admin/pos/index.blade.php`: prepend động đơn POS vừa thanh toán vào sidebar `"Đơn POS gần đây hôm nay"`.

### Giai đoạn 3 — Mini App Notification Center & P3 Frontend Minors (`miniapp/`)
1. Tạo `src/features/notifications/{types,api,helpers,helpers.test}.ts` (`isSafeInternalRoute`, `getNotificationTone`).
2. Tạo `src/pages/notifications.tsx` (`/notifications`, Lazy Auth, lọc `Tất cả`/`Chưa đọc`, đọc từng mục & đọc tất cả, phân trang `PaginationBar`, điều hướng nội bộ an toàn).
3. Đăng ký route `/notifications` trong `src/components/layout.tsx` và nâng cấp `src/components/bottom-nav.tsx` thành 5 tab gọn kèm badge số chưa đọc.
4. Xử lý P3 Frontend Minors:
   - `CartPage`: roving `tabIndex` cho `role="radiogroup"`, ẩn nút `"Đổi hạn mức"` với vai trò `student`, invalidate `["notifications"]`/`["notifications-unread"]` sau checkout.
   - `LimitHistoryPage` & `CanteenHubPage`: câu chữ thân thiện cho vai trò `student` + invalidate notification queries sau POS swipe/limit/unlink.
   - `MenuPage`: kiểm tra chạm trần tồn kho trước khi gọi `addCartItem()` để hiển thị snackbar cảnh báo chính xác.

### Giai đoạn 4 — Automated Tests, Security/E2E Audit Report & Docs
1. Viết 2 bộ test Backend mới:
   - `NotificationModuleTest.php` (7 loại sự kiện, `LOW_SPENDING_LIMIT` `<= 20%`, `ZaloOaPushChannel` & `ZaloZnsPushChannel` qua `Http::fake()`, resilience khi push 500 không rollback đơn hàng, API unread/read/read-all & IDOR 404, Admin Blade RBAC, Security Headers).
   - `ProductionReadinessEndToEndTest.php` (kịch bản E2E toàn hệ thống xuyên suốt Trục 3: P1 Zalo Phone Auth + SMS OTP → P2 Sandbox Bank + VNPay IPN + Reconciliation → P3 Inventory + Staff POS + 4-Role RBAC → P4 Notifications + PII/IDOR Audit).
2. Cập nhật tài liệu: `docs/testing/truc3-security-e2e-audit.md`, `docs/api-contract.md` (§11), `docs/database.md` (§7), `docs/PROJECT_STATE.md`.

### Giai đoạn 5 — Independent Code Review & Khắc phục
1. Chạy **Independent Adversarial Code Review** (`research` subagent) toàn bộ diff trên cả 3 repo.
2. Sửa triệt để toàn bộ Critical & Important (và Minor nhanh).

### Giai đoạn 6 — Final Fresh Verification, Docker Teardown, Commit & Handoff
1. Chạy tươi: `vendor/bin/pint --test`, `php artisan test --compact`, `php artisan migrate:fresh --force --seed`, `npm test`, `npx tsc --noEmit`, `npm run build`.
2. Dừng & dọn sạch Docker stack (`down -v`), commit sạch trên cả 3 repo (`feat/p4-notifications-security-e2e`), bàn giao báo cáo nghiệm thu toàn bộ Trục 3.
