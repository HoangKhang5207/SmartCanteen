# Smart Canteen — Thiết kế Nâng cấp Nền tảng: Tách 2 Sub-repo, Docker hóa & GitHub Actions CI (Trục 1)

- **Ngày:** 2026-11-25
- **Trạng thái:** **APPROVED 2026-11-25** bởi người dùng.
- **Quyết định đã chốt từ người dùng:** `1A` (Trục 1 trước, Trục 2 sau) · `2A` (Tách Git độc lập ngay tại `api/` và `miniapp/` bảo toàn lịch sử bằng `git subtree split`) · `3A` (Mở rộng & hợp nhất phân hệ Căn tin Số vào Smart Canteen hiện có ở Trục 2).
- **Môi trường mục tiêu:** Local / Development / Testing / Staging-ready (KHÔNG push/merge/deploy lên remote hay production).
- **Ký hiệu:** `[C]` CONFIRMED · `[I]` INFERRED · `[A]` ASSUMPTION (quyết định kỹ thuật của agent, có thể điều chỉnh)

---

## 1. Mục tiêu & Lý do (Intent & Why)

Smart Canteen hiện đã hoàn thiện MVP đặt món S1–S5 trong một monorepo duy nhất (`api/` và `miniapp/` nằm chung một `.git`, chạy thủ công qua Herd PHP + PostgreSQL local + Vite, chưa có Docker và chưa có CI).

Để nâng cấp dự án từ mức demo lên chuẩn **Production-Ready Engineering** (trên môi trường Local/Dev/Testing) trước khi xây dựng phân hệ **Căn tin Số** ([FEATURES.md](../../../FEATURES.md)):

1. `[C]` **Tách độc lập vòng đời mã nguồn:** `api/` (Laravel 13 Backend + Blade Admin) và `miniapp/` (Zalo Mini App React/TS) trở thành 2 Git repository riêng biệt ngay tại vị trí hiện tại, vẫn giữ nguyên lịch sử commit liên quan của từng bên để sau này gắn lên 2 remote GitHub độc lập.
2. `[C]` **Container hóa chuyên nghiệp (Docker):** Mỗi sub-repo tự đóng gói đầy đủ `Dockerfile` (multi-stage, non-root, healthcheck), `compose.yaml` (dev/test) và `compose.prod.yaml` (production-like) để bất kỳ máy nào cài Docker đều chạy & kiểm thử nhất quán trong 1 lệnh mà không phụ thuộc môi trường host.
3. `[C]` **Tự động hóa kiểm soát chất lượng (GitHub Actions CI):** Mỗi sub-repo có workflow `.github/workflows/ci.yml` độc lập chạy Lint/Type-check, Automated Tests (PHPUnit trên PostgreSQL 17 thật / Vitest), Security Audit và kiểm chứng build Docker image.

---

## 2. Phạm vi, Ngoài phạm vi & Anti-Goals

### Goals (Trong phạm vi Trục 1)
- Tách `api/` và `miniapp/` thành 2 sub-repo có `.git` riêng với nhánh `main`, giữ nguyên commit history từng thư mục via `git subtree split`, không mất bất kỳ file local nào (`.env`, `vendor/`, `node_modules/`, ảnh sản phẩm).
- Cập nhật `.gitignore` và tài liệu của repo cha (`TestPart2`) để repo cha đóng vai trò **Workspace Hub** (chứa `docs/`, `DESIGN.md`, `README.md`), bỏ theo dõi `api/` và `miniapp/` trong index repo cha (`git rm -r --cached` — không xóa file trên đĩa).
- Bổ sung bộ Docker chuẩn chỉnh cho riêng `api/` (`Dockerfile`, `.dockerignore`, `compose.yaml`, `compose.prod.yaml`, `docker/nginx/default.conf`, `docker/php/php.ini`, `docker/php/opcache.ini`, `docker/postgres/init-databases.sh`, `docker/entrypoint.sh`, `.env.docker.example`).
- Bổ sung bộ Docker chuẩn chỉnh cho riêng `miniapp/` (`Dockerfile`, `.dockerignore`, `compose.yaml`, `compose.prod.yaml`, `docker/nginx/default.conf`).
- Bổ sung GitHub Actions CI chuẩn chỉnh cho riêng `api/.github/workflows/ci.yml` và `miniapp/.github/workflows/ci.yml`.
- Bổ sung tài liệu hướng dẫn chạy Docker & gắn remote GitHub trong `api/README.md`, `miniapp/README.md`, `README.md` và cập nhật `docs/PROJECT_STATE.md`.

