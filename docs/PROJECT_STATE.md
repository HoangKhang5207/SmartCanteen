# Trạng thái dự án — Smart Canteen (`PROJECT_STATE.md`)

## Tóm tắt chung

- **Dự án:** Smart Canteen (Zalo Mini App × React × Laravel 13 × PostgreSQL)
- **Nhánh hiện tại:** `feature/s1-menu` (tách từ `main` commit `e53e23a`)
- **Đặc tả thiết kế:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

---

## Lát cắt S1 — Monorepo + Thực đơn (Menu Vertical Slice)

- **Mã tác vụ:** `S1-MENU-SLICE`
- **Trạng thái:** `COMPLETED_VERIFIED` (trong phạm vi S1; kiểm chứng trực quan bằng subagent Chrome DevTools bị chặn bởi cấu hình trình duyệt máy chủ, đã kiểm chứng qua HTTP/build/tests).
- **Các bước đã hoàn thành:**
  - [x] Task 0: Khởi tạo Git monorepo, Laravel 13 (`api/`), Zalo Mini App React + TS + ZaUI (`miniapp/`), PostgreSQL 17.5 (`titkul_canteen_demo`, `titkul_canteen_test`).
  - [x] Task 1: Migrations, Models, Factories cho `categories` và `products` (giá lưu `unsignedInteger` VND, khóa ngoại `restrictOnDelete`).
  - [x] Task 2: `MenuSeeder` tạo dữ liệu giả lập (4 danh mục, 12 sản phẩm, idempotent).
  - [x] Task 3: Public Menu REST API (`GET /api/v1/categories`, `GET /api/v1/products`, `GET /api/v1/products/{id}`).
  - [x] Task 4: Cấu hình CORS theo biến môi trường `CORS_ALLOWED_ORIGINS` (`http://localhost:3000,https://h5.zdn.vn`).
  - [x] Task 5: Mini App API client (`apiGet`, `ApiError`), định dạng tiền tệ (`formatVnd`), lớp dữ liệu thực đơn (`fetchCategories`, `fetchProducts`, `fetchProduct`).
  - [x] Task 6: Màn hình Thực đơn (`MenuPage`) và Chi tiết món (`ProductDetailPage`) với đầy đủ 4 trạng thái (`Loading`, `Error` + nút thử lại, `Empty`, `Success`), tìm kiếm debounce 300ms, lọc theo danh mục, truy cập bàn phím.
  - [x] Task 7: Tài liệu hóa (`README.md`, `docs/api-contract.md`, `DESIGN.md`, `docs/PROJECT_STATE.md`).

- **Kết quả cổng chất lượng (Quality Gates):**
  - `php artisan test` (PostgreSQL `titkul_canteen_test`): **PASS** — 6 test suites, 16 tests, 73 assertions.
  - `vendor\bin\pint --test`: **PASS** — `{"tool":"pint","result":"passed"}`.
  - `npm test` (Vitest): **PASS** — 2 test files, 5 tests.
  - `npx tsc --noEmit`: **PASS** — 0 lỗi kiểu dữ liệu.
  - `npm run build` (Vite production build): **PASS** — tạo bundle tại `miniapp/www/`.

- **Công việc tiếp theo (S2):**
  - Xác thực (Zalo Open API + demo login ở local), giỏ hàng trong bộ nhớ, tạo đơn hàng trong DB transaction kèm `Idempotency-Key`, tính tiền phía server và màn hình "Đơn của tôi".
