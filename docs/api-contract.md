# Hợp đồng REST API — Smart Canteen (`/api/v1`)

- **Base URL (Local):** `http://localhost:8000/api/v1`
- **Định dạng:** JSON (`Accept: application/json`, `Content-Type: application/json`)
- **Tiền tệ:** Số nguyên dương theo đơn vị `VND` (`integer`, không dùng `float`).
- **Xác thực:** Laravel Sanctum Personal Access Token qua header `Authorization: Bearer <token>`.
- **CORS:** Cho phép các origin cấu hình trong `CORS_ALLOWED_ORIGINS` (mặc định: `http://localhost:3000,https://h5.zdn.vn`).
- **Định dạng lỗi chuẩn:**
  ```json
  {
    "message": "Mô tả lỗi",
    "errors": {
      "ten_truong": ["Chi tiết lỗi validation"]
    }
  }
  ```

---

## 1. Thực đơn công khai (S1 — Public Menu Endpoints)

### 1.1. Danh sách danh mục đang hoạt động

- **Endpoint:** `GET /api/v1/categories`
- **Xác thực:** Công khai
- **Response `200 OK`:**
  ```json
  {
    "data": [
      { "id": 1, "name": "Cơm" },
      { "id": 2, "name": "Bún & Phở" }
    ]
  }
  ```
- **Quy tắc:** Chỉ trả về danh mục có `is_active = true`, sắp xếp tăng dần theo `id`.

### 1.2. Danh sách món ăn

- **Endpoint:** `GET /api/v1/products`
- **Xác thực:** Công khai
- **Query Parameters:**
  | Tham số | Kiểu | Bắt buộc | Quy tắc |
  |---|---|---|---|
  | `category_id` | `integer` | Không | Phải tồn tại trong bảng `categories` (`exists:categories,id`) |
  | `search` | `string` | Không | Tối đa 100 ký tự; tìm kiếm không phân biệt hoa thường trên `name` (hỗ trợ tiếng Việt) |
  | `page` | `integer` | Không | Số trang hiện tại (`min:1`, mặc định `1`) |
  | `per_page` | `integer` | Không | Số món mỗi trang (`min:1, max:100`, mặc định `12`; Mini App sử dụng `8`) |
- **Response `200 OK`:** Trả về mảng `data` kèm đối tượng `meta` (`current_page`, `last_page`, `per_page`, `total`, `from`, `to`) và `links` (`first`, `last`, `prev`, `next`) chuẩn Laravel Resource Pagination:
  ```json
  {
    "data": [
      {
        "id": 1,
        "category_id": 1,
        "name": "Cơm sườn nướng mật ong",
        "description": "Sườn heo nướng mật ong, dưa góp, canh theo ngày.",
        "price": 45000,
        "image_url": null,
        "is_available": true
      }
    ]
  }
  ```
- **Response `422 Unprocessable Entity`:** Khi `category_id` không phải số nguyên hoặc không tồn tại.
- **Quy tắc:**
  - Chỉ trả về sản phẩm thuộc danh mục có `is_active = true`.
  - Bao gồm cả sản phẩm có `is_available = false` để giao diện hiển thị trạng thái `"Hết món"`.
  - Sắp xếp theo `category_id ASC, id ASC`.

### 1.3. Chi tiết món ăn

- **Endpoint:** `GET /api/v1/products/{id}`
- **Xác thực:** Công khai
- **Response `200 OK`:**
  ```json
  {
    "data": {
      "id": 1,
      "category_id": 1,
      "name": "Cơm sườn nướng mật ong",
      "description": "Sườn heo nướng mật ong, dưa góp, canh theo ngày.",
      "price": 45000,
      "image_url": null,
      "is_available": true
    }
  }
  ```
- **Response `404 Not Found`:** Khi sản phẩm không tồn tại hoặc thuộc danh mục `is_active = false`.

---

## 2. Xác thực khách hàng (S2 — Customer Authentication Endpoints)

### 2.1. Đăng nhập bằng Zalo Access Token

- **Endpoint:** `POST /api/v1/auth/zalo`
- **Xác thực:** Công khai
- **Request Body:**
  ```json
  {
    "access_token": "zalo_oauth_access_token_string"
  }
  ```
- **Cơ chế xác minh phía Server:**
  - Backend gọi `GET https://graph.zalo.me/v2.0/me?fields=id,name,picture` kèm header `access_token` và `appsecret_proof = hash_hmac('sha256', $accessToken, config('services.zalo.app_secret'))`.
  - Không tin tưởng thông tin định danh do client tự gửi mà không qua bước xác minh token với Zalo Open API.
