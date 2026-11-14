# Smart Canteen — Zalo Mini App × React × Laravel 13

Ứng dụng đặt suất ăn & quản lý đơn hàng căn tin qua Zalo Mini App, kết nối tới Laravel 13 REST API + Cổng Quản trị Blade và cơ sở dữ liệu PostgreSQL độc lập (100% dữ liệu giả lập, không kết nối hệ thống Căn Tin Số production).

## Cấu trúc Monorepo

```text
.
├── api/                  # Laravel 13 REST API + Admin Blade Portal (PHP 8.5 + Sanctum + PostgreSQL)
├── miniapp/              # Zalo Mini App (React 18 + TypeScript + ZaUI + Jotai + TanStack Query)
├── docs/
│   ├── api-contract.md   # Hợp đồng REST API (/api/v1)
│   ├── PROJECT_STATE.md  # Trạng thái triển khai theo từng lát cắt (Vertical Slices)
│   └── superpowers/      # Đặc tả thiết kế & kế hoạch triển khai đã phê duyệt
├── canteen_logo.jpg      # Logo gốc thương hiệu Smart Canteen
└── DESIGN.md             # Hệ thống thiết kế giao diện & bảng màu thương hiệu
```

## Các tính năng đã hoàn thiện (S1 + S2 + S3)

- **Nhận diện thương hiệu Smart Canteen:** Tích hợp logo chính thức và hệ màu xanh lá (`brand-500: #5DBE46` kết hợp `brand-700: #2E7D32` đạt độ tương phản chữ trắng `5.14:1` chuẩn WCAG 2.1 AA) đồng bộ trên cả Zalo Mini App và Cổng Quản trị Laravel Blade.
- **Thực đơn & Tìm kiếm (S1):** Danh mục món ăn, tìm kiếm tiếng Việt debounce 300ms, lọc theo danh mục, hiển thị trạng thái còn món / hết món, chi tiết món ăn.
- **Xác thực khách hàng (S2):**
  - Xác minh `access_token` thật với Zalo Open API (`https://graph.zalo.me/v2.0/me` kèm `appsecret_proof` HMAC-SHA256) và cấp Laravel Sanctum Bearer token.
  - Hỗ trợ `POST /api/v1/auth/demo` khi chạy ở môi trường `local`/`testing` (`DEMO_LOGIN_ENABLED=true`) và tự động khóa (`403 Forbidden`) trên `production`.
- **Giỏ hàng & Đặt đơn an toàn (S2):**
  - Quản lý giỏ hàng trong bộ nhớ bằng Jotai (`1..20` phần tử mỗi món).
  - Tạo đơn hàng trong DB transaction (`DB::transaction` + `lockForUpdate`) kèm header `Idempotency-Key` chống tạo đơn trùng.
  - Tính tiền hoàn toàn phía server từ bảng `products` (`unit_price`, `subtotal`, `total_amount` lưu cố định theo đơn vị nguyên `VND`), bỏ qua mọi giá do client gửi lên.
- **Đơn của tôi & Chi tiết đơn (S2):**
  - Khách hàng chỉ xem được đơn của chính mình (`OrderPolicy`, trả `404` nếu truy cập đơn người khác).
  - Tự động làm mới (polling 7 giây) cho các đơn đang xử lý (`PENDING`, `PREPARING`, `READY`).
- **Hệ thống Quản trị Admin/Staff (S3):**
  - Xác thực Session Auth (`/admin/login`) và phân quyền RBAC middleware `role:staff,admin` (chặn `customer` với `403 Forbidden` trên cả Web lẫn REST API `/api/v1/admin/*`).
  - Dashboard thống kê hôm nay (`/admin` và `GET /api/v1/admin/statistics`): tổng đơn hôm nay, đơn đang xử lý, doanh thu các đơn `COMPLETED` hôm nay, phân bổ theo trạng thái (tự động làm mới mỗi 10 giây).
  - Bảng điều phối đơn hàng (`/admin/orders`, `/admin/orders/{order}` và `PATCH /api/v1/admin/orders/{id}/status`): kiểm tra chặt chẽ State Machine, kiểm soát đồng thời bằng Optimistic Locking (`version`, trả `409 Conflict` khi xung đột), và ghi lịch sử `order_status_logs`.
  - Quản lý Thực đơn (`/admin/categories`, `/admin/products`): CRUD Danh mục & Món ăn, bảo toàn nguyên vẹn giá lịch sử của các đơn hàng cũ (**AC-11**) và chặn xóa danh mục/món ăn đã có ràng buộc dữ liệu.

## Yêu cầu môi trường

- **PHP:** `8.5.x` (Laravel Herd, bật `pdo_pgsql`, `mbstring`, `intl`, `bcmath`)
- **Composer:** `2.x`
- **PostgreSQL:** `17.x`
- **Node.js:** `20+` / `24.x` & **npm** `10+`

## Hướng dẫn khởi chạy Local

### 1. Khởi chạy PostgreSQL

Nếu máy bạn đã có PostgreSQL chạy ở cổng `5432`, tạo 2 database `titkul_canteen_demo` và `titkul_canteen_test` thuộc quyền `demo_user`.

Nếu sử dụng cụm PostgreSQL cục bộ độc lập trong `.local/pgdata` (cổng `54329`):

```powershell
D:\DB_Design\PostgreSQL\pgsql\bin\postgres.exe -D .local\pgdata -p 54329 -c listen_addresses=127.0.0.1
```

### 2. Khởi chạy Backend & Cổng Quản trị (`api/`)

```powershell
cd api
Copy-Item .env.example .env
# Cập nhật DB_PORT / DB_USERNAME / DB_PASSWORD trong .env cho đúng PostgreSQL của bạn
php artisan key:generate
php artisan migrate --seed
php artisan serve --host=127.0.0.1 --port=8000
```

- **REST API:** `http://127.0.0.1:8000/api/v1`
- **Cổng Quản trị Admin/Staff (Blade):** `http://127.0.0.1:8000/admin/login`
  - Tài khoản Admin mẫu: `admin@smartcanteen.local` / `Admin@123456`
  - Tài khoản Staff mẫu: `staff@smartcanteen.local` / `Staff@123456`

### 3. Khởi chạy Frontend Mini App (`miniapp/`)

```powershell
cd miniapp
Copy-Item .env.example .env
npm install
npm run dev
```

Mở trình duyệt tại `http://localhost:3000/` (ứng dụng tự động đăng nhập qua `/api/v1/auth/demo` khi chạy trên trình duyệt local ngoài Zalo).

## Kiểm thử & Cổng chất lượng (Quality Gates)

```powershell
# Backend (PHPUnit trên PostgreSQL + Laravel Pint)
cd api
php artisan test
vendor\bin\pint --test

# Frontend (Vitest + TypeScript + Vite Production Build)
cd ../miniapp
npm test
npx tsc --noEmit
npm run build
```
