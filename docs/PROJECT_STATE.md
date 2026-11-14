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
  - `php artisan test` (PostgreSQL `titkul_canteen_test`): **PASS** — 14 test suites, 49 tests, 344 assertions.
  - `vendor\bin\pint --test`: **PASS** — `{"tool":"pint","result":"passed"}`.
  - `npm test` (Vitest): **PASS** — 3 test files, 11 tests.
  - `npx tsc --noEmit`: **PASS** — 0 lỗi kiểu dữ liệu.
  - `npm run build` (Vite production build): **PASS** — tạo bundle tại `miniapp/www/`.

- **Công việc tiếp theo (S5 — Khi được cấp quyền triển khai):**
  - Thực thi `docs/releases/s5-staging-zalo-deploy-runbook.md` để mở HTTPS tunnel/staging và deploy phiên bản Zalo Mini App Testing lấy mã QR cho Giám đốc.