- **Response `200 OK`:**
  ```json
  {
    "data": {
      "token": "1|sanctum_personal_access_token",
      "user": {
        "id": 1,
        "name": "Nguyễn Văn A",
        "role": "customer",
        "zalo_user_id": "849123456789"
      }
    }
  }
  ```
- **Response `401 Unauthorized`:** Khi `access_token` không hợp lệ hoặc bị Zalo Open API từ chối.

### 2.2. Đăng nhập Demo (Chỉ dành cho môi trường `local` / `testing`)

- **Endpoint:** `POST /api/v1/auth/demo`
- **Xác thực:** Công khai (chỉ hoạt động khi `APP_ENV` thuộc `['local', 'testing']` **và** `DEMO_LOGIN_ENABLED=true`)
- **Request Body (tùy chọn):**
  ```json
  {
    "demo_key": "customer1"
  }
  ```
- **Response `200 OK`:** Cấu trúc giống `POST /api/v1/auth/zalo`.
- **Response `403 Forbidden`:** Khi `DEMO_LOGIN_ENABLED=false` hoặc `APP_ENV` không phải `local`/`testing`.

### 2.3. Thông tin phiên đăng nhập hiện tại

- **Endpoint:** `GET /api/v1/auth/me`
- **Xác thực:** `Bearer <token>` (`auth:sanctum`)
- **Response `200 OK`:**
  ```json
  {
    "data": {
      "id": 1,
      "name": "Khách hàng Demo (customer1)",
      "role": "customer",
      "zalo_user_id": "demo_customer1"
    }
  }
  ```
- **Response `401 Unauthorized`:** Khi thiếu hoặc sai Bearer token.

---

## 3. Đặt món & Lịch sử đơn hàng (S2 — Customer Orders Endpoints)

### 3.1. Tạo đơn hàng mới (Transactional + Idempotent + Server Pricing)

- **Endpoint:** `POST /api/v1/orders`
- **Xác thực:** `Bearer <token>` (`auth:sanctum`)
- **Headers bắt buộc:**
  | Header | Kiểu | Bắt buộc | Quy tắc |
  |---|---|---|---|
  | `Idempotency-Key` | `string` | Có | Độ dài `1..100` ký tự (khuyến nghị UUID v4 sinh cho mỗi thao tác đặt đơn) |
- **Request Body:**
  ```json
  {
    "items": [
      { "product_id": 1, "quantity": 2 },
      { "product_id": 3, "quantity": 1 }
    ],
    "note": "Ít cơm, không hành"
  }
  ```
- **Quy tắc nghiệp vụ & bảo toàn dữ liệu:**
  1. `items`: mảng từ `1..50` phần tử; mỗi phần tử có `product_id` nguyên dương không trùng lặp (`distinct`) và `quantity` trong khoảng `1..20`.
  2. `note`: tùy chọn, tối đa `500` ký tự.
  3. **Tính tiền phía Server (AC-04, AC-11):** Server khóa bản ghi (`lockForUpdate`) các sản phẩm liên quan trong bảng `products`, tự tính `unit_price = product.price`, `subtotal = unit_price * quantity`, `total_amount = sum(subtotal)` và lưu cố định vào `order_items` / `orders`. Mọi trường giá hoặc tổng tiền do client gửi lên đều bị bỏ qua.
  4. **Kiểm tra khả dụng (AC-05):** Nếu bất kỳ `product_id` nào không tồn tại, có `is_available = false`, hoặc thuộc danh mục có `is_active = false`, giao dịch bị từ chối với `422 Unprocessable Entity`.
  5. **Chống đơn trùng (AC-06):**
     - Cùng `(user_id, Idempotency-Key)` và cùng `payload_hash` (SHA-256 của items chuẩn hóa + note): trả về `200 OK` kèm đơn hàng đã tạo trước đó (không tạo bản ghi mới).
     - Cùng `(user_id, Idempotency-Key)` nhưng khác `payload_hash`: trả về `409 Conflict`.
