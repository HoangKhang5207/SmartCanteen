# Smart Canteen & Phân hệ Căn tin Số — Workspace Hub

Workspace điều phối tài liệu kiến trúc, đặc tả, kiểm thử và hướng dẫn vận hành cho hệ thống **Smart Canteen & Căn tin Số** (phủ đầy **10/10 Module** theo `FEATURES.md` §9), gồm **2 sub-repository Git độc lập**:

- **`api/`** *(Sub-repo Git độc lập)*: Laravel 13 REST API (`/api/v1`) + Admin Portal (`/admin`) + Quầy thu ngân POS (`/admin/pos`) + PostgreSQL 17 + Docker + GitHub Actions CI. Xem chi tiết: [api/README.md](api/README.md).
- **`miniapp/`** *(Sub-repo Git độc lập)*: Zalo Mini App (React 18 + TypeScript + ZaUI + `zmp-sdk` + Jotai + TanStack Query + Tailwind CSS) + Docker + GitHub Actions CI. Xem chi tiết: [miniapp/README.md](miniapp/README.md).

---

## 1. Bảng So sánh 3 Chế độ Khởi chạy Dự án (`api/` + `miniapp/`)

| Tiêu chí | **Chế độ 1: Thuần Local (Host)** | **Chế độ 2: Thuần Docker Compose** | **Chế độ 3: Zalo Mobile (Testing)** |
|---|---|---|---|
| **Mục đích** | Dev nhanh trên máy đã cài PHP 8.4, PostgreSQL 17 & Node 24 | Demo/Dev chuẩn nhất — **không cần cài PHP/PostgreSQL/Node** | Quét mã QR mở trực tiếp trên **App Zalo điện thoại** |
| **Env `api/`** | `api/.env.example` → `api/.env` | `api/.env.docker.example` → `api/.env` | `api/.env.zalo.example` → `api/.env` |
| **Env `miniapp/`** | `miniapp/.env.example` → `miniapp/.env` | `miniapp/.env.docker.example` → `miniapp/.env` | `miniapp/.env.zalo.example` → `miniapp/.env` |
| **API / Admin / POS** | `http://localhost:8000` | `http://localhost:8000` | `https://<domain-https>` (Cloudflare Tunnel / VPS) |
| **Mini App** | `http://localhost:3000` | `http://localhost:3000` | Zalo Mobile App (QR Testing từ `zmp deploy`) |
| **Đăng nhập Mini App** | Demo Phone (`0901234567` / `0901234568`) | Demo Phone (`0901234567` / `0901234568`) | **SĐT Zalo thật** (`zmp-sdk getPhoneNumber()`) |

---

## 2. Hướng dẫn Khởi chạy Từng Chế độ

### Chế độ 1 — Chạy cả `api/` và `miniapp/` thuần Local trên Host

**Terminal 1 (Backend `api/` — đảm bảo PostgreSQL 17 đang chạy ở cổng `54329` hoặc sửa `DB_PORT` trong `api/.env`):**
```powershell
cd api
Copy-Item .env.example .env
composer install
php artisan key:generate
php artisan migrate:fresh --seed
php artisan serve --host=127.0.0.1 --port=8000
```

**Terminal 2 (Frontend `miniapp/`):**
```powershell
cd miniapp
Copy-Item .env.example .env
npm ci
npm run dev
```

- Mở Mini App: `http://localhost:3000` · Mở Admin/POS: `http://localhost:8000/admin/login` (`admin@smartcanteen.local` / `Admin@123456`).

---

### Chế độ 2 — Chạy cả `api/` và `miniapp/` bằng Docker / Docker Compose (Khuyến nghị)

**Terminal 1 (Backend `api/` + PostgreSQL 17 + Nginx):**
```powershell
cd api
Copy-Item .env.docker.example .env
docker compose up -d --build --wait
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate:fresh --force --seed
```

**Terminal 2 (Frontend `miniapp/` — Vite Dev Server trong Docker):**
```powershell
cd miniapp
Copy-Item .env.docker.example .env
docker compose up -d --build
```

