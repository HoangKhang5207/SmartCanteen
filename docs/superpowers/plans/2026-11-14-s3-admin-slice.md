# S3 — Admin/Staff Portal, RBAC, Order Dispatch, Statistics & Menu CRUD Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Xây dựng hệ thống quản trị Admin/Staff trên Laravel 13 (bao gồm Session Auth trên Blade + Sanctum RBAC middleware `role:...`, Dashboard thống kê hôm nay, Bảng điều phối đơn hàng kèm chuyển trạng thái theo State Machine + Optimistic Locking `version` + ghi `order_status_logs`, và CRUD Danh mục / Món ăn), kết nối trọn vẹn luồng 8 bước với Zalo Mini App.

**Architecture:** Mở rộng Backend Laravel (`api/`): bổ sung bảng `order_status_logs`, middleware `EnsureUserHasRole` (`role:staff,admin` và `role:admin`), mở rộng `OrderService::transitionStatus()` xử lý kiểm tra state machine + optimistic locking (`version`) + ghi log trong `DB::transaction`, xây dựng cả REST API quản trị (`/api/v1/admin/*`) lẫn giao diện Laravel Blade (`/admin/*`) dùng chung `OrderService` và hệ màu thương hiệu Smart Canteen (`#5DBE46` / `#2E7D32`).

**Tech Stack:** PHP 8.5.12, Laravel 13, Blade + Tailwind CSS, Sanctum + Session Auth, PostgreSQL 17.5, PHPUnit.

**Spec:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

## Global Constraints
- Chuyển trạng thái đơn hàng phải tuân thủ nghiêm ngặt State Machine đã phê duyệt (`PENDING → PREPARING | CANCELLED`, `PREPARING → READY | CANCELLED`, `READY → COMPLETED`; `COMPLETED` và `CANCELLED` là trạng thái kết thúc). Mọi chuyển trạng thái trái quy tắc trả `422 Unprocessable Entity` (**AC-08**).
- Kiểm soát đồng thời bằng Optimistic Locking (`version`): khi chuyển trạng thái phải gửi kèm `version` hiện tại; nếu `version` không khớp bản ghi trong DB → trả `409 Conflict` (**Section 1.G**, **Section 2.3**).
- Mỗi lần chuyển trạng thái thành công phải tự động tăng `orders.version` thêm `1` và tạo bản ghi lịch sử trong `order_status_logs` (`order_id`, `from_status`, `to_status`, `changed_by`) trong cùng một `DB::transaction` (**AC-08**).
- Phân quyền RBAC (**AC-10**, **Section 2.2**):
  - `customer`: bị chặn (`403 Forbidden`) khi gọi `/api/v1/admin/*` hoặc truy cập `/admin/*`.
  - `staff` và `admin`: truy cập được Dashboard thống kê, danh sách/chi tiết đơn và chuyển trạng thái đơn.
  - `admin` (hoặc `staff,admin` theo cấu hình route): quản lý Danh mục & Món ăn (CRUD Categories/Products); không cho phép xóa danh mục đang có món ăn hoặc xóa món ăn đã phát sinh trong `order_items` (`restrictOnDelete` → trả lỗi thân thiện `422` / flash error).
- Tiền tệ: số nguyên `VND` (`unsignedInteger`); thay đổi giá sản phẩm trong CRUD không làm thay đổi `unit_price`, `subtotal`, `total_amount` của các đơn hàng đã đặt trước đó (**AC-11**).

