# Trạng thái dự án — Smart Canteen (`PROJECT_STATE.md`)

## Tóm tắt chung

- **Dự án:** Smart Canteen (Zalo Mini App × React × Laravel 13 × PostgreSQL)
- **Nhánh hiện tại:** `feature/s3-admin` (tách từ `main` commit `d45ca09` sau khi đã merge S1 và S2 vào `main`)
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
- **Trạng thái:** `COMPLETED_VERIFIED`
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-14-s3-admin-slice.md`
- **Các bước đã hoàn thành:**
  - [x] Task 1: Bảng `order_status_logs` & model `OrderStatusLog`, middleware phân quyền `EnsureUserHasRole` (`role:staff,admin`), và `StaffSeeder` tạo tài khoản `admin@smartcanteen.local` / `staff@smartcanteen.local` idempotent.
  - [x] Task 2: `OrderService::transitionStatus()` xử lý chuyển trạng thái theo State Machine + Optimistic Locking (`version`, trả `409 Conflict` khi xung đột) + ghi `order_status_logs` trong `DB::transaction`, cùng REST API `GET /api/v1/admin/orders` và `PATCH /api/v1/admin/orders/{id}/status`.
  - [x] Task 3: `StatisticsService::getTodaySummary()` và API `GET /api/v1/admin/statistics` tính tổng số đơn hôm nay, số đơn đang xử lý, doanh thu các đơn `COMPLETED` hôm nay và phân bổ theo từng trạng thái.
  - [x] Task 4: Cổng quản trị Laravel Blade (`/admin/login`, `/admin`, `/admin/orders`, `/admin/orders/{order}`, `/admin/categories`, `/admin/products`) mang nhận diện thương hiệu Smart Canteen, tích hợp tự động làm mới 10 giây, chuyển trạng thái kèm `version`, và CRUD Danh mục / Món ăn bảo toàn lịch sử đơn hàng (**AC-11**).
  - [x] Task 5: Cập nhật tài liệu (`docs/api-contract.md`, `README.md`, `docs/PROJECT_STATE.md`) và chạy toàn bộ cổng chất lượng.

- **Kết quả cổng chất lượng (Quality Gates):**
  - `php artisan test` (PostgreSQL `titkul_canteen_test`): **PASS** — 12 test suites, 45 tests, 273 assertions.
  - `vendor\bin\pint --test`: **PASS** — `{"tool":"pint","result":"passed"}`.
  - `npm test` (Vitest): **PASS** — 3 test files, 11 tests.
  - `npx tsc --noEmit`: **PASS** — 0 lỗi kiểu dữ liệu.
  - `npm run build` (Vite production build): **PASS** — tạo bundle tại `miniapp/www/`.

- **Công việc tiếp theo (S4):**
  - Hoàn thiện bộ tài liệu kỹ thuật bàn giao (`docs/requirements.md`, `docs/database.md`, `docs/test-cases.md`, `docs/demo-script.md`), rà soát toàn diện các tiêu chí nghiệm thu P0/P1 (AC-01..AC-13) và kịch bản trình diễn 8 bước end-to-end.
