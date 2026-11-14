# S4 — RBAC/Validation Polish, End-to-End 8-Step Verification & Full Technical Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Xử lý triệt để toàn bộ 3 điểm Rulings & Deferred Minors từ Lát cắt S3 cùng phân quyền RBAC chi tiết giữa `staff` và `admin`, kiểm chứng tự động kịch bản trình diễn 8 bước đầu-cuối (End-to-End Demo Workflow) và hoàn thiện trọn bộ hồ sơ tài liệu kỹ thuật bàn giao (`docs/requirements.md`, `docs/database.md`, `docs/test-cases.md`, `docs/demo-script.md`, `docs/architecture/`, `docs/design/`, `docs/plans/`, `docs/testing/`, `docs/releases/`).

**Architecture:** Tinh chỉnh tầng Controller/Validation của Laravel 13 (`Api\V1\Admin\OrderController`, `Admin\AuthController`, `routes/web.php` tách nhóm quyền `role:admin` cho thao tác ghi/xóa thực đơn khỏi `role:staff,admin` cho điều phối đơn & xem thực đơn), bổ sung bài kiểm thử tự động mô phỏng trọn vẹn 8 bước demo từ Mini App sang Admin và ngược lại, đồng thời biên soạn đầy đủ các tài liệu kiến trúc, cơ sở dữ liệu, kiểm thử, kịch bản trình diễn và phát hành.

**Tech Stack:** PHP 8.5.12, Laravel 13, Blade, Sanctum, PostgreSQL 17.5, PHPUnit; React 18, TypeScript, ZaUI, Jotai, TanStack Query, Vitest.

**Spec:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

## Global Constraints
- Xử lý triệt để 3 điểm Deferred Minors từ Báo cáo Bàn giao Lát cắt S3:
  1. `GET /api/v1/admin/orders?status=...`: chuẩn hóa chữ hoa và validate chặt bằng `Rule::enum(OrderStatus::class)` — nếu truyền `status` không hợp lệ (ví dụ `?status=INVALID`) phải trả `422 Unprocessable Entity` thay vì âm thầm bỏ qua bộ lọc.
  2. `GET /api/v1/admin/orders`: áp dụng giới hạn an toàn `->limit($limit)` (mặc định `100`, cho phép `?limit=1..100`) để bảo vệ bộ nhớ và hiệu năng khi eager-load `['user', 'items.product', 'statusLogs.actor']`.
  3. `GET /admin/login` (`Admin\AuthController::showLogin`): nếu phiên web hiện tại đang đăng nhập bằng tài khoản không có quyền quản trị (`role = customer`), tự động đăng xuất (`Auth::logout()`, invalidate session) và hiển thị trang đăng nhập `/admin/login` (`200 OK`) thay vì redirect vòng lặp sang `/admin` (`403`).
- Tinh chỉnh RBAC chi tiết (**Section 1.B** của Spec):
  - `staff` và `admin`: truy cập Dashboard (`/admin`), Điều phối đơn hàng (`/admin/orders*`, `/api/v1/admin/*`), và xem danh sách Món ăn / Danh mục (`GET /admin/products`, `GET /admin/categories`).
  - Chỉ `admin` (`role:admin`): được phép thêm/sửa/xóa Danh mục và Món ăn (`POST/PUT/DELETE /admin/categories*`, `GET create/edit` & `POST/PUT/DELETE /admin/products*`). Nếu `staff` cố tình gọi các route chỉnh sửa thực đơn sẽ nhận `403 Forbidden`.
- Kiểm chứng đầy đủ các tiêu chí nghiệm thu **AC-01..AC-13** (AC-14..AC-16 thuộc S5 Deployment đang ở trạng thái `NOT AUTHORIZED`, được tài liệu hóa rõ trong `docs/releases/` và `docs/demo-script.md`).

## Review Focus
1. Gọi `GET /api/v1/admin/orders?status=INVALID_STATUS` → trả `422 Unprocessable Entity` kèm lỗi tại trường `status`; gọi `?status=pending` (chữ thường) → tự chuẩn hóa thành `PENDING` và trả `200 OK`. Test ở Task 1.
2. Gọi `GET /api/v1/admin/orders?limit=5` → trả tối đa 5 đơn hàng; gọi `?limit=500` → trả `422 Unprocessable Entity` (vượt ngưỡng tối đa 100). Test ở Task 1.
3. Một `customer` đang có session cookie truy cập `GET /admin/login` → hệ thống tự động đăng xuất session `customer` đó và trả về trang đăng nhập `200 OK` thay vì redirect sang `/admin` bị lỗi `403`. Test ở Task 1.
4. Nhân viên bếp (`role = staff`) xem được `GET /admin/products` và `GET /admin/categories` (`200 OK`), nhưng khi gửi `POST /admin/products`, `PUT /admin/products/{id}`, `DELETE /admin/products/{id}` hoặc `POST /admin/categories` → nhận `403 Forbidden`. Test ở Task 1.
5. Luồng trình diễn 8 bước (Khách đăng nhập → xem menu → tạo đơn kèm `Idempotency-Key` và giá giả mạo → Admin thấy đơn `PENDING` → chuyển `PREPARING` → chuyển `READY` → Khách polling thấy `READY` và xem lịch sử đơn → Admin chuyển `COMPLETED` và kiểm tra doanh thu hôm nay) chạy xuyên suốt trong một bài kiểm thử tích hợp tự động. Test ở Task 2.