## Review Focus
1. Hai nhân viên cùng mở một đơn ở `version = 1`; người thứ nhất chuyển sang `PREPARING` thành công (`version` lên `2`), người thứ hai bấm chuyển với `version = 1` → hệ thống phải từ chối với `409 Conflict` và không ghi thêm `order_status_logs`. Test ở Task 2.
2. Nhân viên cố chuyển đơn từ `PENDING` thẳng sang `READY` hoặc `COMPLETED`, hoặc từ `READY` sang `CANCELLED` → hệ thống từ chối với `422 Unprocessable Entity`. Test ở Task 2.
3. Tài khoản `customer` dùng Bearer token hợp lệ gọi `GET /api/v1/admin/orders`, `PATCH /api/v1/admin/orders/{id}/status`, `GET /api/v1/admin/statistics` hoặc đăng nhập vào `/admin/login` → bị từ chối với `403 Forbidden`. Test ở Task 1 & Task 2.
4. Thống kê hôm nay (`GET /api/v1/admin/statistics` và Dashboard `/admin`): `today_completed_revenue` chỉ cộng `total_amount` của các đơn có `status = COMPLETED` trong ngày hôm nay, không cộng đơn `PENDING`/`PREPARING`/`READY`/`CANCELLED`. Test ở Task 3.
5. Admin sửa giá món ăn hoặc tắt `is_available = false` trên trang quản trị → các đơn hàng lịch sử vẫn giữ nguyên `unit_price`, `subtotal`, `total_amount` cũ (**AC-11**), đồng thời khách đặt mới món vừa tắt `is_available` sẽ bị chặn `422` (**AC-05**). Test ở Task 4.

---

### Task 1: OrderStatusLog Schema, RBAC Middleware `role` & StaffSeeder

**Files:**
- Create: `api/database/migrations/2026_11_14_150001_create_order_status_logs_table.php`
- Create: `api/app/Models/OrderStatusLog.php`
- Create: `api/app/Http/Middleware/EnsureUserHasRole.php`
- Create: `api/database/seeders/StaffSeeder.php`
- Modify: `api/app/Models/Order.php`, `api/bootstrap/app.php`, `api/database/seeders/DatabaseSeeder.php`, `api/app/Http/Resources/OrderResource.php`
- Test: `api/tests/Feature/Admin/RbacAndStatusLogSchemaTest.php`

**Interfaces — Produces:**
- `order_status_logs`: `id`, `order_id` FK→orders cascadeOnDelete, `from_status` string(20), `to_status` string(20), `changed_by` FK→users restrictOnDelete, timestamps.
- `Order::statusLogs(): HasMany` (`OrderStatusLog::class`, ordered by `id ASC`).
- Middleware alias `'role' => EnsureUserHasRole::class`: kiểm tra `$request->user()` có `role->value` nằm trong danh sách tham số (ví dụ `role:staff,admin`); trả `401` nếu chưa đăng nhập, `403` nếu không đủ quyền.
- `StaffSeeder`: tạo idempotent 2 tài khoản nội bộ giả lập cho môi trường demo/local:
  - Admin: `admin@smartcanteen.local` / `Admin@123456` (`role = UserRole::ADMIN`)
  - Staff: `staff@smartcanteen.local` / `Staff@123456` (`role = UserRole::STAFF`)

- [x] Viết `RbacAndStatusLogSchemaTest` kiểm tra: tạo `OrderStatusLog`, quan hệ `order->statusLogs`, `StaffSeeder` idempotent tạo đúng `admin` và `staff`, và middleware `role:staff,admin` chặn `customer` với `403`.
- [x] Chạy test → FAIL. Hiện thực hóa migration, model, middleware, seeder. Chạy test → PASS. Commit `feat(api): add order status logs schema, role middleware, and staff seeder`.

### Task 2: Chuyển trạng thái đơn hàng (State Machine + Optimistic Locking `version`) & Admin Orders API

**Files:**
- Create: `api/app/Exceptions/InvalidOrderStatusTransitionException.php`, `api/app/Exceptions/OrderVersionConflictException.php`
- Create: `api/app/Http/Requests/UpdateOrderStatusRequest.php`
- Create: `api/app/Http/Resources/OrderStatusLogResource.php`
- Create: `api/app/Http/Controllers/Api/V1/Admin/OrderController.php`
- Modify: `api/app/Services/OrderService.php`, `api/app/Http/Resources/OrderResource.php`, `api/routes/api.php`
- Test: `api/tests/Feature/Admin/AdminOrderApiTest.php`

**Interfaces — Produces:**
- `OrderService::transitionStatus(Order $order, OrderStatus $nextStatus, int $expectedVersion, User $actor): Order`
  - Chạy trong `DB::transaction` với `Order::query()->whereKey($order->id)->lockForUpdate()->firstOrFail()`.
  - Nếu `$lockedOrder->version !== $expectedVersion` → ném `OrderVersionConflictException` (`409 Conflict`).
  - Nếu `! $lockedOrder->status->canTransitionTo($nextStatus)` → ném `InvalidOrderStatusTransitionException` (`422 Unprocessable Entity`).
  - Cập nhật `status = $nextStatus`, `version = $lockedOrder->version + 1` và tạo `OrderStatusLog` (`from_status`, `to_status`, `changed_by = $actor->id`).
