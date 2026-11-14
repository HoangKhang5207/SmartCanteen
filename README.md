# Smart Canteen — Zalo Mini App × React × Laravel 13

Ứng dụng đặt suất ăn & quản lý đơn hàng căn tin qua Zalo Mini App, kết nối tới Laravel 13 REST API và cơ sở dữ liệu PostgreSQL độc lập (100% dữ liệu giả lập, không kết nối hệ thống Căn Tin Số production).

## Cấu trúc Monorepo

```text
.
├── api/                  # Laravel 13 REST API (PHP 8.5 + PostgreSQL)
├── miniapp/              # Zalo Mini App (React 18 + TypeScript + ZaUI + TanStack Query)
├── docs/
│   ├── api-contract.md   # Hợp đồng REST API
│   ├── PROJECT_STATE.md  # Trạng thái triển khai theo từng lát cắt (Vertical Slices)
│   └── superpowers/      # Đặc tả thiết kế & kế hoạch triển khai đã phê duyệt
└── DESIGN.md             # Hệ thống thiết kế giao diện (Documented Assumptions)
```

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

Mở trình duyệt tại `http://localhost:3000/` (hoặc dùng `npm start` / Zalo Mini App Studio khi đã đăng nhập tài khoản Zalo Developer).

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