### Non-Goals (Ngoài phạm vi Trục 1)
- Chưa viết nghiệp vụ phân hệ Căn tin Số (`FEATURES.md`) trong đợt này — sẽ triển khai ngay ở **Trục 2** (Slices D1–D3) sau khi Trục 1 hoàn tất và được kiểm chứng.
- Không đẩy code (`git push`), không tạo remote GitHub tự động, không triển khai lên production/staging cloud.
- Không thêm Redis, Kafka, RabbitMQ, Kubernetes hay microservices không cần thiết.

### Anti-Goals (Những gì tuyệt đối không để xảy ra)
- Không làm mất file untracked ở root (`FEATURES.md`, `2026-11-25 13-33-34.mp4`) hay file `.env` cục bộ của `api/` và `miniapp/`.
- Không đưa secrets (`.env`, `APP_KEY`, `ZALO_APP_SECRET`, mật khẩu thật) vào Docker image, `compose.yaml`, workflow CI hoặc git commit.
- Không chạy container production dưới quyền `root`.
- Không làm hỏng bất kỳ bài kiểm thử hiện có nào (52 PHPUnit tests + 18 Vitest tests).

---

## 3. Ràng buộc Kỹ thuật đã Xác minh (Verified Constraints)

| Thành phần | Ràng buộc thực tế từ codebase | Quyết định phiên bản chuẩn |
|---|---|---|
| **Backend PHP** | `api/composer.lock` yêu cầu tối thiểu `php >= 8.4.1` (Laravel 13 + PHPUnit 12.5) | **PHP 8.4** (`php:8.4-fpm-alpine` trong Docker & `php-version: '8.4'` trong CI) |
| **PHP Extensions** | Laravel 13 + PostgreSQL + Pint + PHPUnit | `pdo_pgsql`, `pgsql`, `bcmath`, `intl`, `mbstring`, `opcache`, `zip`, `pcntl` |
| **Database** | `docs/database.md` & `api/phpunit.xml` dùng PostgreSQL 17 (`titkul_canteen_demo` + `titkul_canteen_test`) | `postgres:17-alpine` |
| **Backend Health Endpoint** | `api/bootstrap/app.php` đã cấu hình `health: '/up'` | Dùng `/up` cho healthcheck cả Nginx & PHP-FPM |
| **Backend Views** | Blade admin views không dùng `@vite` (không cần Node.js trong image `api`) | Image `api` thuần PHP-FPM + Nginx (siêu nhẹ, không cài Node) |
| **Frontend Node** | Máy local dùng Node `v24.16.0` / npm `11.13.0`; `miniapp/package.json` dùng Vite 5 + TS 5.9 + Vitest 2.1 | **Node 24** (`node:24-alpine` trong Docker & `node-version: '24'` trong CI) |
| **Frontend Output** | `miniapp/package.json` script `build`: `vite build . --outDir www --emptyOutDir` | Copy `/app/www` sang Nginx static container |
| **Git Safety** | Cấm `git push`, `git merge`, hoặc viết lại lịch sử nhánh `main` của repo cha | Dùng `git subtree split` (chỉ đọc lịch sử cha, không rewrite `main`) + commit mới tách tracking |

---

## 4. Các Phương án đã Cân nhắc (Approaches)

### 4.1. Cách tách 2 Sub-repo
1. **Phương án 2A (Đã chọn):** Khởi tạo Git độc lập ngay tại `api/` và `miniapp/` dùng `git subtree split` bảo toàn lịch sử commit riêng từng thư mục; bỏ theo dõi 2 thư mục khỏi repo cha bằng `git rm -r --cached` + `.gitignore`.
   - *Ưu điểm:* Giữ nguyên đường dẫn quen thuộc, bảo toàn lịch sử phát triển S1–S5 của từng sub-repo, không phụ thuộc sẵn remote URL như Submodule.
   - *Đánh đổi:* Repo cha không tự đồng bộ commit của 2 sub-repo (người dùng commit/push từng sub-repo độc lập — đúng bản chất tách 2 repo).
