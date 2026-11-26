# Trạng thái dự án — Smart Canteen (`PROJECT_STATE.md`)

## Tóm tắt chung

- **Dự án:** Smart Canteen (Zalo Mini App × React × Laravel 13 × PostgreSQL)
- **Nhánh hiện tại:** `feature/s4-docs-polish` (tách từ `main` commit `51601fc` sau khi đã merge S1, S2 và S3 vào `main`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

---

## Lát cắt S1 — Monorepo + Thực đơn (Menu Vertical Slice)

- **Mã tác vụ:** `S1-MENU-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED` & **Đã merge vào `main` (`6f44c9c`)**.

---

## Lát cắt S2 — Nhận diện thương hiệu, Xác thực, Giỏ hàng & Đặt đơn (Checkout Vertical Slice)

- **Mã tác vụ:** `S2-CHECKOUT-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED` & **Đã merge vào `main` (`d45ca09`)**.

---

## Lát cắt S3 — Trang quản trị Admin/Staff, RBAC, Điều phối đơn hàng, Thống kê & CRUD Thực đơn

- **Mã tác vụ:** `S3-ADMIN-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED` & **Đã merge vào `main` (`51601fc`)**.

---

## Lát cắt S4 — Tinh chỉnh RBAC/Validation, Kiểm thử E2E 8 Bước & Hồ sơ Tài liệu Kỹ thuật Bàn giao

- **Mã tác vụ:** `S4-DOCS-POLISH-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED`
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-14-s4-docs-polish-slice.md`
- **Các bước đã hoàn thành:**
  - [x] Task 1: Xử lý triệt để toàn bộ 3 điểm Deferred Minors từ Lát cắt S3 (`status` enum validation `422` + `limit` parameter `1..100` trên `GET /api/v1/admin/orders`; tự động dọn session `customer` trên `GET /admin/login`) và tách quyền RBAC `role:admin` cho các thao tác thêm/sửa/xóa Danh mục & Món ăn (trong khi `role:staff,admin` xem thực đơn và điều phối đơn).
  - [x] Task 2: Xây dựng bài kiểm thử tích hợp tự động `EndToEndDemoWorkflowTest` kiểm chứng xuyên suốt 8 bước trình diễn đầu-cuối (46 assertions) và các tiêu chí nghiệm thu AC-01..AC-11.
  - [x] Task 3: Hoàn thiện tài liệu Yêu cầu (`docs/requirements.md`, `docs/requirements/acceptance-criteria-matrix.md`), Kiến trúc (`docs/architecture/system-architecture.md`), Thiết kế UI/UX (`docs/design/ui-ux-brand-guidelines.md`) và Cơ sở dữ liệu (`docs/database.md`).
  - [x] Task 4: Hoàn thiện tài liệu Kế hoạch (`docs/plans/roadmap-and-slices.md`), Kịch bản & Báo cáo kiểm thử (`docs/test-cases.md`, `docs/testing/test-report.md`), Kịch bản trình diễn 8 bước (`docs/demo-script.md`) và Tài liệu phát hành / Runbook S5 (`docs/releases/release-v0.1.0-mvp.md`, `docs/releases/s5-staging-zalo-deploy-runbook.md`).

- **Kết quả cổng chất lượng (Quality Gates):**
  - `php artisan test` (PostgreSQL `titkul_canteen_test`): **PASS** — 14 test suites, 52 tests, 497 assertions.
  - `vendor\bin\pint --test`: **PASS** — `{"tool":"pint","result":"passed"}`.
  - `npm test` (Vitest): **PASS** — 5 test files, 18 tests.
  - `npx tsc --noEmit`: **PASS** — 0 lỗi kiểu dữ liệu.
  - `npm run build` (Vite production build): **PASS** — tạo bundle tại `miniapp/www/`.

- **Công việc tiếp theo (S5 — Khi được cấp quyền triển khai):**
  - Thực thi `docs/releases/s5-staging-zalo-deploy-runbook.md` để mở HTTPS tunnel/staging và deploy phiên bản Zalo Mini App Testing lấy mã QR cho Giám đốc.

---

## Trục 1 — Tách 2 Sub-repo Git Độc lập, Docker hóa & GitHub Actions CI

- **Mã tác vụ:** `PLATFORM-SUBREPOS-DOCKER-CI`
- **Trạng thái:** `COMPLETED_VERIFIED` (chờ người dùng duyệt merge/push trên từng repo)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-subrepos-docker-ci-design.md` (APPROVED 2026-11-25)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-truc1-subrepos-docker-ci.md`
- **Nhánh hiện tại:**
  - Workspace cha: `feat/subrepos-docker-ci` (đã detach `api/` và `miniapp/` khỏi index cha, giữ nguyên lịch sử `main`)
  - Sub-repo `api/`: `feat/docker-ci` (tách từ `main @ 1532dd0` giữ trọn lịch sử S1–S5)
  - Sub-repo `miniapp/`: `feat/docker-ci` (tách từ `main @ 142dd27` giữ trọn lịch sử S1–S5)
- **Các bước đã triển khai:**
  - [x] Tách `api/` và `miniapp/` thành 2 Git repository độc lập via `git subtree split`, bảo toàn `.env`, `vendor/`, `node_modules/`, `FEATURES.md`, và video gốc.
  - [x] Bổ sung bộ Docker multi-stage chuẩn Production-Ready cho `api/` (`Dockerfile`, `compose.yaml`, `compose.prod.yaml`, Nginx unprivileged + immutable `docker/nginx/Dockerfile`, PHP 8.4 FPM non-root `appuser`, PostgreSQL 17 tự khởi tạo 2 DB demo/test).
  - [x] Bổ sung bộ Docker multi-stage chuẩn Production-Ready cho `miniapp/` (`Dockerfile`, `compose.yaml`, `compose.prod.yaml`, Nginx unprivileged + `/healthz`).
  - [x] Bổ sung GitHub Actions CI độc lập tại `api/.github/workflows/ci.yml` và `miniapp/.github/workflows/ci.yml`.
  - [x] Cập nhật tài liệu `api/README.md`, `miniapp/README.md`, `README.md`.
- **Kết quả cổng chất lượng (Quality Gates — Đã kiểm chứng thực tế):**
  - `api/` (chạy trong Docker PHP 8.4 + PostgreSQL 17): `composer validate --strict` PASS, `composer audit --locked` 0 lỗ hổng, `vendor/bin/pint --test` PASS (89 files), `php artisan test --compact` PASS (**53 passed, 506 assertions**), Nginx `/up` trả `200`, production image non-root `appuser` & không có `composer`.
  - `miniapp/`: `npm test` PASS (5 files, 18 tests), `npx tsc --noEmit` PASS, `npm run build` PASS, `npm audit --omit=dev --audit-level=high` 0 lỗ hổng, Docker production image non-root UID `101`, `/healthz` trả `ok`, `/` trả `200`.
  - `compose.yaml` & `compose.prod.yaml` của cả 2 sub-repo PASS `docker compose config --quiet`; cả 2 workflow CI PASS YAML parser.
- **Lộ trình tiếp theo (Trục 2 — Phân hệ Căn tin Số theo `FEATURES.md`, phương án `3A`):**
  - Slice D1: **COMPLETED_VERIFIED & Đã merge vào `main` (`4c76e83` / `fa9418a` / `01c8db3`)**.
  - Slice D2: **COMPLETED_VERIFIED & Đã merge vào `main` (`941014c` / `89d76d1` / `14a433b`)**.
  - Slice D3: **COMPLETED_VERIFIED** (chờ người dùng duyệt merge/push trên từng repo).

---

## Trục 2 · Slice D1 — Hồ sơ Học sinh, Thẻ Thông minh & Liên kết Ngân hàng + OTP

- **Mã tác vụ:** `CANTEEN-D1-CARD-BANK-OTP`
- **Trạng thái:** `MERGED_TO_MAIN` (`root @ 4c76e83`, `api @ fa9418a`, `miniapp @ 01c8db3`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-d1-student-card-bank-link-design.md` (APPROVED 2026-11-25)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-d1-student-card-bank-link.md`
- **Nhánh hiện tại:** `feat/d1-student-card-bank-link` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 4 migrations PostgreSQL (kèm Partial Unique Index `WHERE status = 'ACTIVE'`), 3 Enums, 4 Models (`encrypted` cast cho STK/CCCD), `StudentCardSeeder`, `BankingGatewayInterface` + `MockEpayBankingGateway`, `DataMaskingService`, `StudentCardService`, `BankLinkingService`, 6 REST endpoints `/api/v1/canteen/*`.
  - [x] Frontend `miniapp/`: Mở rộng `BottomNav` thành 4 tab, bổ sung `StudentCardView`, `OtpCodeInput`, 3 trang `/canteen`, `/canteen/link-bank`, `/canteen/verify-otp` kèm Jotai in-memory draft & validation.
  - [x] Khắc phục 100% các điểm Critical & Important từ Code Review độc lập (chặn lách lockout qua `resend-otp`, sửa chỉnh chữ số giữa `OtpCodeInput`, chuẩn hóa thứ tự khóa `student_cards → bank_links → otp_verifications`, đưa `catch (UniqueConstraintViolationException)` ra ngoài `DB::transaction()`, wall-clock countdown & double-submit guards).

---

## Trục 2 · Slice D2 — Hạn mức Chi tiêu Hằng ngày, Lịch sử Hạn mức, Thanh toán Trừ Hạn mức & Chi tiết Đơn Gắn Thẻ

- **Mã tác vụ:** `CANTEEN-D2-LIMIT-PAYMENT-ORDERS`
- **Trạng thái:** `MERGED_TO_MAIN` (`root @ 941014c`, `api @ 89d76d1`, `miniapp @ 14a433b`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-d2-spending-limit-orders-design.md` (APPROVED 2026-11-25)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-d2-spending-limit-orders.md`
- **Nhánh hiện tại:** `feat/d2-spending-limit-orders` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 3 migrations additive (`spending_limits`, `spending_limit_logs`, mở rộng `orders`), `PaymentMethod`/`PaymentStatus`, `SpendingLimitService` (lazy daily reset `Asia/Ho_Chi_Minh`), mở rộng `OrderService` (`createOrder` hỗ trợ thẻ học sinh, `createPosPurchase` `#POS-...`, `transitionStatus` tự hoàn hạn mức theo thứ tự khóa chuẩn `student_cards → spending_limits → orders`), 4 endpoints mới `/api/v1/canteen/cards/{cardId}/{spending-limit,limit-logs,orders,pos-purchases}`, + xử lý Slice D1 Backend Minors.
  - [x] Frontend `miniapp/`: `SpendingLimitSheet` (`20k/30k/50k/100k` + tùy chỉnh), `LimitHistoryPage` & `CartPage` đồng bộ `selectedCardIdAtom` đa thẻ, nâng cấp `CanteenHubPage` (hạn mức + đơn hàng gần đây + mô phỏng quẹt thẻ POS), `OrderDetailPage` (đủ 14 trường `DETAIL-01..14`), + xử lý Slice D1 Frontend A11y Minors.

---

## Trục 2 · Slice D3 — Hủy Liên kết Ngân hàng qua OTP, Quản trị Blade Căn tin Số, 7 Deferred Minors & E2E 4 Luồng

- **Mã tác vụ:** `CANTEEN-D3-UNLINK-ADMIN-E2E`
- **Trạng thái:** `MERGED_TO_MAIN` (`root @ fc93101`, `api @ c9caf98`, `miniapp @ b73ce24`; Minor Polish merged `9549812` / `22046ee` / `ca2e40d`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-d3-unlink-admin-e2e-design.md` (APPROVED 2026-11-25)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-d3-unlink-admin-e2e.md`
- **Nhánh hiện tại:** `feat/d3-unlink-admin-e2e` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 3 endpoints hủy liên kết OTP `/api/v1/canteen/bank-links/{id}/unlink/{initiate,verify-otp,resend-otp}` (`throttle:10,1`, canonical lock `student_cards → bank_links → otp_verifications`), Quản trị Blade Căn tin số (`/admin/canteen/cards`, `/admin/canteen/cards/{card}`, chuyển trạng thái thẻ, điều chỉnh hạn mức Admin, nâng cấp Dashboard & Orders Blade), 3 Deferred Minors Backend, + 3 test suites mới (`BankUnlinkOtpTest`, `AdminCanteenManagementTest`, `DigitalCanteenEndToEndFlowsTest`).
  - [x] Frontend `miniapp/`: `UnlinkConfirmDialog`, `BankUnlinkOtpPage` (`/canteen/unlink-otp`) kèm màn hình xác nhận Hủy liên kết thành công (Screen #15), nâng cấp `CanteenHubPage`, + 3 Deferred Minors Frontend (A11y, focus restore, `Asia/Ho_Chi_Minh` timezone).
  - [x] Tài liệu `docs/`: cập nhật `api-contract.md`, `database.md` (Minor #7 ERD & tiêu đề), `PROJECT_STATE.md`.

---

## Trục 3 · Slice P1 — Xác thực SĐT Zalo (Lazy Auth) & Cổng SMS OTP Thực tế

- **Mã tác vụ:** `CANTEEN-P1-ZALO-PHONE-SMS-OTP`
- **Trạng thái:** `MERGED_TO_MAIN` (`root @ 8bd8340`, `api @ 63e1d02`, `miniapp @ c7889be`)
- **Lộ trình Trục 3:** `docs/superpowers/specs/2026-11-25-truc3-production-readiness-roadmap.md` (`APPROVED 2026-11-25`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-p1-zalo-phone-auth-sms-otp-design.md` (`APPROVED 2026-11-25`)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-p1-zalo-phone-auth-sms-otp.md`
- **Nhánh hiện tại:** `feat/p1-zalo-phone-auth-sms-otp` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 2 migrations additive (`users.phone` + `phone_verified_at`, `otp_verifications.delivery_*`), `PhoneNormalizer`, `ZaloIdentityService::resolvePhoneNumber()`, 3 endpoints `/api/v1/auth/{zalo-phone,demo-phone,logout}`, `SmsGatewayInterface` với 3 driver (`log`, `esms`, `twilio`), tích hợp rollback sạch vào `BankLinkingService`, + 2 test suites mới (`ZaloPhoneAuthTest`, `SmsOtpGatewayTest`).
  - [x] Frontend `miniapp/`: Kiến trúc **Lazy Auth** (Thực đơn/Chi tiết/Giỏ hàng Public 100%; Căn tin/Đơn hàng/Đặt đơn yêu cầu xác thực SĐT Zalo), `ZaloPhoneLoginPanel` & `ZaloPhoneLoginModal`, auto-fill SĐT phụ huynh ở `BankLinkFormPage`, nút Đăng xuất ở `CanteenHubPage`, + unit test `phone.test.ts`.

---

## Trục 3 · Slice P2 — `SandboxBankingGateway`, `VNPay Sandbox`, Sổ cái `payment_transactions`, Webhook HMAC-SHA256 & Đối soát Ngân hàng

- **Mã tác vụ:** `CANTEEN-P2-SANDBOX-VNPAY-RECONCILIATION`
- **Trạng thái:** `MERGED_TO_MAIN` (`root @ 120e480`, `api @ 3ab9af5`, `miniapp @ 7a2d4e5`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-p2-sandbox-banking-vnpay-reconciliation-design.md` (`APPROVED 2026-11-25`)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-p2-sandbox-vnpay-reconciliation.md`
- **Nhánh hiện tại:** `feat/p2-sandbox-vnpay-reconciliation` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 4 migrations additive (`sandbox_bank_accounts`, `payment_transactions`, `reconciliation_batches`/`items`, `gateway_webhook_events`), `SandboxBankingGateway`, `PaymentLedgerService`, `BankingWebhookController` (HMAC-SHA256 + anti-replay), `VnpayGatewayService` & `VnpayPaymentController` (HMAC-SHA512 + IPN idempotent), `ReconciliationService` + CLI `canteen:reconcile`, Admin Blade `/admin/canteen/reconciliation` + khung Sandbox Account trên chi tiết thẻ, + 4 test suites mới & Slice P1 Minor SMS rollback test.
  - [x] Frontend `miniapp/`: phương thức **Thanh toán VNPay (Sandbox)** trên `CartPage`, nút mở cổng VNPay & badge `FAILED`/`PENDING_RECONCILE` trên `OrderDetailPage`, unit test `payment.test.ts`, + Slice P1 Minor `VITE_DEMO_PHONE_LOGIN`.

---

## Trục 3 · Slice P3 — POS Quầy Căn tin (Staff), Quản lý Tồn kho (`stock_quantity`), CRUD Học sinh & RBAC 4 Vai trò

- **Mã tác vụ:** `CANTEEN-P3-POS-INVENTORY-STUDENTS-RBAC`
- **Trạng thái:** `MERGED_TO_MAIN` (`root @ 42e1146`, `api @ f2a5625`, `miniapp @ 3044130`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-p3-pos-inventory-students-rbac-design.md` (`APPROVED 2026-11-25`)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-p3-pos-inventory-students-rbac.md`
- **Nhánh hiện tại:** `feat/p3-pos-inventory-students-rbac` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 2 migrations additive (`products.stock_quantity`/`low_stock_threshold`, `students.student_user_id`), `InsufficientStockException` & trừ/hoàn kho nguyên tử trong `OrderService` theo đúng thứ tự khóa chuẩn, `PosTerminalController` + Blade `/admin/pos` & biên lai in, `StudentManagementController` + Blade `/admin/canteen/students` & cấp phát thẻ, nhập kho nhanh `/admin/products/{product}/stock`, RBAC 4 vai trò (`admin`/`staff`/`parent`/`student`, tương thích ngược `customer`), `StudentCardSeeder` có sẵn tài khoản Học sinh demo `0901234568`, + P2 Minor `gateway_txn_ref` cho `TIMEOUT`, + 4 bộ test mới (`ProductInventoryTest`, `StaffPosTerminalTest`, `AdminStudentCrudTest`, `FourRoleRbacTest`).
  - [x] Frontend `miniapp/`: hiển thị & giới hạn số lượng theo tồn kho ở `MenuPage`/`ProductDetailPage`/`CartPage`, chế độ Học sinh chỉ-xem trên `CanteenHubPage` + nút SĐT demo học sinh `0901234568`, + 2 P2 Minors (badge trạng thái thanh toán trên `OrdersPage` & điều hướng phím mũi tên WAI-ARIA cho `role="radiogroup"` trên `CartPage`).

---

## Trục 3 · Slice P4 — Notification Module (In-App + Push Zalo OA/ZNS) & Security/E2E Audit Toàn Hệ thống

- **Mã tác vụ:** `CANTEEN-P4-NOTIFICATIONS-SECURITY-E2E`
- **Trạng thái:** `COMPLETED_VERIFIED` (chờ người dùng duyệt merge cục bộ vào `main` trên cả 3 repo)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-26-p4-notifications-security-e2e-design.md` (`APPROVED 2026-11-26`)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-26-p4-notifications-security-e2e.md`
- **Nhánh hiện tại:** `feat/p4-notifications-security-e2e` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: Migration `canteen_notifications`, `PushNotificationChannelInterface` (`log`/`zalo_oa`/`zalo_zns`), `NotificationService` gắn 7 sự kiện tự động (fail-safe sau commit), REST API `/api/v1/notifications/*`, Admin Blade `/admin/canteen/notifications`, `SecurityHeadersMiddleware`, + P3 Minors (khóa metadata/tồn kho & prepend đơn POS mới), + 2 bộ test mới (`NotificationModuleTest`, `ProductionReadinessEndToEndTest`).
  - [x] Frontend `miniapp/`: Trung tâm Thông báo `/notifications` (Lazy Auth, lọc chưa đọc, đọc tất cả, `isSafeInternalRoute`), `BottomNav` 5 tab kèm badge chưa đọc, + P3 Minors (roving `tabIndex`, câu chữ chuẩn vai trò học sinh, snackbar chạm trần tồn kho).
  - [x] Tài liệu `docs/`: báo cáo kiểm toán `docs/testing/truc3-security-e2e-audit.md`, `api-contract.md` (§11), `database.md` (§7), `PROJECT_STATE.md`.