- `GET /api/v1/admin/orders` (`auth:sanctum`, `role:staff,admin`): hỗ trợ lọc `?status=PENDING` và tìm kiếm `?search=SC-...`, trả danh sách `OrderResource` kèm thông tin khách hàng (`customer_name`) và `status_logs`.
- `PATCH /api/v1/admin/orders/{id}/status` (`auth:sanctum`, `role:staff,admin`): body `{ status: string, version: int }` → trả `200 { data: OrderResource }`, `422` khi sai state machine, `409` khi lệch `version`.

- [x] Viết `AdminOrderApiTest` kiểm chứng AC-07, AC-08, AC-10 và Optimistic Locking:
  - `test_staff_can_list_all_orders_and_filter_by_status_or_code` (AC-07)
  - `test_staff_transitions_order_through_valid_states_increments_version_and_writes_status_logs` (AC-08)
  - `test_invalid_status_transition_returns_422_and_writes_no_log` (AC-08)
  - `test_stale_version_returns_409_conflict_and_prevents_concurrent_overwrite`
  - `test_customer_cannot_access_admin_order_endpoints` (AC-10)
- [x] Chạy test → FAIL. Hiện thực hóa exceptions, `OrderService::transitionStatus`, `UpdateOrderStatusRequest`, `Admin\OrderController`, routes. Chạy test → PASS. Commit `feat(api): add admin order status transition with optimistic locking and audit logs`.

### Task 3: API Thống kê Hôm nay (`GET /api/v1/admin/statistics`) & `StatisticsService`

**Files:**
- Create: `api/app/Services/StatisticsService.php`
- Create: `api/app/Http/Controllers/Api/V1/Admin/StatisticsController.php`
- Modify: `api/routes/api.php`
- Test: `api/tests/Feature/Admin/AdminStatisticsApiTest.php`

**Interfaces — Produces:**
- `StatisticsService::getTodaySummary(): array` trả về:
  - `date`: chuỗi `YYYY-MM-DD` hôm nay
  - `today_total_orders`: tổng số đơn tạo trong ngày hôm nay
  - `today_completed_revenue`: tổng `total_amount` (integer VND) của các đơn tạo hôm nay có `status = COMPLETED`
  - `today_active_orders`: số đơn hôm nay đang ở `PENDING`, `PREPARING`, hoặc `READY`
  - `counts_by_status`: `{ PENDING: int, PREPARING: int, READY: int, COMPLETED: int, CANCELLED: int }` (trong ngày hôm nay)
- `GET /api/v1/admin/statistics` (`auth:sanctum`, `role:staff,admin`) → `200 { data: ... }`.

- [x] Viết `AdminStatisticsApiTest` kiểm tra: tính đúng số đơn theo từng trạng thái trong ngày hôm nay, chỉ cộng doanh thu đơn `COMPLETED` hôm nay, không tính đơn của ngày hôm trước vào chỉ số hôm nay, và chặn `customer` với `403`.
- [x] Chạy test → FAIL. Hiện thực hóa `StatisticsService` và `StatisticsController`. Chạy test → PASS. Commit `feat(api): add today statistics service and admin endpoint`.

### Task 4: Giao diện Quản trị Laravel Blade (Session Auth, Dashboard, Điều phối đơn, CRUD Món ăn & Danh mục)

**Files:**
- Create: `api/app/Http/Controllers/Admin/AuthController.php`
- Create: `api/app/Http/Controllers/Admin/DashboardController.php`
- Create: `api/app/Http/Controllers/Admin/OrderManagementController.php`
- Create: `api/app/Http/Controllers/Admin/CategoryManagementController.php`
- Create: `api/app/Http/Controllers/Admin/ProductManagementController.php`
- Create: `api/resources/views/admin/layout.blade.php`
- Create: `api/resources/views/admin/login.blade.php`
- Create: `api/resources/views/admin/dashboard.blade.php`
- Create: `api/resources/views/admin/orders/index.blade.php`
- Create: `api/resources/views/admin/orders/show.blade.php`
- Create: `api/resources/views/admin/categories/index.blade.php`
- Create: `api/resources/views/admin/products/index.blade.php`
- Create: `api/resources/views/admin/products/form.blade.php`
- Modify: `api/routes/web.php`, `api/public/images/canteen-logo.jpg` (copy từ `canteen_logo.jpg`)
- Test: `api/tests/Feature/Admin/AdminWebPortalTest.php`

