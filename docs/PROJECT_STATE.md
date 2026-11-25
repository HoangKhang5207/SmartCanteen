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
- **Trạng thái:** `COMPLETED_VERIFIED` (chờ người dùng duyệt merge/push trên từng repo)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-25-d3-unlink-admin-e2e-design.md` (APPROVED 2026-11-25)
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-25-d3-unlink-admin-e2e.md`
- **Nhánh hiện tại:** `feat/d3-unlink-admin-e2e` trên cả 3 repository (`root`, `api/`, `miniapp/`)
- **Phạm vi đã hiện thực hóa:**
  - [x] Backend `api/`: 3 endpoints hủy liên kết OTP `/api/v1/canteen/bank-links/{id}/unlink/{initiate,verify-otp,resend-otp}` (`throttle:10,1`, canonical lock `student_cards → bank_links → otp_verifications`), Quản trị Blade Căn tin số (`/admin/canteen/cards`, `/admin/canteen/cards/{card}`, chuyển trạng thái thẻ, điều chỉnh hạn mức Admin, nâng cấp Dashboard & Orders Blade), 3 Deferred Minors Backend, + 3 test suites mới (`BankUnlinkOtpTest`, `AdminCanteenManagementTest`, `DigitalCanteenEndToEndFlowsTest`).
  - [x] Frontend `miniapp/`: `UnlinkConfirmDialog`, `BankUnlinkOtpPage` (`/canteen/unlink-otp`) kèm màn hình xác nhận Hủy liên kết thành công (Screen #15), nâng cấp `CanteenHubPage`, + 3 Deferred Minors Frontend (A11y, focus restore, `Asia/Ho_Chi_Minh` timezone).
  - [x] Tài liệu `docs/`: cập nhật `api-contract.md`, `database.md` (Minor #7 ERD & tiêu đề), `PROJECT_STATE.md`.