2. **Phương án 2B (Đã loại):** Git Submodules — yêu cầu sẵn 2 URL remote GitHub và thao tác phức tạp hơn khi dev hàng ngày.
3. **Phương án 2C (Đã loại):** Tách ra 2 thư mục ngang hàng ngoài `TestPart2` — làm rời rạc workspace hiện tại và phá vỡ các đường dẫn tham chiếu.

### 4.2. Kiến trúc Docker cho `api/`
1. **Tách Container theo Single-Responsibility (Đã chọn `[A]`):** `app` (`php:8.4-fpm-alpine` non-root) + `nginx` (`nginxinc/nginx-unprivileged:1.27-alpine` non-root) + `postgres` (`postgres:17-alpine`).
   - *Ưu điểm:* Chuẩn production container (1 process/container, không cần `supervisord`, log đẩy thẳng ra `stdout`/`stderr`, healthcheck tách bạch từng tầng, dễ mở rộng).
2. **All-in-one container PHP-FPM + Nginx + Supervisord (Đã loại):** Khó quản lý signal/logs và vi phạm best practice single-process.

---

## 5. Thiết kế Chi tiết Trục 1

### 5.1. Kiến trúc Tách 2 Sub-repo (`api/` & `miniapp/`)

Quy trình thực hiện an toàn (zero data loss, không rewrite lịch sử repo cha):
1. Tại repo cha (`d:\Coding\TitKul_ProjectTest\TestPart2`):
   - Tách lịch sử chỉ đọc:
     - `git subtree split --prefix=api -b split/api`
     - `git subtree split --prefix=miniapp -b split/miniapp`
2. Với `api/`:
   - `git init -b main` ngay trong `api/`.
   - Lấy trọn vẹn lịch sử từ nhánh `split/api` của repo cha bằng `git fetch .. split/api:main` (sau khi xử lý an toàn trạng thái working tree — giữ nguyên toàn bộ file untracked/ignored như `.env`, `vendor/`), reset index sạch về `HEAD` mà không chạm file local.
   - Xác minh `git log --oneline` trong `api/` chứa đầy đủ commit S1–S5 của riêng `api/` và `git status` sạch (không track `.env` hay `vendor/`).
3. Với `miniapp/`:
   - Thực hiện tương tự với `split/miniapp`, giữ nguyên `.env` và `node_modules/`.
   - Xác minh `git log --oneline` trong `miniapp/` chứa đầy đủ commit S1–S5 của riêng `miniapp/` và `git status` sạch.
4. Tại repo cha (`TestPart2`):
   - Xóa nhánh tạm `split/api` và `split/miniapp` (`git branch -D split/api split/miniapp`).
   - Gỡ theo dõi `api/` và `miniapp/` khỏi Git index repo cha bằng `git rm -r --cached api miniapp` (chỉ xóa khỏi index, **tuyệt đối không xóa trên ổ đĩa**).
   - Cập nhật `.gitignore` gốc để ignore `/api/` và `/miniapp/` (kèm chú thích rõ rằng đây là 2 Git sub-repo độc lập), giữ nguyên `FEATURES.md` và `.mp4` untracked.
   - Tạo 1 commit gọn gàng trên repo cha: `chore(repo): detach api and miniapp into independent git sub-repos`.

---

### 5.2. Container hóa chuẩn Production-Ready cho `api/`

#### Cấu trúc tệp mới trong `api/`
```text
api/
├── Dockerfile                      # Multi-stage: composer-deps -> runtime-base -> dev / production
├── .dockerignore                   # Loại trừ .git, vendor, node_modules, .env*, storage logs, caches
├── .env.docker.example             # Biến môi trường mẫu chạy trong Docker Compose
├── compose.yaml                    # Local / Dev / Test stack (postgres + app[dev] + nginx)
├── compose.prod.yaml               # Production-like override (immutable app[production] + nginx, no bind mount)
└── docker/
    ├── entrypoint.sh               # Startup hook: đảm bảo writable storage, optional config/route cache
    ├── nginx/
    │   ├── Dockerfile              # Immutable production Nginx image (copy config + public/ static assets)
    │   └── default.conf            # Reverse proxy -> app:9000, security headers, static /images/ cache, /up
    ├── php/
    │   ├── php.ini                 # expose_php=Off, upload/memory limits, timezone Asia/Ho_Chi_Minh
    │   ├── opcache.ini             # OPcache production settings (validate_timestamps configurable via env)
    │   └── www.conf                # PHP-FPM pool non-root (appuser) + /fpm-ping health endpoint
    └── postgres/
        └── init-databases.sh       # Tự động tạo cả 2 DB: titkul_canteen_demo & titkul_canteen_test
```

