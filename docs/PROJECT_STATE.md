# Trạng thái dự án — Smart Canteen (`PROJECT_STATE.md`)

## Tóm tắt chung

- **Dự án:** Smart Canteen (Zalo Mini App × React × Laravel 13 × PostgreSQL)
- **Nhánh hiện tại:** `feature/s2-checkout` (tách từ `main` commit `6f44c9c` sau khi đã merge S1 vào `main`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

---

## Lát cắt S1 — Monorepo + Thực đơn (Menu Vertical Slice)

- **Mã tác vụ:** `S1-MENU-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED` & **Đã merge vào `main` (`6f44c9c`)**.

---

## Lát cắt S2 — Nhận diện thương hiệu, Xác thực, Giỏ hàng & Đặt đơn (Checkout Vertical Slice)

- **Mã tác vụ:** `S2-CHECKOUT-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED`
- **Kế hoạch triển khai:** `docs/superpowers/plans/2026-11-14-s2-checkout-slice.md`
- **Các bước đã hoàn thành:**
  - [x] Task 1: Tích hợp Logo chính thức (`canteen_logo.jpg` → `miniapp/src/static/canteen-logo.jpg`), `BrandLogo` component, hệ màu thương hiệu xanh lá chuẩn WCAG AA (`brand-500: #5DBE46`, `brand-700: #2E7D32`), cập nhật `DESIGN.md`, `tailwind.config.js`, `app.scss` và xử lý 3 góp ý Minor từ S1.
  - [x] Task 2: Database schema & models cho `users` (`role`, `zalo_user_id`), `orders` (`code`, `status`, `total_amount`, `idempotency_key`, `payload_hash`, `version`), `order_items` (`quantity`, `unit_price`, `subtotal`), cùng `UserRole` và `OrderStatus` enums.
  - [x] Task 3: Xác thực khách hàng qua Zalo Open API (`ZaloIdentityService` với `appsecret_proof` HMAC-SHA256) và Demo Login local (`POST /api/v1/auth/demo`, tự động chặn `403` trên production), phát hành Sanctum Bearer token.
  - [x] Task 4: Đặt đơn hàng trong DB transaction (`OrderService` + `lockForUpdate`), kiểm chứng `Idempotency-Key` (`201 Created` / `200 OK` khi retry cùng payload / `409 Conflict` khi khác payload), tính tiền phía server (**AC-04**, **AC-11**), kiểm tra món khả dụng (**AC-05**) và phân quyền xem đơn cá nhân (**AC-10**).
  - [x] Task 5: Lớp dữ liệu Mini App (`apiPost`, Bearer token trong bộ nhớ, logic giỏ hàng Jotai `cart-logic.ts` kèm unit test, API client cho `auth` và `orders`).
  - [x] Task 6: Giao diện Mini App (`BottomNav`, `CartPage`, `OrdersPage`, `OrderDetailPage`, nút thêm vào giỏ trên `MenuPage` & `ProductDetailPage`, tự động làm mới trạng thái đơn).
  - [x] Task 7: Cập nhật tài liệu (`docs/api-contract.md`, `README.md`, `docs/PROJECT_STATE.md`) và chạy toàn bộ cổng chất lượng.

- **Kết quả cổng chất lượng (Quality Gates):**
  - `php artisan test` (PostgreSQL `titkul_canteen_test`): **PASS** — 8 test suites, 31 tests, 157 assertions.
  - `vendor\bin\pint --test`: **PASS** — `{"tool":"pint","result":"passed"}`.
  - `npm test` (Vitest): **PASS** — 3 test files, 11 tests.
  - `npx tsc --noEmit`: **PASS** — 0 lỗi kiểu dữ liệu.
  - `npm run build` (Vite production build): **PASS** — tạo bundle tại `miniapp/www/`.

- **Công việc tiếp theo (S3):**
  - Trang quản trị Admin/Staff trên Laravel (Session Auth, RBAC middleware `role:staff,admin`, Dashboard thống kê hôm nay, Bảng điều phối đơn hàng kèm chuyển trạng thái với Optimistic Locking `version`, và CRUD Món ăn / Danh mục).