- **Response `201 Created` (hoặc `200 OK` khi gửi lại cùng `Idempotency-Key`):**
  ```json
  {
    "data": {
      "id": 15,
      "code": "SC-261114-000015",
      "status": "PENDING",
      "total_amount": 115000,
      "note": "Ít cơm, không hành",
      "version": 1,
      "created_at": "2026-11-14T13:05:00+00:00",
      "updated_at": "2026-11-14T13:05:00+00:00",
      "items": [
        {
          "id": 29,
          "product_id": 1,
          "product_name": "Cơm sườn nướng mật ong",
          "quantity": 2,
          "unit_price": 45000,
          "subtotal": 90000
        },
        {
          "id": 30,
          "product_id": 3,
          "product_name": "Trà tắc mật ong",
          "quantity": 1,
          "unit_price": 25000,
          "subtotal": 25000
        }
      ]
    }
  }
  ```
- **Response `409 Conflict`:** Khi tái sử dụng `Idempotency-Key` với payload khác.
- **Response `422 Unprocessable Entity`:** Khi thiếu `Idempotency-Key`, số lượng ngoài khoảng `1..20`, hoặc có món không khả dụng.

### 3.2. Danh sách đơn hàng của tôi (Có phân trang)

- **Endpoint:** `GET /api/v1/orders`
- **Xác thực:** `Bearer <token>` (`auth:sanctum`)
- **Query Parameters:**
  | Tham số | Kiểu | Bắt buộc | Quy tắc |
  |---|---|---|---|
  | `page` | `integer` | Không | Số trang hiện tại (`min:1`, mặc định `1`) |
  | `per_page` | `integer` | Không | Số đơn hàng mỗi trang (`min:1, max:50`, mặc định `10`; Mini App sử dụng `6`) |
- **Response `200 OK`:** Danh sách `OrderResource` của riêng người dùng đang đăng nhập kèm `meta` và `links` phân trang, sắp xếp mới nhất trước (`created_at DESC, id DESC`).

### 3.3. Chi tiết đơn hàng của tôi

- **Endpoint:** `GET /api/v1/orders/{id}`
- **Xác thực:** `Bearer <token>` (`auth:sanctum`)
- **Response `200 OK`:** `{ "data": OrderResource }`
- **Response `404 Not Found`:** Khi đơn hàng không tồn tại hoặc thuộc về người dùng khác (**AC-10** — không tiết lộ sự tồn tại của đơn hàng người khác).

---

## 4. Quản trị & Điều phối đơn hàng (S3 — Staff/Admin Endpoints)

> **Phân quyền (RBAC):** Toàn bộ nhóm `/api/v1/admin/*` yêu cầu xác thực `auth:sanctum` **và** middleware `role:staff,admin`. Tài khoản có `role = customer` gọi vào sẽ nhận `403 Forbidden`.

### 4.1. Danh sách đơn hàng cho Nhân viên / Quản trị viên

- **Endpoint:** `GET /api/v1/admin/orders`
- **Xác thực & Phân quyền:** `Bearer <token>` (`role:staff,admin`)
- **Query Parameters:**
  | Tham số | Kiểu | Bắt buộc | Quy tắc |
  |---|---|---|---|
  | `status` | `string` | Không | Không phân biệt hoa thường (`PENDING`, `PREPARING`, `READY`, `COMPLETED`, `CANCELLED`); truyền giá trị ngoài danh sách trả `422 Unprocessable Entity` |
  | `search` | `string` | Không | Tìm kiếm không phân biệt hoa thường theo mã đơn (`code`, tối đa 64 ký tự) |
  | `page` | `integer` | Không | Số trang hiện tại (`min:1`, mặc định `1`) |
  | `per_page` | `integer` | Không | Số lượng bản ghi mỗi trang (`1..100`, mặc định `20`); ngoài khoảng trả `422` |
  | `limit` | `integer` | Không | Bí danh tương thích ngược của `per_page` (`1..100`); ngoài khoảng trả `422` |
- **Response `200 OK`:** Danh sách `OrderResource` (bao gồm `customer_name`, `items` và `status_logs`) kèm `meta` và `links` phân trang.

### 4.2. Chuyển trạng thái đơn hàng (State Machine + Optimistic Locking)

- **Endpoint:** `PATCH /api/v1/admin/orders/{id}/status`
- **Xác thực & Phân quyền:** `Bearer <token>` (`role:staff,admin`)
- **Request Body:**
  ```json
  {
    "status": "PREPARING",
    "version": 1
  }
  ```
- **Quy tắc chuyển trạng thái (State Machine):**
  - `PENDING` → `PREPARING` hoặc `CANCELLED`
  - `PREPARING` → `READY` hoặc `CANCELLED`
  - `READY` → `COMPLETED`
  - `COMPLETED`, `CANCELLED` → Trạng thái kết thúc (không chuyển tiếp)