#### Chi tiết kỹ thuật `api/Dockerfile`
- **Stage `composer-deps` (`composer:2.8`):** Copy `composer.json` & `composer.lock` trước để tận dụng layer cache; cài `composer install --no-dev --no-interaction --prefer-dist --no-scripts --no-autoloader`, sau đó copy mã nguồn cần thiết (`app/`, `bootstrap/`, `config/`, `database/`, `routes/`, `artisan`) và chạy `composer dump-autoload --optimize --classmap-authoritative --no-dev` ngay trong stage này (giữ `production` image hoàn toàn không chứa binary `composer`).
- **Stage `runtime-base` (`php:8.4-fpm-alpine`):**
  - Cài runtime libs (`postgresql-libs`, `icu-libs`, `libzip`, `fcgi`) và build PHP extensions (`pdo_pgsql`, `pgsql`, `bcmath`, `intl`, `opcache`, `zip`, `pcntl`), sau đó xóa build deps để image gọn nhẹ.
  - Tạo non-root user `appuser` (UID `1000`, GID `1000`), cấu hình PHP-FPM pool (`docker/php/www.conf`) chạy dưới `appuser`, listen `0.0.0.0:9000`, bật `/fpm-ping` cho healthcheck nhẹ qua `cgi-fcgi`.
  - Copy `docker/php/php.ini`, `docker/php/opcache.ini`, `docker/entrypoint.sh` (`chmod +x`).
- **Stage `dev` (từ `runtime-base`):**
  - Copy `composer` binary từ `composer:2.8`.
  - Bật `OPCACHE_VALIDATE_TIMESTAMPS=1` để hot-reload PHP code qua bind-mount khi dev & chạy `php artisan test` / `vendor/bin/pint`.
- **Stage `production` (từ `runtime-base`):**
  - Copy `vendor/` (đã dump-autoload tối ưu) từ `composer-deps` và mã nguồn ứng dụng (không chứa `tests/`, `.git/`, `.env`, `composer`).
  - Set quyền sở hữu `storage/` & `bootstrap/cache/` cho `appuser:appuser`.
  - `USER appuser` + `HEALTHCHECK` kiểm tra `/fpm-ping` qua `cgi-fcgi`.

#### Chi tiết `api/compose.yaml` & `api/compose.prod.yaml`
- `postgres`: `postgres:17-alpine`, env cấu hình qua `${DB_DATABASE:-titkul_canteen_demo}`, `${DB_USERNAME:-demo_user}`, `${DB_PASSWORD:-demo_secret_local}`, mount `docker/postgres/init-databases.sh` vào `/docker-entrypoint-initdb.d/` để tự tạo thêm `titkul_canteen_test`, healthcheck `pg_isready`, named volume `api_pgdata`.
- `app`: build target `dev` (mặc định), bind-mount source (với named volume cho `vendor` để không xung đột host Windows/Linux), depends_on `postgres: condition: service_healthy`.
- `nginx`: `nginxinc/nginx-unprivileged:1.27-alpine` (non-root, port nội bộ `8080`, publish `${API_PORT:-8000}:8080`), mount `docker/nginx/default.conf` & `./public:ro` trong dev, healthcheck `wget -qO- http://127.0.0.1:8080/up`.
- `compose.prod.yaml`: override `app` dùng target `production` và `nginx` build từ `docker/nginx/Dockerfile` (chứa sẵn `public/` + `default.conf`, không bind-mount từ host), set `APP_ENV=production`, `APP_DEBUG=false`, `DEMO_LOGIN_ENABLED=false`, `restart: unless-stopped`.

---

### 5.3. Container hóa chuẩn Production-Ready cho `miniapp/`

