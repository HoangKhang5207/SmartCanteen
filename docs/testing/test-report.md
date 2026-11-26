# Báo cáo Kết quả Kiểm thử Tự động Toàn Hệ thống — Smart Canteen & Căn tin Số (`docs/testing/test-report.md`)

- **Ngày thực thi gần nhất:** `2026-11-26`
- **Môi trường kiểm thử chuẩn (Docker & Local):**
  - Backend: PHP `8.4-fpm-alpine` (Laravel `13.x`, PHPUnit `12.x`) trong Docker (`api/compose.yaml`)
  - Cơ sở dữ liệu kiểm thử: PostgreSQL `17-alpine` (`titkul_canteen_test` — test trên Postgres thật, không dùng SQLite)
  - Frontend: Node.js `v24`, Vitest `2.1.9`, TypeScript `5.9.3`, Vite `5.4.21`

---

## 1. Tổng hợp Kết quả Thực thi (Executive Test Summary)

| Hạng mục Kiểm thử | Công cụ | Phạm vi | Số bài test | Số khẳng định (Assertions) | Kết quả |
|---|---|---|---|---|---|
| **Backend Feature & Unit Tests** | PHPUnit + PostgreSQL 17 | **32 test files** (`api/tests/`) | **`126`** | **`1,387`** | **100% PASS (`126/126`)** |
| **Backend Code Style** | Laravel Pint (`vendor/bin/pint --test`) | **`200` PHP files** | — | — | **PASS (`200 files`)** |
| **DB Migrations & Idempotent Seeders** | `migrate:fresh --seed` + `db:seed` lần 2 | **27 migrations · 4 seeders** | — | — | **PASS (0 lỗi, idempotent)** |
| **Frontend Unit Tests** | Vitest (`npm test`) | **11 test files** (`miniapp/src/`) | **`44`** | — | **100% PASS (`44/44`)** |
| **Frontend Static Type Check** | TypeScript (`npx tsc --noEmit`) | Toàn bộ `miniapp/src` | — | — | **PASS (`0 errors`)** |
| **Frontend Production Bundle** | Vite (`npm run build`) | `miniapp/www` (`608 modules`) | — | — | **PASS (`8.49s`)** |

---

## 2. Chi tiết 32 Bộ Kiểm thử Backend (`api/tests/`)

### 2.1. Nhóm Nền tảng, Thực đơn, CORS & Tồn kho (`S1 + P3`)
1. `Tests\Unit\ExampleTest` & `Tests\Feature\ExampleTest` — Kiểm tra môi trường unit/health route.
2. `Tests\Unit\DataMaskingServiceTest` — Che mờ STK (`****5678`), CCCD (`********1234`), SĐT (`090***4567`).
3. `Tests\Feature\CorsTest` — Preflight `OPTIONS` cho `https://h5.zdn.vn` & chặn origin lạ.
4. `Tests\Feature\Menu\SchemaTest` — Giá nguyên VND & `restrictOnDelete`.
5. `Tests\Feature\Menu\MenuSeederTest` — `MenuSeeder` 6 danh mục, 30 món ăn, ảnh chuẩn & idempotent.
6. `Tests\Feature\Menu\MenuApiTest` — Danh mục active, lọc/tìm tiếng Việt, phân trang, chi tiết món `404`.
7. `Tests\Feature\Inventory\ProductInventoryTest` — Trừ/hoàn `stock_quantity` nguyên tử, `422 INSUFFICIENT_STOCK`, nhập kho Admin & chống ghi đè tồn kho đồng thời (`original_stock_quantity`).

### 2.2. Nhóm Xác thực & Phân quyền 4 Vai trò (`S2 + S3 + P1 + P3`)
8. `Tests\Feature\Auth\AuthApiTest` — Zalo OAuth (`appsecret_proof` HMAC-SHA256), Demo Login (bật/tắt), `/auth/me`.
9. `Tests\Feature\Auth\ZaloPhoneAuthTest` — Zalo Phone (`POST /auth/zalo-phone`), chuẩn hóa SĐT VN, `ACCOUNT_CONFLICT`, Demo Phone, Logout thu hồi token.
10. `Tests\Feature\Admin\RbacAndStatusLogSchemaTest` — `StaffSeeder`, `OrderStatusLog`, middleware `role`.
11. `Tests\Feature\Canteen\FourRoleRbacTest` — Ma trận phân quyền 4 vai trò (`admin`/`staff`/`parent`/`student`), chặn `403 PARENT_ROLE_REQUIRED` cho học sinh trên mọi route tài chính phụ huynh.

### 2.3. Nhóm Đơn hàng, Quản trị Bếp & Quầy POS (`S2 + S3 + S4 + P3`)
12. `Tests\Feature\Order\OrderSchemaTest` — Schema `users`, `orders`, `order_items`, `OrderStatus::allowedTransitions()`.
13. `Tests\Feature\Order\OrderApiTest` — Server-side pricing, món ngừng bán, `Idempotency-Key` (`201/200/409`), cô lập IDOR `404`, bảo toàn giá lịch sử.
14. `Tests\Feature\Admin\AdminOrderApiTest` — Danh sách/lọc đơn Admin, State Machine, Optimistic Locking `version` (`409`).
15. `Tests\Feature\Admin\AdminStatisticsApiTest` — Thống kê hôm nay & doanh thu đơn `COMPLETED`.
16. `Tests\Feature\Admin\AdminWebPortalTest` — Đăng nhập Blade, Dashboard, CRUD Danh mục/Món ăn (`role:admin`).
17. `Tests\Feature\Admin\StaffPosTerminalTest` — Tra cứu thẻ không lộ PII, thanh toán POS `#POS-...` (thẻ & tiền mặt), idempotency, biên lai in.

