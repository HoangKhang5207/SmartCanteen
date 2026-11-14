# S1 — Monorepo + Menu Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Khách mở Mini App ở local, thấy danh mục + danh sách món + chi tiết món lấy thật từ Laravel API đọc PostgreSQL.

**Architecture:** Monorepo tại workspace root: `api/` (Laravel 13, MVC + FormRequest + API Resource; S1 chưa cần Service/Policy vì chỉ đọc công khai) và `miniapp/` (Zalo Mini App React + TS + ZaUI + TanStack Query). REST JSON dưới `/api/v1`.

**Tech Stack:** PHP 8.5.12 (Herd), Laravel `^13.0`, PHPUnit, PostgreSQL; Node 24, Zalo Mini App CLI, React, TypeScript, `zmp-ui`, `@tanstack/react-query`, Vitest.

**Spec:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

## Global Constraints
- Tiền: số nguyên VND (`unsignedInteger`), không float.
- Dữ liệu seed 100% giả lập; không URL/ảnh/tên từ Căn Tin Số.
- Không commit `.env`, secrets; `.env.example` chỉ chứa giá trị giả.
- Định dạng lỗi: chuẩn Laravel `{ "message": string, "errors"?: {...} }`.
- Không thêm thư viện ngoài danh sách Tech Stack.
- Không dùng localStorage/cookie trong Mini App.
- Không push/merge/deploy.

## Review Focus
1. `search` có dấu tiếng Việt / khác hoa thường ("phở", "PHỞ") → vẫn tìm được (PostgreSQL `LIKE` phân biệt hoa thường → dùng `whereLike(..., caseSensitive: false)`). Test ở Task 3.
2. `category_id` không phải số / không tồn tại → 422, không 500. Test ở Task 3.
3. Sản phẩm thuộc danh mục `is_active=false` → không xuất hiện trong list, detail trả 404. Test ở Task 3.
4. API sập / mạng lỗi → Mini App hiện Error + nút "Thử lại", không màn trắng. Kiểm thủ công Task 6.
5. Origin lạ gọi API → không nhận header CORS cho phép. Test ở Task 4.

---

### Task 0: Khởi tạo repo & môi trường (cần mạng → chạy ngoài sandbox, chỉ các lệnh tải gói)

**Files:** Create `.gitignore`, `README.md` (khung), `api/` (Laravel), `miniapp/` (Zalo template)

- [ ] `git init` tại root; `.gitignore` gồm `node_modules/`, `vendor/`, `.env`, `*.log`, `miniapp/dist/`, `api/storage/*.key`.
- [ ] Commit baseline tài liệu hiện có: `git add . ; git commit -m "chore: baseline project docs"`.
- [ ] `composer create-project laravel/laravel api "^13.0"` → `php artisan --version` hiển thị `Laravel Framework 13.x`.
- [ ] `php artisan install:api` (routes/api.php + Sanctum) → kiểm tra diff tạo ra.
- [ ] `php artisan config:publish cors`.
- [ ] **Điều kiện tiên quyết (người dùng):** PostgreSQL server chạy tại `127.0.0.1:5432`, có DB `titkul_canteen_demo` + `titkul_canteen_test`, user `demo_user`. Cấu hình `api/.env` (không commit). `.env.example`: `DB_CONNECTION=pgsql`, `DB_DATABASE=titkul_canteen_demo`, `DB_USERNAME=demo_user`, `DB_PASSWORD=change_me`.
- [ ] `phpunit.xml`: `DB_CONNECTION=pgsql`, `DB_DATABASE=titkul_canteen_test` (nếu Postgres chưa sẵn sàng: chạy tạm `DB_CONNECTION=sqlite DB_DATABASE=:memory:` bằng env override và ghi nhận là `PARTIAL`).
- [ ] `npx create-zalo-mini-app miniapp` (React + TypeScript); `npm install`; `npm run build` thành công.
- [ ] Commit: `chore: scaffold laravel api and zalo mini app`.

### Task 1: Schema + Models

**Files:**
- Create: `api/database/migrations/xxxx_create_categories_table.php`, `xxxx_create_products_table.php`
- Create: `api/app/Models/Category.php`, `api/app/Models/Product.php`
- Create: `api/database/factories/CategoryFactory.php`, `ProductFactory.php`
- Test: `api/tests/Feature/Menu/SchemaTest.php`

