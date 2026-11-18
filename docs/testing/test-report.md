# Báo cáo Kết quả Kiểm thử Tự động — Smart Canteen (`docs/testing/test-report.md`)

- **Ngày thực thi:** `2026-11-14`
- **Môi trường kiểm thử:**
  - Hệ điều hành: Windows 11
  - Backend: PHP `8.5.12` (Laravel `13.x`, PHPUnit `12.5.44`)
  - Cơ sở dữ liệu kiểm thử: PostgreSQL `17.5` (`titkul_canteen_test` trên cổng `54329`)
  - Frontend: Node.js `v24.16.0`, Vitest `2.1.9`, TypeScript `5.x`, Vite `5.4.21`

---

## 1. Tổng hợp Kết quả Thực thi (Executive Test Summary)

| Hạng mục Kiểm thử | Công cụ | Số bộ test (Suites) | Số bài test (Tests) | Số khẳng định (Assertions) | Kết quả |
|---|---|---|---|---|---|
| **Backend Integration & Feature Tests** | PHPUnit + PostgreSQL | `14` | `52` | `497` | **100% PASS (`52/52`)** |
| **Backend Code Style & Formatting** | Laravel Pint | Toàn bộ `api/` | — | — | **PASS (`{"tool":"pint","result":"passed"}`)** |
| **Frontend Unit Tests** | Vitest | `5` | `18` | `41` | **100% PASS (`18/18`)** |
| **Frontend Static Type Checking** | TypeScript (`tsc --noEmit`) | Toàn bộ `miniapp/src` | — | — | **PASS (`0 errors`)** |
| **Frontend Production Bundle Build** | Vite (`npm run build`) | `miniapp/www` | — | — | **PASS (`588 modules`)** |

---

## 2. Chi tiết Các Bộ Kiểm thử Backend (`php artisan test`)

1. **`Tests\Unit\ExampleTest`** (`1 test`) — Kiểm tra khởi tạo môi trường unit test cơ bản.
2. **`Tests\Feature\ExampleTest`** (`1 test`) — Kiểm tra health route gốc.
3. **`Tests\Feature\Menu\SchemaTest`** (`2 tests`) — Kiểm tra lưu trữ số nguyên `VND` và khóa ngoại `restrictOnDelete` giữa `categories` và `products`.
4. **`Tests\Feature\Menu\MenuSeederTest`** (`2 tests`) — Kiểm tra `MenuSeeder` tạo đủ `6` danh mục, `30` món ăn kèm đường dẫn ảnh chuẩn `/images/products/<slug>.jpg` và tính idempotent.
5. **`Tests\Feature\Menu\MenuApiTest`** (`8 tests`) — Kiểm tra `GET /api/v1/categories`, `GET /api/v1/products` (lọc danh mục, tìm kiếm tiếng Việt không phân biệt hoa thường, ẩn danh mục ngừng hoạt động, lỗi `422`), `GET /api/v1/products/{id}` (`200` / `404`).
6. **`Tests\Feature\CorsTest`** (`2 tests`) — Kiểm tra preflight `OPTIONS` từ `https://h5.zdn.vn` được chấp nhận và origin lạ bị từ chối.
7. **`Tests\Feature\Order\OrderSchemaTest`** (`3 tests`) — Kiểm tra schema `users` (khách Zalo không cần email/password), `orders` + `order_items` (unique `idempotency_key` theo user) và `OrderStatus::allowedTransitions()`.
8. **`Tests\Feature\Auth\AuthApiTest`** (`5 tests`) — Kiểm tra `POST /api/v1/auth/zalo` (`appsecret_proof` HMAC-SHA256), `POST /api/v1/auth/demo` (`200` ở local/testing, `403` khi tắt), `GET /api/v1/auth/me`.
9. **`Tests\Feature\Order\OrderApiTest`** (`7 tests`) — Kiểm tra tính tiền phía server (**AC-04**), từ chối món ngừng bán / số lượng sai (**AC-05**), chống đơn trùng `Idempotency-Key` (`201` / `200` / `409`, **AC-06**), cô lập đơn hàng cá nhân (**AC-10**), và bảo toàn giá lịch sử (**AC-11**).
10. **`Tests\Feature\Admin\RbacAndStatusLogSchemaTest`** (`3 tests`) — Kiểm tra `OrderStatusLog`, `StaffSeeder` idempotent, và middleware `role:staff,admin`.
11. **`Tests\Feature\Admin\AdminOrderApiTest`** (`6 tests`) — Kiểm tra danh sách/lọc đơn hàng cho nhân viên (**AC-07**), validate `status` & `limit` (`422`), chuyển trạng thái theo State Machine (**AC-08**), xung đột `version` (`409`), và chặn `customer` (`403`).
12. **`Tests\Feature\Admin\AdminStatisticsApiTest`** (`2 tests`) — Kiểm tra thống kê hôm nay và doanh thu đơn `COMPLETED` trong ngày.
13. **`Tests\Feature\Admin\AdminWebPortalTest`** (`6 tests`) — Kiểm tra đăng nhập/đăng xuất Blade Admin, tự động dọn session `customer` trên `/admin/login`, Dashboard, Điều phối đơn, CRUD Danh mục & Món ăn (chỉ `role:admin` được thêm/sửa/xóa).
14. **`Tests\Feature\EndToEndDemoWorkflowTest`** (`1 test`, `46 assertions`) — Kiểm tra trọn vẹn luồng trình diễn 8 bước từ lúc khách mở Mini App đến khi hoàn tất đơn hàng trên Admin.

---

## 3. Chi tiết Các Bộ Kiểm thử Frontend (`npm test`)

1. **`src/lib/format.test.ts`** (`1 test`) — Định dạng tiền tệ `formatVnd` sang chuẩn tiếng Việt (`45.000 ₫`).
2. **`src/lib/api-client.test.ts`** (`5 tests`) — Kiểm tra `apiGet`, `apiPost`, truyền header `Authorization: Bearer <token>`, truyền `Idempotency-Key`, và bóc tách lỗi `ApiError`.
3. **`src/features/cart/cart-logic.test.ts`** (`5 tests`) — Kiểm tra thêm món, cộng dồn số lượng, giới hạn tối đa `20`, chặn món `is_available = false`, giảm về `0` tự xóa, và tính `computeCartFingerprint` phục vụ `Idempotency-Key`.
4. **`src/components/product-image.test.ts`** (`4 tests`) — Kiểm tra chuẩn hóa đường dẫn hình ảnh sản phẩm (`resolveProductImageUrl`) cho cả đường dẫn nội bộ `/images/products/<slug>.jpg` và URL `https://...`.
5. **`src/components/pagination-bar.test.ts`** (`3 tests`) — Kiểm tra `normalizePaginatedResult` và thuật toán tính cửa sổ trang `buildPageItems` (`1 ... current-1 current current+1 ... last`).
