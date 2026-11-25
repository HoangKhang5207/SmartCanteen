# Smart Canteen — Workspace Hub (Zalo Mini App × React × Laravel 13)

Workspace điều phối tài liệu kiến trúc, đặc tả và kiểm thử cho hệ thống **Smart Canteen** gồm **2 sub-repository Git độc lập**:

- **`api/`** (Sub-repo độc lập): Laravel 13 REST API (`/api/v1`) + Cổng Quản trị Blade (`/admin`) + PostgreSQL 17 + Docker + GitHub Actions CI. Xem chi tiết tại `api/README.md`.
- **`miniapp/`** (Sub-repo độc lập): Zalo Mini App (React 18 + TypeScript + ZaUI + Jotai + TanStack Query + Tailwind CSS) + Docker + GitHub Actions CI. Xem chi tiết tại `miniapp/README.md`.

> **Lưu ý cấu trúc Git:** Từ phiên bản nâng cấp nền tảng (Trục 1), `api/` và `miniapp/` đã được tách thành 2 Git repository độc lập có bảo toàn trọn vẹn lịch sử commit S1–S5 của từng bên và được khai báo trong `.gitignore` của workspace cha. Workspace cha lưu giữ hồ sơ tài liệu dùng chung (`docs/`, `DESIGN.md`, `canteen_logo.jpg`).

## Cấu trúc Workspace & Hồ sơ Tài liệu Kỹ thuật

```text
.
├── api/                                # [Sub-repo Git độc lập] Laravel 13 + PostgreSQL 17 + Docker + CI
├── miniapp/                            # [Sub-repo Git độc lập] Zalo Mini App React/TS + Docker + CI
├── docs/
│   ├── requirements.md                 # Đặc tả yêu cầu nghiệp vụ & kỹ thuật (P0 / P1 / P2)
│   ├── requirements/
│   │   └── acceptance-criteria-matrix.md # Ma trận truy vết tiêu chí nghiệm thu (AC-01 .. AC-16)
│   ├── architecture/
│   │   └── system-architecture.md      # Kiến trúc hệ thống, luồng Auth, Idempotency & State Machine
│   ├── design/
│   │   └── ui-ux-brand-guidelines.md   # Hệ màu thương hiệu chuẩn WCAG 2.1 AA & quy chuẩn UI/UX
│   ├── api-contract.md                 # Hợp đồng REST API (/api/v1)
│   ├── database.md                     # Sơ đồ ERD, mô tả 7 bảng dữ liệu & Seeders
│   ├── plans/
│   │   └── roadmap-and-slices.md       # Lộ trình & các lát cắt dọc S1–S5
│   ├── test-cases.md                   # Danh mục 21 kịch bản kiểm thử trọng yếu (TC-01 .. TC-21)
│   ├── testing/
│   │   └── test-report.md              # Báo cáo kết quả kiểm thử tự động
│   ├── demo-script.md                  # Kịch bản trình diễn trực tiếp 8 bước
│   ├── releases/
│   │   ├── release-v0.1.0-mvp.md       # Ghi chú phát hành phiên bản v0.1.0-mvp
│   │   └── s5-staging-zalo-deploy-runbook.md # Sổ tay triển khai HTTPS & Zalo Testing QR
│   ├── PROJECT_STATE.md                # Trạng thái triển khai theo từng lát cắt
│   └── superpowers/                    # Đặc tả thiết kế & kế hoạch triển khai chi tiết
├── canteen_logo.jpg                    # Logo gốc thương hiệu Smart Canteen
└── DESIGN.md                           # Hệ thống thiết kế giao diện & Design Tokens
```

## Khởi chạy nhanh Toàn Hệ thống với Docker (Khuyến nghị)

### 1. Khởi chạy Backend API + PostgreSQL (`api/`)

```powershell
cd api
Copy-Item .env.docker.example .env
docker compose up -d --build
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate --seed
```

- **REST API:** `http://localhost:8000/api/v1` (Healthcheck: `http://localhost:8000/up`)
- **Cổng Quản trị Admin/Staff:** `http://localhost:8000/admin/login`
  - Admin mẫu: `admin@smartcanteen.local` / `Admin@123456`
  - Staff mẫu: `staff@smartcanteen.local` / `Staff@123456`

### 2. Khởi chạy Frontend Mini App (`miniapp/`)

```powershell
cd miniapp
Copy-Item .env.example .env
docker compose up -d --build
```

- **Mini App (Local):** `http://localhost:3000/`

### 3. Chế độ Production-like (Local Staging Verification)

```powershell
# Backend (immutable non-root PHP-FPM + Nginx unprivileged)
cd api
docker compose -f compose.yaml -f compose.prod.yaml up -d --build

# Frontend (multi-stage build -> Nginx unprivileged + /healthz)
cd ../miniapp
docker compose -f compose.yaml -f compose.prod.yaml up -d --build
```

## Kiểm thử & Cổng chất lượng (Quality Gates)

```powershell
# Backend (trong Docker container hoặc host PHP 8.4+)
cd api
docker compose exec app vendor/bin/pint --test
docker compose exec app php artisan test

# Frontend (Vitest + TypeScript + Vite Production Build)
cd ../miniapp
npm test
npx tsc --noEmit
npm run build
```
