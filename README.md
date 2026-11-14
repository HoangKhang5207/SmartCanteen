# Smart Canteen — Zalo Mini App × React × Laravel 13

Ứng dụng đặt suất ăn & quản lý đơn hàng căn tin qua Zalo Mini App, kết nối tới Laravel 13 REST API và cơ sở dữ liệu PostgreSQL độc lập (100% dữ liệu giả lập, không kết nối hệ thống Căn Tin Số production).

## Cấu trúc Monorepo

```text
.
├── api/                  # Laravel 13 REST API (PHP 8.5 + Sanctum + PostgreSQL)
├── miniapp/              # Zalo Mini App (React 18 + TypeScript + ZaUI + Jotai + TanStack Query)
├── docs/
│   ├── api-contract.md   # Hợp đồng REST API (/api/v1)
│   ├── PROJECT_STATE.md  # Trạng thái triển khai theo từng lát cắt (Vertical Slices)
│   └── superpowers/      # Đặc tả thiết kế & kế hoạch triển khai đã phê duyệt
├── canteen_logo.jpg      # Logo gốc thương hiệu Smart Canteen
└── DESIGN.md             # Hệ thống thiết kế giao diện & bảng màu thương hiệu
```

## Các tính năng đã hoàn thiện (S1 + S2)

- **Nhận diện thương hiệu Smart Canteen:** Tích hợp logo chính thức và hệ màu xanh lá (`brand-500: #5DBE46` kết hợp `brand-700: #2E7D32` đạt độ tương phản chữ trắng `5.14:1` chuẩn WCAG 2.1 AA).
- **Thực đơn & Tìm kiếm (S1):** Danh mục món ăn, tìm kiếm tiếng Việt debounce 300ms, lọc theo danh mục, hiển thị trạng thái còn món / hết món, chi tiết món ăn.
- **Xác thực khách hàng (S2):**
  - Xác minh `access_token` thật với Zalo Open API (`https://graph.zalo.me/v2.0/me` kèm `appsecret_proof` HMAC-SHA256) và cấp Laravel Sanctum Bearer token.
  - Hỗ trợ `POST /api/v1/auth/demo` khi chạy ở môi trường `local`/`testing` (`DEMO_LOGIN_ENABLED=true`) và tự động khóa (`403 Forbidden`) trên `production`.
- **Giỏ hàng & Đặt đơn an toàn (S2):**
  - Quản lý giỏ hàng trong bộ nhớ bằng Jotai (`1..20` phần tử mỗi món).
  - Tạo đơn hàng trong DB transaction (`DB::transaction` + `lockForUpdate`) kèm header `Idempotency-Key` chống tạo đơn trùng khi mạng chập chờn hoặc bấm đúp.
  - Tính tiền hoàn toàn phía server từ bảng `products` (`unit_price`, `subtotal`, `total_amount` lưu cố định theo đơn vị nguyên `VND`), bỏ qua mọi giá do client gửi lên.
- **Đơn của tôi & Chi tiết đơn (S2):**
  - Khách hàng chỉ xem được đơn của chính mình (`OrderPolicy`, trả `404` nếu truy cập đơn người khác).
  - Tự động làm mới (polling 7 giây) cho các đơn đang xử lý (`PENDING`, `PREPARING`, `READY`).

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

### 2. Khởi chạy Backend (`api/`)

```powershell
cd api
Copy-Item .env.example .env
# Cập nhật DB_PORT / DB_USERNAME / DB_PASSWORD trong .env cho đúng PostgreSQL của bạn
php artisan key:generate
php artisan migrate --seed
php artisan serve --host=127.0.0.1 --port=8000
```

Dữ liệu giả lập gồm **4 danh mục** (`Cơm`, `Bún & Phở`, `Đồ uống`, `Tráng miệng`) và **12 món ăn** (có 1 món trạng thái hết hàng để kiểm chứng giao diện).

### 3. Khởi chạy Frontend Mini App (`miniapp/`)

```powershell
cd miniapp
Copy-Item .env.example .env
npm install
npm run dev
```

Mở trình duyệt tại `http://localhost:3000/` (ứng dụng tự động đăng nhập qua `/api/v1/auth/demo` khi chạy trên trình duyệt local outside Zalo).

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