### 2.4. Nhóm Căn tin Số, Ngân hàng Sandbox, VNPay, Đối soát & Thông báo (`D1–D3 + P1–P4`)
18. `Tests\Feature\Canteen\DigitalCanteenBankLinkTest` — Thẻ học sinh, liên kết ngân hàng AES-256 + OTP 6 số, Partial Unique Index, lockout/cooldown.
19. `Tests\Feature\Canteen\SmsOtpGatewayTest` — `SmsGatewayInterface` (`log`/`esms`/`twilio`) & rollback sạch khi cổng SMS lỗi HTTP 500 (`503`).
20. `Tests\Feature\Canteen\SpendingLimitAndOrderPaymentTest` — Hạn mức ngày `LIMIT-01..13`, lazy reset `Asia/Ho_Chi_Minh`, thanh toán trừ hạn mức, POS, hoàn hạn mức khi hủy.
21. `Tests\Feature\Canteen\BankUnlinkOtpTest` — Hủy liên kết ngân hàng qua OTP `BANK_UNLINK`, bảo toàn hạn mức, lockout an toàn.
22. `Tests\Feature\Admin\AdminCanteenManagementTest` — Quản trị thẻ `/admin/canteen/cards`, khóa/mở/thu hồi thẻ tự ngắt ngân hàng, Admin override hạn mức.
23. `Tests\Feature\Admin\AdminStudentCrudTest` — CRUD Học sinh `/admin/canteen/students`, cấp thẻ thay thế, chặn chuyển phụ huynh khi còn `BankLink` mở.
24. `Tests\Feature\Canteen\SandboxBankingPaymentTest` — Trừ/hoàn số dư Sandbox, `INSUFFICIENT_BANK_BALANCE` (`422`), `BANK_DECLINED` (`502`), `GATEWAY_TIMEOUT` (`504`) + ghi bền vững sổ cái ngoài rollback.
25. `Tests\Feature\Canteen\BankingWebhookTest` — Xác thực chữ ký `HMAC-SHA256`, cửa sổ timestamp `±300s`, anti-replay `gateway_webhook_events`, `CHARGE_SETTLED`/`FAILED`/`REFUND_SETTLED`.
26. `Tests\Feature\Payment\VnpaySandboxTest` — Sinh URL ký `HMAC-SHA512`, IPN xác thực số tiền/mã lỗi/idempotency, fail-closed trên production.
27. `Tests\Feature\Admin\AdminReconciliationTest` — `ReconciliationService` 5 trạng thái khớp/lệch, CLI `canteen:reconcile`, Admin Blade resolve giữ nguyên `note` gốc.
28. `Tests\Feature\Canteen\NotificationModuleTest` — 7 sự kiện thông báo, IDOR `404`, đọc/đọc tất cả, `ZaloOaPushChannel` + `ZaloZnsPushChannel` resilient khi HTTP 500, lọc PII `meta`, chặn Open Redirect.
29. `Tests\Feature\Canteen\DemoShowcaseSeederTest` — Tính idempotent của 4 Seeder, thống kê hôm nay = `0`, kiểm tra đủ chỉ mục `pg_indexes` trên `canteen_notifications`.

### 2.5. Nhóm Kiểm thử Tích hợp E2E Toàn luồng
30. `Tests\Feature\EndToEndDemoWorkflowTest` — Luồng 8 bước trình diễn MVP (`AC-01..AC-11`).
31. `Tests\Feature\Canteen\DigitalCanteenEndToEndFlowsTest` — 4 luồng E2E Căn tin số (`D1–D3`).
32. `Tests\Feature\Canteen\ProductionReadinessEndToEndTest` — Luồng E2E liên hoàn toàn bộ Trục 3 (`P1 → P2 → P3 → P4`) + kiểm tra Security Headers & không rò rỉ PII.

---

## 3. Chi tiết 11 Bộ Kiểm thử Frontend (`miniapp/src/`, `44 tests`)

1. `src/lib/format.test.ts` (`1`) — Định dạng tiền tệ `formatVnd`.
2. `src/lib/api-client.test.ts` (`5`) — `apiGet`/`apiPost`/`apiPatch`, Bearer token, `Idempotency-Key`, `ApiError`.
3. `src/lib/use-focus-trap.test.ts` (`3`) — Vòng lặp phím `Tab`/`Escape` cho Modal A11y.
4. `src/components/product-image.test.ts` (`4`) — Chuẩn hóa URL ảnh & fallback logo.
5. `src/components/pagination-bar.test.ts` (`3`) — Chuẩn hóa phân trang & thuật toán cửa sổ trang.
6. `src/features/auth/phone.test.ts` (`3`) — Chuẩn hóa & kiểm tra SĐT Việt Nam.
7. `src/features/cart/cart-logic.test.ts` (`6`) — Thêm/sửa/xóa giỏ hàng, giới hạn tồn kho & `computeCartFingerprint`.
8. `src/features/canteen/validation.test.ts` (`6`) — Validate form liên kết ngân hàng theo từng ngân hàng & CCCD/SĐT.
9. `src/features/canteen/spending-limit.test.ts` (`6`) — Validate hạn mức bội số `1000`, tính `%` sử dụng & cảnh báo ngưỡng thấp.
10. `src/features/orders/payment.test.ts` (`4`) — Nhãn trạng thái thanh toán & `isSafeHttpUrl` cho VNPay.
11. `src/features/notifications/helpers.test.ts` (`3`) — `formatUnreadBadge` (`99+`), nhãn mức độ & `isSafeInternalRoute` chặn Open Redirect.
