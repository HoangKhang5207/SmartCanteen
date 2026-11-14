# Thiết kế Cơ sở Dữ liệu — Smart Canteen (`docs/database.md`)

- **Hệ quản trị CSDL:** PostgreSQL `17.5`
- **Database phát triển / trình diễn:** `titkul_canteen_demo`
- **Database kiểm thử tự động:** `titkul_canteen_test`
- **Bảng mã:** `UTF-8`
- **Quy ước tiền tệ:** Tất cả các cột giá và tổng tiền (`price`, `unit_price`, `subtotal`, `total_amount`) đều lưu dưới dạng **số nguyên không dấu (`INTEGER` / `unsignedInteger`) theo đơn vị `VND`**, tuyệt đối không dùng kiểu số thực (`FLOAT` / `DOUBLE`).

---

## 1. Sơ đồ Thực thể Kết hợp (Entity-Relationship Diagram)

```mermaid
erDiagram
    USERS ||--o{ ORDERS : "đặt đơn (user_id)"
    USERS ||--o{ ORDER_STATUS_LOGS : "thực hiện đổi trạng thái (changed_by)"
    CATEGORIES ||--o{ PRODUCTS : "chứa (category_id)"
    ORDERS ||--|{ ORDER_ITEMS : "bao gồm (order_id)"
    PRODUCTS ||--o{ ORDER_ITEMS : "được tham chiếu (product_id)"
    ORDERS ||--o{ ORDER_STATUS_LOGS : "ghi nhật ký (order_id)"

    USERS {
        bigint id PK
        varchar name
        varchar email UK "nullable (dành cho staff/admin)"
        varchar password "nullable (bcrypt)"
        varchar role "customer | staff | admin"
        varchar zalo_user_id UK "nullable (ID định danh Zalo)"
        timestamp created_at
        timestamp updated_at
    }

    CATEGORIES {
        bigint id PK
        varchar name
        boolean is_active "default true"
        timestamp created_at
        timestamp updated_at
    }

    PRODUCTS {
        bigint id PK
        bigint category_id FK "restrictOnDelete"
        varchar name
        text description "nullable"
        integer price "Đơn vị: VND (> 0)"
        varchar image_url "nullable"
        boolean is_available "default true"
        timestamp created_at
        timestamp updated_at
    }

    ORDERS {
        bigint id PK
        bigint user_id FK "restrictOnDelete"
        varchar code UK "SC-YYMMDD-XXXXXX"
        varchar status "PENDING | PREPARING | READY | COMPLETED | CANCELLED"
        integer total_amount "Tổng tiền chốt phía server (VND)"
        varchar note "nullable (<= 500 ký tự)"
        varchar idempotency_key "Unique theo (user_id, idempotency_key)"
        varchar payload_hash "SHA-256 của items + note"
        integer version "Optimistic locking (default 1)"
        timestamp created_at
        timestamp updated_at
    }

    ORDER_ITEMS {
        bigint id PK
        bigint order_id FK "cascadeOnDelete"
        bigint product_id FK "restrictOnDelete"
        smallint quantity "1..20"
        integer unit_price "Đơn giá chốt lúc đặt (VND)"
        integer subtotal "unit_price * quantity (VND)"
        timestamp created_at
        timestamp updated_at
    }

    ORDER_STATUS_LOGS {
        bigint id PK
        bigint order_id FK "cascadeOnDelete"
        varchar from_status
        varchar to_status
        bigint changed_by FK "restrictOnDelete -> users.id"
        timestamp created_at
        timestamp updated_at
    }
```

---

## 2. Chi tiết Các Bảng & Ràng buộc Toàn vẹn (Tables & Constraints)

