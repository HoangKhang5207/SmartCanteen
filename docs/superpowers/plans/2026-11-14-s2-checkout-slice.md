# S2 — Brand Theme, Authentication, Cart & Order Checkout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Cập nhật nhận diện thương hiệu Smart Canteen (Logo + bảng màu xanh lá chủ đạo chuẩn WCAG AA) cho Mini App, đồng thời hoàn thiện luồng xác thực (Zalo Open API + Demo Login local), giỏ hàng trong bộ nhớ, đặt đơn hàng trong DB transaction có `Idempotency-Key`, tính tiền phía server và màn hình "Đơn của tôi" / "Chi tiết đơn".

**Architecture:** Mở rộng Monorepo từ S1: Backend (`api/`) bổ sung `UserRole` & `OrderStatus` enums, `ZaloIdentityService`, `AuthController`, `Order` & `OrderItem` models/migrations, `OrderService` (transaction + idempotency + tính tiền server), `OrderPolicy` và `OrderController`. Frontend (`miniapp/`) áp dụng hệ màu `brand` từ `canteen_logo.jpg`, bổ sung `apiPost` + Bearer token trong bộ nhớ, `cart` state bằng Jotai, và các màn hình Giỏ hàng (`/cart`), Đơn của tôi (`/orders`), Chi tiết đơn (`/orders/:id`).

**Tech Stack:** PHP 8.5.12, Laravel 13, Sanctum, PostgreSQL 17.5, PHPUnit; React 18, TypeScript, ZaUI (`zmp-ui`), Jotai, TanStack Query, Tailwind CSS, Vitest.

**Spec:** `docs/superpowers/specs/2026-11-14-smart-canteen-design.md` (APPROVED 2026-11-14)

## Global Constraints
- Tiền tệ: số nguyên VND (`unsignedInteger`), backend tự tính `unit_price`, `subtotal`, `total_amount` từ DB; bỏ qua mọi giá/tổng tiền do client gửi lên (**AC-04**, **AC-11**).
- Chống đơn trùng bằng header `Idempotency-Key` (1..100 ký tự): cùng user + cùng key + cùng payload → trả lại đơn cũ; cùng key nhưng khác payload → `409 Conflict` (**AC-06**).
- Từ chối đơn có món không tồn tại, thuộc danh mục ngừng hoạt động, hoặc `is_available = false`, hoặc số lượng ngoài khoảng `1..20` → `422 Unprocessable Entity` (**AC-05**).
- Khách hàng chỉ xem được đơn của chính mình; truy cập đơn của người khác trả `404 Not Found` (**AC-10**).
- Không lưu token hay giỏ hàng vào `localStorage`/`cookie` trên Mini App.
- Không đưa `ZALO_APP_SECRET` vào mã nguồn Frontend; chỉ lưu ở `.env` phía Backend.

## Review Focus
1. Client cố tình gửi `total_amount: 1`, `unit_price: 1` trong body `POST /api/v1/orders` → Backend bỏ qua hoàn toàn và lưu đúng tổng tiền tính từ bảng `products`. Test ở Task 4.
2. Khách bấm đặt đơn 2 lần liên tiếp với cùng `Idempotency-Key` và cùng danh sách món → chỉ tạo 1 bản ghi trong `orders`, lần gọi thứ 2 trả về đơn đã tạo. Nếu dùng lại `Idempotency-Key` đó nhưng đổi số lượng/món → trả `409 Conflict`. Test ở Task 4.
3. Giá sản phẩm trong bảng `products` thay đổi sau khi đơn đã tạo → `unit_price`, `subtotal`, `total_amount` của đơn cũ không thay đổi (**AC-11**). Test ở Task 4.
4. User A gọi `GET /api/v1/orders/{id}` của User B → nhận `404 Not Found`, không lộ thông tin đơn (**AC-10**). Test ở Task 4.
5. `POST /api/v1/auth/demo` khi `DEMO_LOGIN_ENABLED=false` hoặc `APP_ENV=production` → trả `403 Forbidden`. Test ở Task 3.

---

### Task 1: Cập nhật Logo & Hệ màu thương hiệu Smart Canteen (Brand UI/UX + S1 Polish)