- **Kiểm soát đồng thời & Nhật ký (Optimistic Locking + Audit Log):**
  - Thực thi trong `DB::transaction` với khóa dòng `lockForUpdate()`.
  - Nếu `version` gửi lên khác `orders.version` hiện tại → trả `409 Conflict` (không ghi đè và không ghi log).
  - Nếu trạng thái đích không hợp lệ theo State Machine → trả `422 Unprocessable Entity`.
  - Khi hợp lệ: cập nhật `status`, tăng `version` thêm `1`, và ghi bản ghi vào bảng `order_status_logs` (`order_id`, `from_status`, `to_status`, `changed_by`).
- **Response `200 OK`:** `{ "data": OrderResource }` (kèm `version` mới và mảng `status_logs`).

### 4.3. Thống kê vận hành trong ngày hôm nay

- **Endpoint:** `GET /api/v1/admin/statistics`
- **Xác thực & Phân quyền:** `Bearer <token>` (`role:staff,admin`)
- **Response `200 OK`:**
  ```json
  {
    "data": {
      "date": "2026-11-14",
      "today_total_orders": 5,
      "today_completed_revenue": 150000,
      "today_active_orders": 2,
      "counts_by_status": {
        "PENDING": 1,
        "PREPARING": 1,
        "READY": 0,
        "COMPLETED": 2,
        "CANCELLED": 1
      }
    }
  }
  ```
- **Quy tắc:** `today_completed_revenue` chỉ cộng `total_amount` (số nguyên `VND`) của các đơn tạo trong ngày hôm nay có `status = COMPLETED`.

---

## 5. Phân hệ Căn tin Số — Thẻ Học sinh & Liên kết Ngân hàng + OTP (Trục 2 · Slice D1)

### 5.1. Danh sách ngân hàng hỗ trợ trích nợ tự động
- **Endpoint:** `GET /api/v1/canteen/banks` (Công khai)
- **Response `200 OK`:** Danh sách các ngân hàng hỗ trợ (`MB`, `VCB`, `BIDV`, `TCB`, `CTG`, `ACB`, `VPB`) kèm `account_min_length`, `account_max_length`.

### 5.2. Danh sách & chi tiết Thẻ học sinh thông minh (`CARD-01..07`)
- **Endpoints:** `GET /api/v1/canteen/cards` và `GET /api/v1/canteen/cards/{cardId}` (`auth:sanctum`)
- **Quy tắc bảo mật:** Chỉ trả thẻ gắn với tài khoản người dùng đang đăng nhập; truy cập `cardId` của người khác trả `404 Not Found`. Thông tin ngân hàng trong `bank_link` luôn ở dạng đã che (`account_number_masked`, `national_id_masked`, `phone_masked`).

### 5.3. Khởi tạo liên kết tài khoản ngân hàng & cấp OTP 6 chữ số (`BANK-02..11`, `OTP-01`, `OTP-07`)
- **Endpoint:** `POST /api/v1/canteen/cards/{cardId}/bank-links` (`auth:sanctum`)
- **Validation:** `bank_code` hợp lệ, `account_number` chỉ gồm chữ số đúng độ dài ngân hàng, `account_holder_name` (`2..150`), `national_id` (`digits:12`), `phone_number` (`^0[35789]\d{8}$`), `email` (`nullable|email`), `agreed_terms` (`accepted`).
- **Response:** `201 Created` trả `bank_link_id`, `phone_masked`, và metadata OTP (`expires_in_seconds: 180`, `resend_cooldown_seconds: 30`, `max_attempts: 5`, `max_resends: 3`). `debug_otp` chỉ trả khi `local`/`testing` + `BANKING_EXPOSE_DEBUG_OTP=true`.
- **Lỗi:** `404` (thẻ người khác), `409 Conflict` (`CARD_ALREADY_LINKED`), `422` (validation hoặc thẻ bị khóa `LOCKED`).

