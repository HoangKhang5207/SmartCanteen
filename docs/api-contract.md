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
- **Response `200 OK`:**
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
    "name": "Khách hàng Demo"
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
      "name": "Khách hàng Demo",
      "role": "customer",
      "zalo_user_id": "demo-customer-local"
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
  1. `items`: mảng từ `1..50` phần tử; mỗi phần tử có `product_id` nguyên dương và `quantity` trong khoảng `1..20`. Các dòng trùng `product_id` được chuẩn hóa/cộng dồn số lượng và kiểm tra tổng số lượng mỗi món `<= 20`.
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

### 3.2. Danh sách đơn hàng của tôi

- **Endpoint:** `GET /api/v1/orders`
- **Xác thực:** `Bearer <token>` (`auth:sanctum`)
- **Response `200 OK`:** Danh sách `OrderResource` của riêng người dùng đang đăng nhập, sắp xếp mới nhất trước (`created_at DESC, id DESC`).

### 3.3. Chi tiết đơn hàng của tôi

- **Endpoint:** `GET /api/v1/orders/{id}`
- **Xác thực:** `Bearer <token>` (`auth:sanctum`)
- **Response `200 OK`:** `{ "data": OrderResource }`
- **Response `404 Not Found`:** Khi đơn hàng không tồn tại hoặc thuộc về người dùng khác (**AC-10** — không tiết lộ sự tồn tại của đơn hàng người khác).