**Files:**
- Create: `miniapp/src/static/canteen-logo.jpg` (copy từ `canteen_logo.jpg`), `miniapp/src/components/brand-logo.tsx`
- Modify: `DESIGN.md`, `miniapp/tailwind.config.js`, `miniapp/src/css/app.scss`, `miniapp/index.html`, `miniapp/app-config.json`, `miniapp/zmp-cli.json`
- Modify: `miniapp/src/pages/menu.tsx`, `miniapp/src/pages/product-detail.tsx`, `miniapp/src/components/state-views.tsx`
- Delete: `miniapp/src/pages/index.tsx`, `miniapp/src/components/clock.tsx`, `miniapp/src/components/logo.tsx`

- [ ] Copy `canteen_logo.jpg` vào `miniapp/src/static/canteen-logo.jpg`.
- [ ] Cập nhật `DESIGN.md` và `miniapp/tailwind.config.js` với bảng màu `brand` trích xuất từ logo (`brand-50: #EEF8EB`, `brand-100: #D8F0D2`, `brand-500: #5DBE46` leaf green, `brand-600: #3FA32B`, `brand-700: #2E7D32` primary CTA chuẩn WCAG AA 5.14:1, `brand-800: #236B27`, `brand-900: #14532D`) và chuyển `purge` sang `content` trong `tailwind.config.js`.
- [ ] Thiết lập biến màu ZaUI (`--zmp-primary-color: #2E7D32`) trong `miniapp/src/css/app.scss` và cập nhật `theme-color` / tiêu đề trong `index.html`, `app-config.json`, `zmp-cli.json`.
- [ ] Tạo `BrandLogo` component hiển thị logo Smart Canteen và tích hợp vào `MenuPage`, `ProductDetailPage`; đồng thời khắc phục 3 điểm Minor từ S1 (`maxLength={100}` trên ô tìm kiếm, hiển thị trạng thái đang tải khi bấm `"Thử lại"` với `isRefetching`, xóa 3 file mẫu không dùng).
- [ ] Chạy `npm test`, `npx tsc --noEmit`, `npm run build` → PASS. Commit `feat(miniapp): apply smart canteen logo and green brand palette`.

### Task 2: Database Schema & Models cho Users, Orders, OrderItems

**Files:**
- Create: `api/app/Enums/UserRole.php`, `api/app/Enums/OrderStatus.php`
- Create: `api/database/migrations/xxxx_add_role_and_zalo_to_users_table.php`, `xxxx_create_orders_table.php`, `xxxx_create_order_items_table.php`
- Create: `api/app/Models/Order.php`, `api/app/Models/OrderItem.php`
- Create: `api/database/factories/OrderFactory.php`
- Modify: `api/app/Models/User.php`, `api/database/factories/UserFactory.php`
- Test: `api/tests/Feature/Order/OrderSchemaTest.php`

**Interfaces — Produces:**
- `UserRole`: backed string enum (`CUSTOMER = 'customer'`, `STAFF = 'staff'`, `ADMIN = 'admin'`).
- `OrderStatus`: backed string enum (`PENDING = 'PENDING'`, `PREPARING = 'PREPARING'`, `READY = 'READY'`, `COMPLETED = 'COMPLETED'`, `CANCELLED = 'CANCELLED'`) + `allowedTransitions(): array`.
- `users`: `email` nullable, `password` nullable, `role` string(20) default `'customer'`, `zalo_user_id` string(64) nullable unique.
- `orders`: `id`, `user_id` FK→users restrictOnDelete, `code` string(32) unique, `status` string(20) default `'PENDING'`, `total_amount` unsignedInteger, `note` string(500) nullable, `idempotency_key` string(100), `payload_hash` string(64), `version` unsignedInteger default 1, timestamps; unique `['user_id', 'idempotency_key']`.
- `order_items`: `id`, `order_id` FK→orders cascadeOnDelete, `product_id` FK→products restrictOnDelete, `quantity` unsignedSmallInteger, `unit_price` unsignedInteger, `subtotal` unsignedInteger, timestamps.

- [ ] Viết `OrderSchemaTest` kiểm tra: tạo user Zalo không cần email/password, tạo order + order_items với integer VND, unique `(user_id, idempotency_key)`, và `OrderStatus::allowedTransitions()`.
- [ ] Chạy test → FAIL. Hiện thực hóa Enums, Migrations, Models, Factories. Chạy test → PASS. Commit `feat(api): add auth and orders schema and models`.

### Task 3: Xác thực Khách hàng (Zalo Open API + Demo Login Local)