### 5.4. Xác thực OTP 6 chữ số (`OTP-02..05`, `BANK-12..15`)
- **Endpoint:** `POST /api/v1/canteen/bank-links/{bankLinkId}/verify-otp` (`auth:sanctum`)
- **Request Body:** `{ "otp_code": "123456" }`
- **Response `200 OK`:** `{ "data": StudentCardResource }` với `is_bank_linked: true` và `bank_link.status = "ACTIVE"`.
- **Lỗi chuẩn:** `404`, `409` (`BANK_LINK_ALREADY_ACTIVE` / `BANK_LINK_INVALID_STATE`), `422` (`OTP_INVALID` kèm `meta.remaining_attempts`, `OTP_EXPIRED`), `429` (`OTP_MAX_ATTEMPTS_EXCEEDED`).

### 5.5. Gửi lại mã OTP (`OTP-01..07`)
- **Endpoint:** `POST /api/v1/canteen/bank-links/{bankLinkId}/resend-otp` (`auth:sanctum`)
- **Response `200 OK`:** Cấp mã OTP mới (vô hiệu hóa mã cũ), cập nhật `remaining_resends`.
- **Lỗi chuẩn:** `404`, `409`, `429` (`OTP_RESEND_COOLDOWN` kèm `meta.retry_after_seconds`, hoặc `OTP_MAX_RESENDS_EXCEEDED`, hoặc `OTP_MAX_ATTEMPTS_EXCEEDED`).

---

## 6. Hạn mức Chi tiêu Ngày, Lịch sử Hạn mức & Thanh toán Trừ Hạn mức / POS (Trục 2 · Slice D2)

### 6.1. Thiết lập / thay đổi hạn mức chi tiêu hằng ngày (`LIMIT-01..12`)
- **Endpoint:** `PUT /api/v1/canteen/cards/{cardId}/spending-limit` (`auth:sanctum`)
- **Request Body:** `{ "daily_limit": 100000, "reason": "Tùy chọn" }` (`0..5_000_000`, bội số `1000`).
- **Response `200 OK`:** `{ "data": StudentCardResource }` (kèm `spending_limit: { daily_limit, spent_today, remaining_limit, effective_date }`).
- **Lỗi:** `404` (thẻ người khác), `422` (`CARD_NOT_BANK_LINKED` hoặc validation).

### 6.2. Lịch sử thay đổi hạn mức (`LIMIT-13`) & Đơn hàng gắn thẻ (`ORDER-01..07`)
- **Endpoints:** `GET /api/v1/canteen/cards/{cardId}/limit-logs` và `GET /api/v1/canteen/cards/{cardId}/orders` (`auth:sanctum`, phân trang `page`/`per_page`).

### 6.3. Mô phỏng Quẹt thẻ POS (`ORDER-01..07`, `DETAIL-01..14`)
- **Endpoint:** `POST /api/v1/canteen/cards/{cardId}/pos-purchases` (`auth:sanctum`, header bắt buộc `Idempotency-Key`)
- **Request Body:** `{ "items": [{ "product_id": 1, "quantity": 1 }], "store_name": "THCS Trần Phú" }`
- **Response `201 Created` / `200 OK` (replay):** `{ "data": OrderResource }` với mã `POS-YYMMDD-XXXXXX`, `status = COMPLETED`, `payment_method = STUDENT_CARD_POS`, `payment_status = PAID`, `remaining_limit_snapshot`.
- **Mở rộng `POST /api/v1/orders` & `OrderResource`:** Nhận tùy chọn `student_card_id` & `payment_method` (`COUNTER_CASH` | `STUDENT_CARD`), tự động hoàn hạn mức và chuyển `payment_status = REFUNDED` khi Staff/Admin hủy đơn thanh toán bằng thẻ trong ngày.

---

## 7. Hủy Liên kết Ngân hàng qua OTP & Quản trị Blade Căn tin Số (Trục 2 · Slice D3)

### 7.1. Khởi tạo yêu cầu hủy liên kết ngân hàng (`UNLINK-01..04`, `OTP-08`)
- **Endpoint:** `POST /api/v1/canteen/bank-links/{bankLinkId}/unlink/initiate` (`auth:sanctum`, `throttle:10,1`)
- **Quy tắc:** Chỉ chủ thẻ có `bank_links.status = ACTIVE` mới được khởi tạo (cho phép hủy liên kết kể cả khi thẻ đang `LOCKED`). Vô hiệu hóa các OTP `BANK_UNLINK` cũ chưa xác thực và phát hành mã OTP 6 chữ số mới (`purpose = BANK_UNLINK`).
- **Response `201 Created`:** Trả `bank_link_id`, `student_card_id`, `purpose: "BANK_UNLINK"`, `phone_masked`, `account_number_masked`, và metadata OTP (`debug_otp` chỉ trả khi `local`/`testing` + `BANKING_EXPOSE_DEBUG_OTP=true`).
- **Lỗi chuẩn:** `404`, `409 Conflict` (`BANK_LINK_NOT_ACTIVE`), `429`.