- Mở Mini App: `http://localhost:3000` · Mở Admin/POS: `http://localhost:8000/admin/login`.
- *(Tùy chọn Production-like stack: dùng `docker compose -f compose.yaml -f compose.prod.yaml up -d --build` trong từng thư mục).*

---

### Chế độ 3 — Chạy `api/` và `miniapp/` trên Zalo Mobile (Development / Testing)

1. **Backend `api/` qua HTTPS công khai:**
   ```powershell
   cd api
   Copy-Item .env.zalo.example .env
   # Điền ZALO_APP_ID, ZALO_APP_SECRET, và domain HTTPS sau khi mở tunnel bên dưới
   docker compose up -d --build --wait
   docker compose exec app php artisan key:generate
   docker compose exec app php artisan migrate --force --seed
   ```
   Mở cửa sổ PowerShell mới để cấp HTTPS công khai cho cổng `8000`:
   ```powershell
   cloudflared tunnel --url http://127.0.0.1:8000
   ```
   Cập nhật URL `https://...trycloudflare.com` vào `APP_URL` & `VNPAY_RETURN_URL` trong `api/.env`, sau đó chạy `docker compose exec app php artisan config:clear`.

2. **Frontend `miniapp/` đẩy lên Zalo Mini App Testing:**
   ```powershell
   cd miniapp
   Copy-Item .env.zalo.example .env
   # Điền VITE_API_BASE_URL=https://...trycloudflare.com/api/v1 và APP_ID=<Zalo Mini App ID>
   npm ci
   npm run build
   npm run login
   npm run deploy
   ```
   Quét **Mã QR Testing** trên ứng dụng **Zalo điện thoại** (tài khoản phải nằm trong danh sách Testers trên Zalo Developer Center). Xem quy trình chi tiết tại [docs/releases/s5-staging-zalo-deploy-runbook.md](docs/releases/s5-staging-zalo-deploy-runbook.md).

---

## 3. Hồ sơ Tài liệu Kỹ thuật (`docs/`)

```text
docs/
├── demo-script.md                                   # Kịch bản trình diễn trực tiếp 12 bước & bảng dữ liệu mẫu
├── requirements.md                                  # Đặc tả yêu cầu 10/10 Module & RBAC 4 vai trò
├── requirements/acceptance-criteria-matrix.md       # Ma trận truy vết tiêu chí nghiệm thu MVP -> Trục 3
├── architecture/
│   ├── system-architecture.md                       # Kiến trúc hệ thống, Lock Order, Auth & State Machines
│   └── system-deep-dive-presentation-guide.md       # Cẩm nang Review Code, Review Test & Q&A cuộc họp
├── design/ui-ux-brand-guidelines.md                 # Hệ màu WCAG 2.1 AA, 11 màn hình & A11y
├── api-contract.md                                  # Hợp đồng REST API đầy đủ (§1–§11)
├── database.md                                      # ERD 18 bảng, 27 migrations & 4 seeders
├── plans/roadmap-and-slices.md                      # Lộ trình & lịch sử triển khai S1–S5, Trục 1–3
├── test-cases.md                                    # Danh mục kịch bản kiểm thử (TC-01..TC-21)
├── testing/
│   ├── test-report.md                               # Báo cáo 126 PHPUnit + 44 Vitest tests
│   └── truc3-security-e2e-audit.md                  # Báo cáo kiểm toán Bảo mật & 10/10 Module
├── releases/
│   ├── release-v1.0.0-digital-canteen.md            # Ghi chú phát hành bản hoàn thiện Căn tin Số
│   ├── release-v0.1.0-mvp.md                        # Ghi chú phát hành mốc MVP lịch sử
│   └── s5-staging-zalo-deploy-runbook.md            # Sổ tay triển khai Staging HTTPS & Zalo Mobile QR
└── PROJECT_STATE.md                                 # Trạng thái toàn dự án
```

## 4. Kiểm thử & Cổng chất lượng (Quality Gates)

```powershell
# Backend (chạy trên PostgreSQL 17 thật trong Docker)
cd api
docker compose exec app vendor/bin/pint --test
docker compose exec app php artisan test --compact

# Frontend (Vitest + TypeScript + Vite Build)
cd ../miniapp
npm test
npx tsc --noEmit
npm run build
```