**Files:**
- Create: `api/app/Services/ZaloIdentityService.php`, `api/app/Exceptions/ZaloAuthException.php`
- Create: `api/app/Http/Controllers/Api/V1/AuthController.php`, `api/app/Http/Resources/UserResource.php`
- Modify: `api/config/services.php`, `api/.env.example`, `api/phpunit.xml`, `api/routes/api.php`
- Test: `api/tests/Feature/Auth/AuthApiTest.php`

**Interfaces — Produces:**
- `POST /api/v1/auth/zalo` body `{ access_token: string }` → xác minh qua `https://graph.zalo.me/v2.0/me?fields=id,name,picture` với header `access_token` và `appsecret_proof = hash_hmac('sha256', $token, $secret)` → upsert `User` (`role = customer`) → trả `200 { data: { token: string, user: UserResource } }` hoặc `401`.
- `POST /api/v1/auth/demo` → chỉ hoạt động khi `APP_ENV` thuộc `['local', 'testing']` và `DEMO_LOGIN_ENABLED=true`; trả `200 { data: { token, user } }` hoặc `403`.
- `GET /api/v1/auth/me` (`auth:sanctum`) → trả `200 { data: UserResource }` hoặc `401`.

- [ ] Viết `AuthApiTest` (dùng `Http::fake()` cho `graph.zalo.me`):
  - `test_zalo_login_verifies_token_with_appsecret_proof_and_returns_sanctum_token`
  - `test_zalo_login_returns_401_when_zalo_api_rejects_token`
  - `test_demo_login_succeeds_when_enabled_in_local_or_testing`
  - `test_demo_login_returns_403_when_disabled_or_in_production`
  - `test_me_endpoint_returns_authenticated_user_and_401_for_guest`
- [ ] Chạy test → FAIL. Hiện thực hóa `ZaloIdentityService`, `AuthController`, `UserResource`, routes. Chạy test → PASS. Commit `feat(api): zalo identity verification and local demo auth`.

### Task 4: Order Creation, Idempotency, Server Pricing & Customer Order History API

**Files:**
- Create: `api/app/Services/OrderService.php`, `api/app/Exceptions/IdempotencyConflictException.php`
- Create: `api/app/Http/Requests/StoreOrderRequest.php`
- Create: `api/app/Http/Resources/OrderResource.php`, `api/app/Http/Resources/OrderItemResource.php`
- Create: `api/app/Policies/OrderPolicy.php`
- Create: `api/app/Http/Controllers/Api/V1/OrderController.php`
- Modify: `api/routes/api.php`
- Test: `api/tests/Feature/Order/OrderApiTest.php`

**Interfaces — Produces:**
- `POST /api/v1/orders` (header `Idempotency-Key`, body `{ items: [{ product_id, quantity }], note?: string }`) → `201 Created` (lần đầu) hoặc `200 OK` (gửi lại cùng key + cùng payload).
- `GET /api/v1/orders` → `200 { data: [OrderResource] }` (chỉ đơn của user hiện tại, mới nhất trước).
- `GET /api/v1/orders/{id}` → `200 { data: OrderResource }` hoặc `404` nếu đơn của người khác / không tồn tại.

- [ ] Viết `OrderApiTest` kiểm chứng đầy đủ AC-04, AC-05, AC-06, AC-10, AC-11:
  - `test_creates_order_and_computes_total_on_server_ignoring_client_total` (AC-04)
  - `test_rejects_unavailable_or_inactive_category_product_or_invalid_quantity` (AC-05)
  - `test_idempotency_key_returns_same_order_on_retry_and_409_on_different_payload` (AC-06)
  - `test_missing_idempotency_key_header_returns_422`
  - `test_changing_product_price_later_does_not_change_existing_order` (AC-11)
  - `test_customer_lists_only_own_orders_and_gets_404_for_other_users_order` (AC-10)
  - `test_unauthenticated_order_requests_return_401`
- [ ] Chạy test → FAIL. Hiện thực hóa `OrderService`, `StoreOrderRequest`, `OrderResource`, `OrderPolicy`, `OrderController`. Chạy test → PASS. Commit `feat(api): transactional order creation with idempotency and server pricing`.

### Task 5: Mini App — Auth Client, Cart State & Order API Layer