### 7.2. Xác thực OTP hủy liên kết (`UNLINK-05..07`, `OTP-08`)
- **Endpoint:** `POST /api/v1/canteen/bank-links/{bankLinkId}/unlink/verify-otp` (`auth:sanctum`, `throttle:10,1`)
- **Request Body:** `{ "otp_code": "123456" }`
- **Response `200 OK`:** `{ "data": StudentCardResource }` với `is_bank_linked: false`, `bank_link: null`, vẫn bảo toàn `spending_limit` (kể cả `spent_today` trong ngày).
- **An toàn khóa phiên:** Nhập sai quá số lần cho phép trả `429` (`OTP_MAX_ATTEMPTS_EXCEEDED`) và vô hiệu hóa phiên OTP hủy đó, **không** phá hỏng `bank_links.status = ACTIVE`.

### 7.3. Gửi lại mã OTP hủy liên kết
- **Endpoint:** `POST /api/v1/canteen/bank-links/{bankLinkId}/unlink/resend-otp` (`auth:sanctum`, `throttle:10,1`)
- **Response `200 OK`:** Cấp mã OTP `BANK_UNLINK` mới (áp dụng cooldown `30s`, tối đa `3` lần, chặn nếu phiên đã khóa do nhập sai).

### 7.4. Tuyến Quản trị Blade Căn tin Số (`/admin/canteen/...`, session `auth`)
- `GET /admin/canteen/cards` & `GET /admin/canteen/cards/{studentCard}` (`role:staff,admin`): Xem, tìm kiếm, lọc và kiểm tra chi tiết thẻ học sinh, lịch sử liên kết (chỉ hiển thị dữ liệu che mờ), hạn mức hôm nay và đơn hàng gắn thẻ.
- `PATCH /admin/canteen/cards/{studentCard}/status` (`role:admin`): Chuyển trạng thái `ACTIVE ↔ LOCKED` hoặc thu hồi vĩnh viễn `REVOKED` (tự động ngắt `UNLINKED` ngân hàng trong cùng transaction).
- `PUT /admin/canteen/cards/{studentCard}/spending-limit` (`role:admin`): Điều chỉnh hạn mức hằng ngày kèm audit log `changed_by = admin.id`.

---

## 8. Xác thực bằng SĐT Zalo (Lazy Auth) & Cổng SMS OTP (Trục 3 · Slice P1)

### 8.1. Đăng nhập bằng Số điện thoại Zalo (`POST /api/v1/auth/zalo-phone`)
- **Middleware:** `throttle:20,1` (Công khai, không yêu cầu token trước)
- **Request Body:** `{ "access_token": "...", "phone_token": "...", "name": "tùy chọn" }` (server chỉ tin cậy `zalo_user_id` được xác minh từ `access_token`, không nhận `zalo_user_id` từ client)
- **Cơ chế:** Server đổi `phone_token` (từ `zmp-sdk getPhoneNumber()`) qua `https://graph.zalo.me/v2.0/me/info` bằng `ZALO_APP_SECRET`, chuẩn hóa SĐT VN (`0xxxxxxxxx`), đối khớp/tạo `User` trong transaction (giữ nguyên role `staff`/`admin` nếu đã có; trả `409 ACCOUNT_CONFLICT` nếu `zalo_user_id` gắn SĐT khác).
- **Response `200 OK`:** `{ "data": { "token": "<sanctum_token>", "user": { "id", "name", "role", "zalo_user_id", "phone", "phone_masked", "has_verified_phone" } } }`.
- **Lỗi chuẩn:** `401` (`ZALO_PHONE_TOKEN_INVALID`), `409` (`ACCOUNT_CONFLICT`), `422` (`INVALID_PHONE_NUMBER`), `503` (`ZALO_CONFIG_MISSING` / `ZALO_UNAVAILABLE`).

### 8.2. Đăng nhập SĐT Demo cho Local/Dev/Testing (`POST /api/v1/auth/demo-phone`)
- **Middleware:** `throttle:20,1` — chỉ hoạt động khi `APP_ENV in [local, testing]` và `DEMO_LOGIN_ENABLED=true` (trả `403` trong mọi trường hợp khác).
- **Request Body:** `{ "phone_number": "0901234567", "name": "tùy chọn" }`.