**Interfaces — Produces:**
- `categories`: `id`, `name` string(100), `is_active` bool default true, timestamps.
- `products`: `id`, `category_id` FK→categories restrictOnDelete, `name` string(150), `description` text nullable, `price` unsignedInteger, `image_url` string nullable, `is_available` bool default true, timestamps; index `category_id`.
- `Category::products(): HasMany`, `Category::scopeActive(Builder)`; `Product::category(): BelongsTo`; casts `price: integer`, `is_available/is_active: boolean`.

- [ ] Test `test_product_price_is_stored_as_integer_vnd`: tạo product `price=35000` → `Product::first()->price === 35000` (int).
- [ ] Test `test_category_cannot_be_deleted_while_it_has_products`: xóa category có product → ném `QueryException`.
- [ ] Chạy → FAIL (bảng không tồn tại). Implement. Chạy → PASS. Commit `feat(api): menu schema`.

### Task 2: Seeder dữ liệu giả lập (AC-01)

**Files:** Create `api/database/seeders/MenuSeeder.php`; Modify `DatabaseSeeder.php`; Test `api/tests/Feature/Menu/MenuSeederTest.php`

- [ ] Test `test_menu_seeder_creates_at_least_3_categories_and_10_products`: `seed(MenuSeeder::class)` → `Category::count() >= 3`, `Product::count() >= 10`, mọi `price` > 0 và chia hết cho 1000.
- [ ] Test `test_menu_seeder_is_idempotent`: seed 2 lần → số lượng không đổi (dùng `updateOrCreate` theo `name`).
- [ ] Implement: 4 danh mục (Cơm, Bún & Phở, Đồ uống, Tráng miệng), 12 món, 1 món `is_available=false`, `image_url=null`.
- [ ] PASS → commit `feat(api): fake menu seed data`.

### Task 3: Menu API

**Files:**
- Create: `api/app/Http/Controllers/Api/V1/CategoryController.php` (`index`), `ProductController.php` (`index`, `show`)
- Create: `api/app/Http/Requests/ListProductsRequest.php`
- Create: `api/app/Http/Resources/CategoryResource.php`, `ProductResource.php`
- Modify: `api/routes/api.php` (prefix `v1`)
- Test: `api/tests/Feature/Menu/MenuApiTest.php`

**Interfaces — Produces (contract dùng bởi Task 5):**
- `GET /api/v1/categories` → 200 `{ "data": [{ "id": int, "name": string }] }` — chỉ `is_active`, sắp theo `id`.
- `GET /api/v1/products?category_id=&search=` → 200 `{ "data": [Product] }`, Product = `{ id, category_id, name, description|null, price:int, image_url|null, is_available:bool }`; chỉ sản phẩm thuộc danh mục active; gồm cả món hết (`is_available=false`) để UI hiện "Hết món"; sắp theo `category_id, id`; không phân trang ở S1.
- `ListProductsRequest`: `category_id` `nullable|integer|exists:categories,id`; `search` `nullable|string|max:100`.
- `GET /api/v1/products/{id}` → 200 `{ "data": Product }`; 404 nếu không tồn tại hoặc danh mục inactive.

- [ ] Tests (mỗi test một assertion chính):
  - `test_lists_only_active_categories` — 2 active + 1 inactive → `assertJsonCount(2,'data')`.
  - `test_lists_products_with_integer_price_and_availability` — `assertJsonPath('data.0.price', 35000)`, `is_available` là bool.
  - `test_filters_products_by_category`.
  - `test_search_is_case_insensitive_with_vietnamese` — món "Phở bò", `search=PHỞ` → 1 kết quả.
  - `test_excludes_products_of_inactive_categories`.
  - `test_invalid_category_id_returns_422` — `category_id=abc` và `category_id=999999` → 422 có `errors.category_id`.
  - `test_show_returns_product` / `test_show_returns_404_for_missing_or_inactive_category`.
- [ ] Chạy → FAIL (404 route). Implement (`whereLike('name', "%{$search}%", caseSensitive: false)`; eager query không N+1). PASS → commit `feat(api): public menu endpoints`.

### Task 4: CORS cho Mini App

**Files:** Modify `api/config/cors.php`, `api/.env.example`; Test `api/tests/Feature/CorsTest.php`

- [ ] `allowed_origins` đọc `CORS_ALLOWED_ORIGINS` (phân tách bằng dấu phẩy); `.env.example`: `CORS_ALLOWED_ORIGINS=http://localhost:3000,https://h5.zdn.vn`; `paths: ['api/*']`; `supports_credentials=false`.
- [ ] Test `test_preflight_from_zalo_origin_is_allowed`: `OPTIONS /api/v1/products` với `Origin: https://h5.zdn.vn`, `Access-Control-Request-Method: GET` → header `Access-Control-Allow-Origin: https://h5.zdn.vn`.
- [ ] Test `test_unknown_origin_is_not_allowed`: `Origin: https://evil.example` → không có header đó.
- [ ] PASS → commit `feat(api): cors for mini app origins`.