**Files:**
- Modify: `miniapp/src/lib/api-client.ts`, `miniapp/src/lib/api-client.test.ts`
- Create: `miniapp/src/features/auth/api.ts`, `miniapp/src/features/auth/store.ts`
- Create: `miniapp/src/features/cart/types.ts`, `miniapp/src/features/cart/cart-logic.ts`, `miniapp/src/features/cart/store.ts`, `miniapp/src/features/cart/cart-logic.test.ts`
- Create: `miniapp/src/features/orders/types.ts`, `miniapp/src/features/orders/api.ts`, `miniapp/src/features/orders/status.ts`

**Interfaces — Produces:**
- `setAuthToken(token: string | null)`, `getAuthToken(): string | null`, `apiPost<T>(path, body?, options?: { headers?: Record<string,string> }): Promise<T>`.
- `addCartItem(items, product, qty)`, `updateCartQuantity(items, productId, qty)`, `removeCartItem(items, productId)`, `clearCart()`, `getCartTotals(items) -> { totalQuantity, estimatedAmount }`, `createIdempotencyKey()`.
- `createOrder(input: { items: { product_id: number; quantity: number }[]; note?: string }, idempotencyKey: string): Promise<Order>`, `fetchMyOrders(): Promise<Order[]>`, `fetchOrder(id: number): Promise<Order>`.

- [ ] Viết unit tests trong `cart-logic.test.ts` (AC-03: thêm món, cộng dồn số lượng, giới hạn tối đa 20, bỏ qua món hết hàng, cập nhật số lượng về 0 tự xóa, xóa món, tính tổng) và mở rộng `api-client.test.ts` (gửi `Authorization: Bearer` và `Idempotency-Key` trong `apiPost`).
- [ ] Chạy `npm test` → FAIL. Hiện thực hóa các module. Chạy `npm test` & `npx tsc --noEmit` → PASS. Commit `feat(miniapp): auth, cart state logic, and orders api client`.

### Task 6: Mini App — Giao diện Giỏ hàng, Đặt món, Đơn của tôi & Chi tiết đơn

**Files:**
- Create: `miniapp/src/components/bottom-nav.tsx`, `miniapp/src/pages/cart.tsx`, `miniapp/src/pages/orders.tsx`, `miniapp/src/pages/order-detail.tsx`
- Modify: `miniapp/src/pages/menu.tsx`, `miniapp/src/pages/product-detail.tsx`, `miniapp/src/components/layout.tsx`

- [ ] Tự động khởi tạo phiên đăng nhập khách hàng (`useAuthSession`: ưu tiên Zalo `getAccessToken()` → `/auth/zalo`, fallback sang `/auth/demo` khi chạy trình duyệt local).
- [ ] `MenuPage` & `ProductDetailPage`: thêm nút `"+ Thêm"` / bộ chọn số lượng + `"Thêm vào giỏ"` cho món còn hàng; thanh nổi xem nhanh giỏ hàng.
- [ ] `CartPage` (`/cart`): tăng/giảm/xóa món, nhập ghi chú (`maxLength={500}`), sinh `Idempotency-Key` cố định theo fingerprint giỏ hàng để chống bấm đúp, gọi `POST /api/v1/orders`, xóa giỏ và chuyển tới `/orders/:id`.
- [ ] `OrdersPage` (`/orders`) & `OrderDetailPage` (`/orders/:id`): hiển thị danh sách lịch sử đơn và chi tiết từng món, mã đơn `SC-...`, nhãn trạng thái màu ngữ nghĩa, nút làm mới + polling tự động 7 giây cho đơn chưa kết thúc.
- [ ] Kiểm chứng `npm test`, `npx tsc --noEmit`, `npm run build` → PASS. Commit `feat(miniapp): cart checkout and order history screens`.

### Task 7: Cập nhật tài liệu & Cổng chất lượng cuối cho S2

**Files:**
- Modify: `docs/api-contract.md`, `README.md`, `docs/PROJECT_STATE.md`, `docs/superpowers/plans/2026-11-14-s2-checkout-slice.md`

- [ ] Bổ sung tài liệu API cho `/auth/*` và `/orders*` vào `docs/api-contract.md`.
- [ ] Cập nhật `README.md` và `docs/PROJECT_STATE.md` với tiến độ S2.
- [ ] Chạy toàn bộ cổng chất lượng: `php artisan test`, `vendor\bin\pint --test`, `npm test`, `npx tsc --noEmit`, `npm run build`.
- [ ] Commit `docs: s2 handoff`.