### 8.3. Đăng xuất (`POST /api/v1/auth/logout`, `auth:sanctum`)
- Thu hồi Sanctum personal access token hiện tại và trả `200 OK`.

### 8.4. Cổng gửi SMS OTP (`SmsGatewayInterface`)
- Tích hợp trên 4 điểm phát hành OTP (`bank-links`, `resend-otp`, `unlink/initiate`, `unlink/resend-otp`), điều khiển qua `SMS_DRIVER=log|esms|twilio`.
- Response `data.otp` bổ sung `delivery_channel: "SMS"` và `delivery_provider`.
- Nếu cổng SMS thất bại: rollback toàn bộ transaction và trả `503 Service Unavailable` (`error_code: "SMS_DELIVERY_FAILED"`).

---

## 9. `SandboxBankingGateway`, `VNPay Sandbox`, Webhook HMAC-SHA256 & Đối soát Ngân hàng (Trục 3 · Slice P2)

### 9.1. Xử lý lỗi thanh toán ngân hàng (`POST /api/v1/orders` & `POST /api/v1/canteen/cards/{cardId}/pos-purchases`)
- Hỗ trợ thêm `payment_method = "VNPAY"` trên `POST /api/v1/orders`.
- Khi `BANKING_GATEWAY_DRIVER=sandbox`, các lỗi từ ngân hàng tự động rollback hạn mức (`spent_today` không bị trừ) nhưng ghi bền vững bản ghi vào sổ cái `payment_transactions`:
  - `422 Unprocessable Entity` (`error_code: "INSUFFICIENT_BANK_BALANCE"`): tài khoản ngân hàng liên kết không đủ số dư.
  - `502 Bad Gateway` (`error_code: "BANK_DECLINED"`): ngân hàng từ chối trích nợ.
  - `504 Gateway Timeout` (`error_code: "GATEWAY_TIMEOUT"`): timeout từ cổng ngân hàng (`payment_transactions.status = TIMEOUT`).

### 9.2. Webhook Ngân hàng Sandbox (`POST /api/v1/webhooks/banking/sandbox`)
- Xác thực bằng header `X-Gateway-Timestamp` (cửa sổ `±300s`) và `X-Gateway-Signature = HMAC-SHA256(timestamp + "." + rawBody, BANKING_WEBHOOK_SECRET)` (`hash_equals`), chống replay idempotent bằng bảng `gateway_webhook_events`.
- Hỗ trợ sự kiện: `CHARGE_SETTLED`, `CHARGE_FAILED`, `REFUND_SETTLED`.

### 9.3. Cổng thanh toán `VNPay Sandbox` (chuẩn v2.1.0 `HMAC-SHA512`)
- `POST /api/v1/payments/vnpay/create-url` (`auth:sanctum`, `throttle:20,1`): nhận `{ "order_id": 15 }` (chỉ chủ đơn `VNPAY` chưa thanh toán/thất bại), trả `{ payment_url, txn_ref, amount, expires_at, simulated }`.
- `GET /api/v1/payments/vnpay/ipn`: xác thực `vnp_SecureHash`, đối chiếu số tiền `vnp_Amount === amount * 100`, cập nhật `orders.payment_status` & `payment_transactions` idempotent (`RspCode: 00/01/02/04/97`).
- `GET /api/v1/payments/vnpay/return`: xác thực chữ ký trả về cho trình duyệt.

### 9.4. Đối soát & Cấu hình Sandbox trên Admin Blade
- `GET /admin/canteen/reconciliation` (`role:staff,admin`): xem phiên đối soát & sổ cái `payment_transactions`.
- `POST /admin/canteen/reconciliation/run` & `PATCH /admin/canteen/reconciliation/items/{item}/resolve` (`role:admin`): chạy đối soát ngày & xác nhận xử lý chênh lệch.
- `PUT /admin/canteen/cards/{studentCard}/sandbox-account` (`role:admin`): chỉnh số dư & chế độ mô phỏng tài khoản Sandbox.
- Artisan CLI: `php artisan canteen:reconcile {date?} {--provider=sandbox_epay}`.

---

## 10. Trục 3 · Slice P3 — Tồn kho Nguyên tử, POS Staff Terminal, CRUD Học sinh & RBAC 4 Vai trò