---

### Task 1: Xử lý triệt để 3 Deferred Minors từ S3 & Tinh chỉnh RBAC (`staff` vs `admin` trên Menu CRUD)

**Files:**
- Modify: `api/app/Http/Controllers/Api/V1/Admin/OrderController.php`
- Modify: `api/app/Http/Controllers/Admin/AuthController.php`
- Modify: `api/routes/web.php`
- Modify: `api/resources/views/admin/products/index.blade.php`, `api/resources/views/admin/categories/index.blade.php`
- Modify: `api/tests/Feature/Admin/AdminOrderApiTest.php`, `api/tests/Feature/Admin/AdminWebPortalTest.php`

**Interfaces — Produces:**
- `GET /api/v1/admin/orders`:
  - Chuẩn hóa `status` sang `strtoupper(trim($status))` trước khi validate với `Rule::enum(OrderStatus::class)` (trả `422` nếu sai).
  - Hỗ trợ `limit` (`nullable`, `integer`, `min:1`, `max:100`, mặc định `100`).
- `Admin\AuthController::showLogin`:
  - Nếu `Auth::check()`: chỉ redirect `/admin` khi user có `role` thuộc `[UserRole::STAFF, UserRole::ADMIN]`; nếu là `customer` thì `Auth::logout()`, `$request->session()->invalidate()`, `$request->session()->regenerateToken()` và trả về `view('admin.login')`.
- `routes/web.php`:
  - `GET /admin/categories` và `GET /admin/products` nằm trong nhóm `role:staff,admin` (để nhân viên bếp tra cứu thực đơn).
  - Các route tạo/sửa/xóa (`POST/PUT/DELETE /admin/categories*`, `GET /admin/products/create`, `POST /admin/products`, `GET /admin/products/{product}/edit`, `PUT/DELETE /admin/products/{product}`) được bảo vệ thêm bằng middleware `role:admin`.

- [x] Bổ sung các test cases mới vào `AdminOrderApiTest` và `AdminWebPortalTest`:
  - `test_invalid_status_filter_or_excessive_limit_on_admin_orders_returns_422`
  - `test_authenticated_customer_visiting_admin_login_is_logged_out_and_sees_login_form`
  - `test_staff_can_view_menu_lists_but_only_admin_can_mutate_categories_and_products`
- [x] Chạy test → FAIL. Hiện thực hóa các thay đổi trong `OrderController`, `AuthController`, `routes/web.php`, và ẩn nút Thêm/Sửa/Xóa trên giao diện Blade khi user là `staff`. Chạy test → PASS. Commit `fix(admin): resolve s3 deferred minors and enforce admin-only menu mutations`.

### Task 2: Kiểm thử Tự động Luồng Trình diễn 8 Bước End-to-End (`EndToEndDemoWorkflowTest`)

**Files:**
- Create: `api/tests/Feature/EndToEndDemoWorkflowTest.php`

**Interfaces — Produces:**
- Bài kiểm thử tích hợp `EndToEndDemoWorkflowTest` kiểm chứng liền mạch toàn bộ 8 bước trình diễn trong `BaiTestPart2.md` (Mục 4) và các tiêu chí nghiệm thu AC-01..AC-11 trên PostgreSQL:
  1. Seed dữ liệu chuẩn (`MenuSeeder` + `StaffSeeder`) → kiểm tra `GET /api/v1/categories` (>= 3 danh mục) và `GET /api/v1/products` (>= 10 món ăn) (**AC-01**, **AC-02**).
  2. Khách hàng đăng nhập và lấy chi tiết món ăn cần đặt.
  3. Khách hàng gửi `POST /api/v1/orders` kèm `Idempotency-Key` và cố tình gửi `total_amount = 1` giả mạo; bấm gửi đúp lần 2 với cùng `Idempotency-Key` → chỉ tạo đúng 1 đơn hàng với tổng tiền tính chuẩn từ DB (**AC-04**, **AC-06**).
  4. Nhân viên đăng nhập và gọi `GET /api/v1/admin/orders?status=PENDING` → nhìn thấy đơn hàng mới vừa tạo (**AC-07**).
  5. Nhân viên chuyển trạng thái đơn từ `PENDING` (`v1`) → `PREPARING` (`v2`), sau đó từ `PREPARING` (`v2`) → `READY` (`v3`); kiểm tra xung đột `version` cũ (`409`) và chuyển trạng thái sai (`422`) (**AC-08**).
  6. Khách hàng gọi `GET /api/v1/orders/{id}` (polling) và `GET /api/v1/orders` → nhìn thấy trạng thái đơn đã cập nhật sang `READY` (`v3`) cùng mã đơn `SC-...`, trong khi khách hàng khác truy cập đơn này bị chặn `404` (**AC-09**, **AC-10**).
  7. Nhân viên xác nhận khách đã nhận món (`READY` `v3` → `COMPLETED` `v4`); Admin đổi giá món ăn trên hệ thống → đơn hàng lịch sử không đổi giá (**AC-11**) và `GET /api/v1/admin/statistics` phản ánh đúng doanh thu `COMPLETED` hôm nay.