### 2.1. Bảng `users`
- **Migrations:** `0001_01_01_000000_create_users_table.php`, `2026_11_14_130001_add_role_and_zalo_to_users_table.php`
- **Đặc điểm:**
  - Khách hàng Zalo (`role = 'customer'`) được định danh qua `zalo_user_id` (`UNIQUE`, nullable), không bắt buộc có `email` hay `password`.
  - Nhân viên (`role = 'staff'`) và Quản trị viên (`role = 'admin'`) đăng nhập qua `email` (`UNIQUE`) và `password` (băm `bcrypt`).

### 2.2. Bảng `categories`
- **Migration:** `2026_11_14_101854_create_categories_table.php`
- **Đặc điểm:** Quản lý nhóm thực đơn (`Cơm`, `Bún & Phở`, `Đồ uống`, `Tráng miệng`). Khi `is_active = false`, danh mục và toàn bộ các món thuộc danh mục đó được ẩn khỏi API thực đơn công khai và không thể đặt mới.

### 2.3. Bảng `products`
- **Migration:** `2026_11_14_101855_create_products_table.php`
- **Khóa ngoại:** `category_id` tham chiếu `categories(id)` với `RESTRICT ON DELETE` (chặn xóa danh mục khi vẫn còn sản phẩm).
- **Chỉ mục:** Index trên `(category_id, is_available)`.

### 2.4. Bảng `orders`
- **Migration:** `2026_11_14_130002_create_orders_table.php`
- **Khóa ngoại:** `user_id` tham chiếu `users(id)` với `RESTRICT ON DELETE`.
- **Ràng buộc chống đơn trùng:** `UNIQUE (user_id, idempotency_key)` kết hợp cột `payload_hash` (`VARCHAR(64)`).
- **Kiểm soát đồng thời:** Cột `version` (`INTEGER`, mặc định `1`) tăng thêm `1` sau mỗi lần chuyển trạng thái thành công.

### 2.5. Bảng `order_items`
- **Migration:** `2026_11_14_130003_create_order_items_table.php`
- **Khóa ngoại:**
  - `order_id` tham chiếu `orders(id)` với `CASCADE ON DELETE`.
  - `product_id` tham chiếu `products(id)` với `RESTRICT ON DELETE` (không cho phép xóa sản phẩm đã từng nằm trong lịch sử đơn hàng).
- **Bảo toàn lịch sử giá (AC-11):** Lưu bản chụp `unit_price` và `subtotal` tại thời điểm đặt đơn.

### 2.6. Bảng `order_status_logs`
- **Migration:** `2026_11_14_150001_create_order_status_logs_table.php`
- **Khóa ngoại:**
  - `order_id` tham chiếu `orders(id)` với `CASCADE ON DELETE`.
  - `changed_by` tham chiếu `users(id)` với `RESTRICT ON DELETE`.
- **Chỉ mục:** Index trên `(order_id, id)` phục vụ truy xuất lịch sử chuyển trạng thái theo trình tự thời gian.

---

## 3. Dữ liệu Khởi tạo Mẫu (Seeders)

Chạy lệnh `php artisan migrate --seed` sẽ thực thi 2 Seeder idempotent (có thể chạy lặp lại nhiều lần không gây trùng dữ liệu):

1. **`MenuSeeder` (`api/database/seeders/MenuSeeder.php`):**
   - Khởi tạo **4 danh mục** (`Cơm`, `Bún & Phở`, `Đồ uống`, `Tráng miệng`).
   - Khởi tạo **12 món ăn** với giá từ `15.000 ₫` đến `55.000 ₫` (trong đó có món *"Bún bò Huế đặc biệt"* được đặt `is_available = false` để kiểm chứng trực tiếp giao diện "Hết món" và quy tắc chặn đặt món ngừng bán **AC-05**).
2. **`StaffSeeder` (`api/database/seeders/StaffSeeder.php`):**
   - Tài khoản Admin: `admin@smartcanteen.local` / mật khẩu `Admin@123456` (`role = admin`).
   - Tài khoản Staff: `staff@smartcanteen.local` / mật khẩu `Staff@123456` (`role = staff`).