**Interfaces — Produces:**
- Web routes (`/admin/...`, middleware `web`):
  - `GET /admin/login`, `POST /admin/login`, `POST /admin/logout` (chỉ cho phép `role:staff,admin` đăng nhập vào khu vực quản trị; nếu tài khoản `customer` cố đăng nhập sẽ bị từ chối `403`).
  - Nhóm `middleware(['auth', 'role:staff,admin'])`:
    - `GET /admin` (Dashboard thống kê hôm nay + danh sách đơn cần xử lý ngay).
    - `GET /admin/orders` (Bảng điều phối đơn hàng: lọc theo trạng thái, tìm theo mã đơn, nút chuyển trạng thái nhanh kèm `version`, tự động làm mới mỗi 10s).
    - `GET /admin/orders/{order}` (Chi tiết đơn hàng, danh sách món, lịch sử chuyển trạng thái `order_status_logs`, và các nút chuyển trạng thái hợp lệ).
    - `PATCH /admin/orders/{order}/status` (Chuyển trạng thái đơn qua form kèm `version`, hiển thị flash success hoặc flash error khi lệch `version` / sai trạng thái).
    - `GET /admin/categories`, `POST /admin/categories`, `PUT /admin/categories/{category}`, `DELETE /admin/categories/{category}` (CRUD Danh mục; chặn xóa danh mục đang chứa món ăn với thông báo lỗi rõ ràng).
    - `GET /admin/products`, `GET /admin/products/create`, `POST /admin/products`, `GET /admin/products/{product}/edit`, `PUT /admin/products/{product}`, `DELETE /admin/products/{product}` (CRUD Món ăn; chặn xóa món ăn đã phát sinh trong `order_items` với thông báo lỗi rõ ràng).

- [x] Viết `AdminWebPortalTest` kiểm tra toàn bộ luồng Web Portal:
  - `test_login_page_renders_and_authenticates_staff_or_admin_while_rejecting_customer`
  - `test_dashboard_and_order_dispatch_board_render_today_stats_and_transition_order_status_with_version_check`
  - `test_category_crud_creates_updates_and_prevents_deleting_category_with_products`
  - `test_product_crud_creates_updates_price_and_availability_without_mutating_past_orders_and_prevents_deleting_ordered_product`
- [x] Chạy test → FAIL. Hiện thực hóa các Controllers, Views Blade (mang nhận diện thương hiệu Smart Canteen: logo `canteen-logo.jpg` + bảng màu xanh lá `#2E7D32` / `#5DBE46`), và `routes/web.php`. Chạy test → PASS. Commit `feat(admin): build blade admin portal for dashboard, order dispatch, and menu crud`.

### Task 5: Cập nhật Tài liệu & Cổng chất lượng cuối cho S3

**Files:**
- Modify: `docs/api-contract.md`, `README.md`, `docs/PROJECT_STATE.md`, `docs/superpowers/plans/2026-11-14-s3-admin-slice.md`

- [x] Bổ sung tài liệu API cho `GET /api/v1/admin/orders`, `PATCH /api/v1/admin/orders/{id}/status`, `GET /api/v1/admin/statistics` vào `docs/api-contract.md`.
- [x] Cập nhật hướng dẫn đăng nhập trang quản trị (`/admin/login`, tài khoản mẫu `admin@smartcanteen.local` / `staff@smartcanteen.local`) trong `README.md` và cập nhật `docs/PROJECT_STATE.md`.
- [x] Chạy `php artisan migrate --seed` trên `titkul_canteen_demo` và chạy toàn bộ cổng chất lượng: `php artisan test`, `vendor\bin\pint --test`, `npm test`, `npx tsc --noEmit`, `npm run build`.
- [x] Commit `docs: s3 handoff`.