- [x] Viết và chạy `php artisan test --filter=EndToEndDemoWorkflowTest` → PASS. Commit `test(api): add full 8-step end-to-end demo workflow integration test`.

### Task 3: Bộ Tài liệu Yêu cầu, Kiến trúc, Thiết kế & Cơ sở dữ liệu (`docs/requirements*`, `docs/architecture/`, `docs/design/`, `docs/database.md`)

**Files:**
- Create: `docs/requirements.md`, `docs/requirements/acceptance-criteria-matrix.md`
- Create: `docs/architecture/system-architecture.md`
- Create: `docs/design/ui-ux-brand-guidelines.md`
- Create: `docs/database.md`

- [x] Biên soạn đầy đủ, chính xác theo đúng thực tế mã nguồn và cấu trúc DB PostgreSQL hiện hành:
  - `docs/requirements.md` & `docs/requirements/acceptance-criteria-matrix.md`: Yêu cầu nghiệp vụ P0/P1/P2, quy tắc nghiệp vụ cốt lõi, và ma trận ánh xạ từng tiêu chí nghiệm thu **AC-01..AC-16** tới file mã nguồn + bài test kiểm chứng.
  - `docs/architecture/system-architecture.md`: Kiến trúc tổng thể Monorepo (`miniapp/` × `api/` × PostgreSQL), sơ đồ luồng xác thực Zalo Open API (`appsecret_proof`), luồng đặt đơn Idempotent & State Machine kèm Optimistic Locking.
  - `docs/design/ui-ux-brand-guidelines.md`: Hệ thống thiết kế trích xuất từ `canteen_logo.jpg`, bảng màu chuẩn WCAG 2.1 AA (`#5DBE46` / `#2E7D32`), quy chuẩn 4 trạng thái màn hình (Loading, Error, Empty, Success) và điều hướng.
  - `docs/database.md`: Sơ đồ thực thể kết hợp (ERD Mermaid), mô tả chi tiết 7 bảng dữ liệu (`users`, `categories`, `products`, `orders`, `order_items`, `order_status_logs`, `personal_access_tokens`), ràng buộc khóa ngoại, chỉ mục và dữ liệu seed mẫu.
- [x] Kiểm tra liên kết và nội dung → Commit `docs: add requirements, architecture, design, and database documentation`.

### Task 4: Bộ Tài liệu Kế hoạch, Kiểm thử, Kịch bản Trình diễn & Phát hành (`docs/plans/`, `docs/testing/`, `docs/test-cases.md`, `docs/demo-script.md`, `docs/releases/`)

**Files:**
- Create: `docs/plans/roadmap-and-slices.md`
- Create: `docs/test-cases.md`, `docs/testing/test-report.md`
- Create: `docs/demo-script.md`
- Create: `docs/releases/release-v0.1.0-mvp.md`, `docs/releases/s5-staging-zalo-deploy-runbook.md`
- Modify: `README.md`, `docs/PROJECT_STATE.md`, `docs/api-contract.md`, `docs/superpowers/plans/2026-11-14-s4-docs-polish-slice.md`

- [x] Biên soạn đầy đủ:
  - `docs/plans/roadmap-and-slices.md`: Lộ trình 7 ngày và tổng kết 4 lát cắt dọc S1–S4 đã hoàn thành cùng kế hoạch S5.
  - `docs/test-cases.md` & `docs/testing/test-report.md`: Danh mục kịch bản kiểm thử chi tiết (Test Cases ID, tiền điều kiện, các bước, kết quả mong đợi, kết quả thực tế) và báo cáo kết quả thực thi tự động trên PostgreSQL + Vitest.
  - `docs/demo-script.md`: Kịch bản trình diễn 8 bước chi tiết dành cho buổi báo cáo với Giám đốc (bao gồm cả lời dẫn, thao tác trên 2 cửa sổ Mini App & Admin, cách demo chống đơn trùng, tính tiền server, đổi trạng thái realtime và bảo toàn giá lịch sử).
  - `docs/releases/release-v0.1.0-mvp.md` & `docs/releases/s5-staging-zalo-deploy-runbook.md`: Ghi chú phát hành `v0.1.0` và hướng dẫn chi tiết từng bước để triển khai S5 (Cloudflare Tunnel / HTTPS Staging + Zalo Mini App Testing QR Code) ngay khi được cấp quyền (`Deployment: NOT AUTHORIZED` ở hiện tại).
- [x] Cập nhật `README.md`, `docs/PROJECT_STATE.md`, `docs/api-contract.md` và đánh dấu hoàn tất kế hoạch S4.
- [x] Chạy toàn bộ cổng chất lượng (`vendor\bin\pint --test`, `php artisan test`, `npm test`, `npx tsc --noEmit`, `npm run build`). Commit `docs: add test cases, demo script, release runbook, and s4 handoff`.