#### Cấu trúc tệp mới trong `miniapp/`
```text
miniapp/
├── Dockerfile                      # Multi-stage: deps -> dev / builder -> production (nginx-unprivileged)
├── .dockerignore                   # Loại trừ .git, node_modules, www, .env*, coverage
├── compose.yaml                    # Local / Dev stack (Vite dev server :3000)
├── compose.prod.yaml               # Production-like static build trên Nginx non-root (:8080)
└── docker/
    └── nginx/
        └── default.conf            # SPA fallback, gzip, immutable asset cache, no-cache index.html, /healthz
```

#### Chi tiết kỹ thuật `miniapp/Dockerfile`
- **Stage `deps` (`node:24-alpine`):** Copy `package.json`, `package-lock.json` → `npm ci --no-audit --no-fund`.
- **Stage `dev` (từ `deps`):** Copy source, chạy dưới non-root `USER node`, `EXPOSE 3000`, `CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "3000"]`.
- **Stage `builder` (từ `deps`):** Nhận `ARG VITE_API_BASE_URL=http://localhost:8000/api/v1`, copy source, chạy `npx tsc --noEmit && npm run build` (xuất bundle vào `/app/www`).
- **Stage `production` (`nginxinc/nginx-unprivileged:1.27-alpine`):**
  - Chạy mặc định dưới non-root user `nginx` (UID `101`), listen port `8080`.
  - Copy `docker/nginx/default.conf` và `/app/www` từ `builder`.
  - Endpoint `GET /healthz` trả `200 OK` (`"ok"`), dùng trực tiếp cho `HEALTHCHECK`.
  - Security headers (`X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`) — không gắn `X-Frame-Options: DENY` cứng để không chặn môi trường Zalo WebView / iframe preview khi cần.

---

### 5.4. GitHub Actions CI chuẩn Production-Ready cho 2 Sub-repo

#### A. `api/.github/workflows/ci.yml`
- **Triggers:** `push` (`main`, `develop`), `pull_request` (`main`, `develop`), `workflow_dispatch`.
- **Security & Cost controls:** `permissions: { contents: read }`, `concurrency` với `cancel-in-progress: true`, `timeout-minutes` trên mọi job.
- **Jobs:**
  1. `code-quality`: `shivammathur/setup-php@v2` (PHP `8.4`, extensions `pdo_pgsql, pgsql, mbstring, intl, bcmath, zip`, Composer cache) → `composer validate --strict` → `composer install --no-interaction --prefer-dist` → `vendor/bin/pint --test`.
  2. `tests`: Service container `postgres:17-alpine` (`POSTGRES_DB: titkul_canteen_test`, `POSTGRES_USER: demo_user`, `POSTGRES_PASSWORD: ci_test_password`, healthcheck `pg_isready`) → `php artisan key:generate --force` → `php artisan migrate --force` → `php artisan test --compact`.
  3. `security`: `composer audit --locked --no-interaction` (kiểm tra lỗ hổng bảo mật trong dependency lockfile).
  4. `docker`: `docker/setup-buildx-action@v3` → `docker/build-push-action@v6` (`push: false`, cache GHA) kiểm chứng build cả 2 target `dev` và `production` của `api/Dockerfile` + chạy smoke test PHP-FPM / `php artisan --version` trên production image.

#### B. `miniapp/.github/workflows/ci.yml`
- **Triggers, Security & Cost controls:** Tương tự `api` (`permissions: { contents: read }`, `concurrency`, `timeout-minutes`).
- **Jobs:**
  1. `quality-and-test`: `actions/setup-node@v4` (Node `24`, `cache: npm`) → `npm ci` → `npx tsc --noEmit` → `npm test` → `npm run build` (kèm check tồn tại `www/index.html`).
  2. `security`: `npm audit --audit-level=high` (cấu hình rõ cơ chế báo cáo/ngưỡng dựa trên kết quả kiểm tra thực tế `npm audit` hiện tại của `zmp-ui` để không tạo false failure hoặc bỏ sót lỗ hổng mới).
  3. `docker`: `docker/setup-buildx-action@v3` → build kiểm chứng target `production` của `miniapp/Dockerfile` (`push: false`, `load: true`) → khởi chạy container tạm & verify `curl -fsS http://127.0.0.1:8080/healthz` trả `200 OK` và `index.html` phục vụ thành công.

---

## 6. Tiêu chí Nghiệm thu Trục 1 (Acceptance Criteria)

