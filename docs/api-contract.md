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
- **Lỗi chuẩn:** `404`, `409`, `429` (`OTP_RESEND_COOLDOWN` kèm `meta.retry_after_seconds`, hoặc `OTP_MAX_RESENDS_EXCEEDED`).