### 10.1. Tồn kho Sản phẩm (`ProductResource` & Lỗi `INSUFFICIENT_STOCK`)
- `GET /api/v1/products`, `GET /api/v1/products/{id}` trả thêm:
  - `stock_quantity` (`int|null` — `null`: không giới hạn, `0`: hết hàng, `> 0`: số phần còn lại), `low_stock_threshold` (`int`, mặc định `5`), `is_in_stock` (`bool`).
- Khi đặt đơn vượt tồn kho (`POST /api/v1/orders`, `POST /api/v1/canteen/cards/{cardId}/pos-purchases`, `POST /admin/pos/checkout`): trả `422` kèm `error_code: "INSUFFICIENT_STOCK"` và `meta: { product_id, product_name, available_stock, requested_quantity }`.
- Khi hủy đơn (`CANCELLED`): hệ thống tự động hoàn `stock_quantity` nguyên tử cùng hoàn hạn mức & số dư sandbox.

### 10.2. RBAC 4 Vai trò (`admin`, `staff`, `parent`, `student`)
- `UserResource` trả thêm `effective_role` (`admin|staff|parent|student`, quy đổi legacy `customer → parent`) và `can_manage_canteen` (`bool`).
- Tài khoản `student` (`students.student_user_id = user.id`): xem thẻ, hạn mức, lịch sử hạn mức/đơn hàng của chính mình và đặt mua trong hạn mức; bị chặn `403` (`error_code: "PARENT_ROLE_REQUIRED"`) khi gọi các endpoint tài chính phụ huynh (`POST /cards/{id}/bank-links`, `verify-otp`, `resend-otp`, `unlink/*`, `PUT /cards/{id}/spending-limit`).

### 10.3. Admin Blade POS Terminal, Quản lý Tồn kho & CRUD Học sinh
- **POS Staff Terminal (`role:staff,admin`):**
  - `GET /admin/pos` — Màn hình POS 2 cột.
  - `GET /admin/pos/lookup-card?q=<card_or_student_code>` — Tra cứu thẻ JSON (không lộ STK/CCCD đầy đủ).
  - `POST /admin/pos/checkout` — Tạo đơn POS `#POS-...` (`STUDENT_CARD_POS` hoặc `COUNTER_CASH`, bắt buộc `idempotency_key`).
  - `GET /admin/pos/receipt/{order}` — Biên lai POS thân thiện máy in (`@media print`).
- **Tồn kho (`role:admin`):** `PATCH /admin/products/{product}/stock` (`mode: set|add|unlimited`).
- **Học sinh & Cấp phát Thẻ:**
  - `GET /admin/canteen/students` (`role:staff,admin`).
  - `GET/POST /admin/canteen/students/create`, `GET/PUT/DELETE /admin/canteen/students/{student}`, `POST /admin/canteen/students/{student}/cards` (`role:admin` only).

---

## 11. Trục 3 · Slice P4 — Notification Module (In-App + Push Zalo OA/ZNS) & Security Headers

### 11.1. Customer / Parent / Student Notification API (`auth:sanctum`, IDOR-protected)
- `GET /api/v1/notifications?page=1&per_page=15&unread_only=0&type=...`: Danh sách thông báo phân trang của người dùng hiện tại kèm `meta.unread_count`.
- `GET /api/v1/notifications/unread-count`: Trả `{ data: { unread_count: int } }`.
- `PATCH /api/v1/notifications/{id}/read`: Đánh dấu đã đọc 1 thông báo (`404` nếu không thuộc tài khoản hiện tại).
- `POST /api/v1/notifications/read-all`: Đánh dấu toàn bộ thông báo chưa đọc thành đã đọc (`{ data: { marked_count, unread_count: 0 } }`).

### 11.2. Sự kiện tự động & Push Drivers
- 7 loại sự kiện: `PURCHASE_SUCCESS`, `LOW_SPENDING_LIMIT` (`<= 20%`), `PAYMENT_FAILED`, `ORDER_REFUNDED`, `ORDER_STATUS_UPDATED`, `BANK_LINK_CHANGED`, `SPENDING_LIMIT_UPDATED`.
- `NOTIFICATION_PUSH_DRIVER`: `log` (mặc định), `zalo_oa`, `zalo_zns`. Lỗi push được ghi nhận `push_status = FAILED` và không ảnh hưởng giao dịch thanh toán.

### 11.3. Admin Blade & Security Headers
- `GET /admin/canteen/notifications` (`role:staff,admin`): Nhật ký thông báo & trạng thái push.
- Global HTTP Security Headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`.