| ID | Tiêu chí | Phương pháp kiểm chứng |
|---|---|---|
| **AC-P1** | `api/` và `miniapp/` là 2 Git repository độc lập (có `.git` riêng, nhánh `main`), bảo toàn commit history S1–S5 tương ứng, `git status` sạch, không mất `.env` / `vendor` / `node_modules`. | `git -C api status && git -C api log --oneline -n 5` và tương tự cho `miniapp/` |
| **AC-P2** | Repo cha (`TestPart2`) không còn track `api/` và `miniapp/`, giữ nguyên `docs/`, `FEATURES.md`, `.mp4`, và không rewrite lịch sử `main`. | `git status && git ls-files api miniapp` (kết quả rỗng) |
| **AC-P3** | `api/` Docker build pass (cả `dev` và `production`), `docker compose config` hợp lệ; chạy được PHPUnit test suite & healthcheck `/up` trả `200` (nếu Docker Desktop đang bật trên máy). | `docker compose -f api/compose.yaml config`, `docker build --target production api`, `docker compose up` + `curl /up` + `php artisan test` |
| **AC-P4** | `miniapp/` Docker build pass (`production` non-root Nginx), `docker compose config` hợp lệ, `GET /healthz` trả `200` và phục vụ SPA; đồng thời `tsc`, `vitest`, `npm run build` pass. | `npm test && npx tsc --noEmit && npm run build`, `docker compose -f miniapp/compose.yaml config`, `docker build --target production miniapp` |
| **AC-P5** | Cả `api/.github/workflows/ci.yml` và `miniapp/.github/workflows/ci.yml` hợp lệ cú pháp YAML, bao phủ Lint/Type-check, Tests, Security Audit, Docker Build, không chứa secret hardcode. | YAML parse validation + code review từng job & env |

---

## 7. Lộ trình Trục 2 (Preview — Sẽ thiết kế chi tiết sau khi Trục 1 xong)

Ngay sau khi Trục 1 đạt `COMPLETED_VERIFIED`, Trục 2 sẽ nâng cấp phân hệ **Căn tin Số** ([FEATURES.md](../../../FEATURES.md), phương án `3A`) trong 2 sub-repo mới theo 3 lát cắt dọc (Vertical Slices):
- **Slice D1 — Hồ sơ Học sinh, Thẻ thông minh & Liên kết Ngân hàng + OTP 6 số:** Schema (`students`, `student_cards`, `bank_links`, `otp_verifications`), Bank/EPay Mock Adapter có interface tách bạch, API + UI Thẻ học sinh (`CARD-01..07`) & Liên kết ngân hàng + OTP (`BANK-01..16`, `OTP-01..08`).
- **Slice D2 — Hạn mức Chi tiêu Ngày + Lịch sử + Thanh toán & Chi tiết Đơn gắn Thẻ:** Schema (`spending_limits`, `spending_limit_logs`, mở rộng `orders`), API + UI thiết lập hạn mức + chọn nhanh `20k/30k/50k/100k` + lịch sử hạn mức (`LIMIT-01..13`), trừ hạn mức khi đặt món / mô phỏng quẹt thẻ POS, hoàn hạn mức khi hủy đơn, Chi tiết đơn đầy đủ theo `DETAIL-01..14`.
- **Slice D3 — Hủy Liên kết Ngân hàng + Quản trị Admin Blade + E2E Verification:** Luồng hủy liên kết + OTP (`UNLINK-01..07`), giao diện Admin quản lý Học sinh / Thẻ / Hạn mức / Giao dịch, kiểm thử E2E toàn bộ 4 luồng Căn tin Số.

---

## 8. Những Quyết định Nhỏ đã Đề xuất (Calls Made by Agent)

1. Chọn **PHP 8.4** cho Docker & CI của `api/` (khớp yêu cầu `>=8.4.1` trong `api/composer.lock`).
2. Dùng mô hình **2 container tách biệt (`app` php-fpm + `nginx` non-root)** cho `api/` thay vì gộp `supervisord`.
3. Dùng **`nginxinc/nginx-unprivileged:1.27-alpine`** (non-root port `8080`) cho cả `api` nginx proxy và `miniapp` static production image.
4. Tự động tạo cả 2 database `titkul_canteen_demo` và `titkul_canteen_test` qua `init-databases.sh` trong container PostgreSQL của `api/` để chạy `php artisan test` ngay trong Docker không cần cấu hình tay.