### Task 5: Mini App — API client & menu data layer

**Files:**
- Create: `miniapp/src/lib/api-client.ts`, `miniapp/src/lib/format.ts`, `miniapp/src/features/menu/api.ts`, `miniapp/src/features/menu/types.ts`
- Create: `miniapp/.env.example` (`VITE_API_BASE_URL=http://localhost:8000/api/v1`)
- Test: `miniapp/src/lib/api-client.test.ts`, `miniapp/src/lib/format.test.ts`
- Modify: `miniapp/package.json` (thêm `vitest`, `@tanstack/react-query`, script `"test": "vitest run"`)

**Interfaces — Produces:**
- `class ApiError extends Error { status: number; errors?: Record<string,string[]> }`
- `apiGet<T>(path: string, params?: Record<string, string|number|undefined>): Promise<T>` — bỏ param `undefined`/rỗng, header `Accept: application/json`, timeout 10s (`AbortController`), non-2xx → `ApiError` với `message` từ body; lỗi mạng/timeout → `ApiError` status `0`.
- `formatVnd(amount: number): string` → `formatVnd(35000)` = `"35.000 ₫"` (`Intl.NumberFormat('vi-VN', {style:'currency', currency:'VND'})`; test so sánh sau khi chuẩn hóa NBSP).
- `fetchCategories(): Promise<Category[]>`, `fetchProducts(q: {categoryId?: number; search?: string}): Promise<Product[]>`, `fetchProduct(id: number): Promise<Product>` (unwrap `data`).

- [ ] Tests: `formatVnd(35000)`; `apiGet` bỏ param rỗng (mock `fetch`, kiểm URL = `.../products?category_id=2`); 404 → `ApiError` `status===404`; fetch reject → `ApiError` `status===0`.
- [ ] `npm test` FAIL → implement → PASS; `npx tsc --noEmit` sạch. Commit `feat(miniapp): api client and menu data layer`.

### Task 6: Mini App — màn Thực đơn & Chi tiết món

**Files:**
- Create: `miniapp/src/pages/menu.tsx`, `miniapp/src/pages/product-detail.tsx`, `miniapp/src/components/state-views.tsx` (`LoadingState`, `ErrorState({message, onRetry})`, `EmptyState({message})`)
- Modify: router/layout của template (route `/` → Menu, `/products/:id` → Detail), bọc `QueryClientProvider` (`retry: 1`, `staleTime: 30_000`)
- Modify: `DESIGN.md` — ghi tokens đang dùng (mặc định ZaUI) là "documented assumption"

- [ ] Menu: tab/chip danh mục ("Tất cả" + categories), ô tìm kiếm (debounce 300ms), danh sách món (tên, giá `formatVnd`, nhãn "Hết món" khi `!is_available`), chạm → detail. Các query categories/products chạy song song (không waterfall). Đủ 4 trạng thái Loading/Error(+Thử lại)/Empty("Không tìm thấy món phù hợp")/Success.
- [ ] Detail: tên, mô tả, giá, trạng thái còn/hết; 404 → "Món không tồn tại" + nút quay lại. (Nút "Thêm vào giỏ" thuộc S2 — chưa hiển thị.)
- [ ] Phần tử bấm được là `button`/link có focus thấy được; ảnh placeholder có `alt`.
- [ ] Kiểm chứng: `npm run build` PASS, `npm test` PASS; chạy `php artisan serve` + `npm start`, mở trình duyệt: thấy 12 món từ API; tắt API → thấy Error + Thử lại; tìm "xyz" → Empty. Chụp ảnh màn hình làm bằng chứng.
- [ ] Commit `feat(miniapp): menu and product detail screens`.

### Task 7: Tài liệu & checkpoint S1

**Files:** Modify `README.md` (yêu cầu, cài đặt, chạy api/miniapp, chạy test), Create `docs/api-contract.md` (phần Menu), Modify `docs/PROJECT_STATE.md` (mục S1)

- [ ] Toàn bộ quality gates: `php artisan test` (pgsql), `vendor/bin/pint --test`, `npm test`, `npx tsc --noEmit`, `npm run build`; ghi lệnh + kết quả thực tế vào PROJECT_STATE.
- [ ] Commit `docs: s1 handoff`.
