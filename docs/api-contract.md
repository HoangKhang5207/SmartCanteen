# Hợp đồng REST API — Smart Canteen (`/api/v1`)

- **Base URL (Local):** `http://localhost:8000/api/v1`
- **Định dạng:** JSON (`Accept: application/json`)
- **Tiền tệ:** Số nguyên dương theo đơn vị `VND` (`integer`, không dùng `float`).
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
