# Trục 1 — Tách 2 Sub-repo, Docker hóa & GitHub Actions CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` (inline execution selected for this session) and `superpowers:verification-before-completion`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tách `api/` và `miniapp/` thành 2 Git sub-repo độc lập có bảo toàn lịch sử commit, bổ sung bộ Container hóa (Docker multi-stage, non-root, healthcheck, dev & production-like compose) và GitHub Actions CI chuẩn Production-Ready riêng cho từng sub-repo.

**Architecture:** Dùng `git subtree split` từ repo cha để tạo nhánh lịch sử riêng cho `api/` và `miniapp/`, khởi tạo `.git` độc lập ngay tại `api/` và `miniapp/` (nhánh `main` giữ nguyên lịch sử S1–S5, công việc Docker/CI nằm trên nhánh `feat/docker-ci` ở từng sub-repo), gỡ tracking 2 thư mục khỏi index repo cha (`git rm -r --cached`). Backend `api/` dùng kiến trúc tách container (`app` PHP 8.4 FPM non-root + `nginx` unprivileged + `postgres` 17-alpine tự khởi tạo 2 DB demo/test). Frontend `miniapp/` dùng Node 24 multi-stage build sang `nginxinc/nginx-unprivileged:1.27-alpine` phục vụ static bundle `www/`.

**Tech Stack:** Git (`subtree split`), Docker / Docker Compose V2, PHP 8.4-FPM Alpine, Composer 2.8, PostgreSQL 17-Alpine, Nginx Unprivileged 1.27-Alpine, Node 24-Alpine, GitHub Actions.

**Spec:** [2026-11-25-subrepos-docker-ci-design.md](../specs/2026-11-25-subrepos-docker-ci-design.md) (APPROVED 2026-11-25)

## Global Constraints

- PHP floor: `>= 8.4.1` (khớp `api/composer.lock`) → dùng `php:8.4-fpm-alpine` và `php-version: '8.4'`.
- Node version: `24` (`node:24-alpine` và `node-version: '24'`), PostgreSQL `17` (`postgres:17-alpine`).
- Tất cả production container phải chạy **non-root** (`appuser:1000` cho PHP-FPM, `nginx:101` cho Nginx unprivileged) và có `HEALTHCHECK`.
- Không đưa secrets (`.env`, `APP_KEY`, `ZALO_APP_SECRET`, mật khẩu thật) vào image, compose, CI hay git.
- Bảo toàn tuyệt đối file untracked gốc (`FEATURES.md`, `2026-11-25 13-33-34.mp4`) và file local (`.env`, `vendor/`, `node_modules/`).
- Không `git push`, không `git merge` vào `main`, không viết lại lịch sử `main`.

## Review Focus

1. **Docker production immutability (`api`):** `compose.prod.yaml` không được phụ thuộc bind-mount host để phục vụ `/images/*` (phải đóng gói `public/` qua `docker/nginx/Dockerfile`).
2. **Composer binary leak:** Production image của `api/` không được chứa binary `composer`, `tests/`, `.git/` hoặc `.env`.
3. **Database test readiness:** Container `postgres` phải tự khởi tạo cả `titkul_canteen_demo` và `titkul_canteen_test` khi init lần đầu để `php artisan test` chạy được ngay mà không cần tạo DB thủ công.
4. **Windows/Linux line-endings (`CRLF` vs `LF`):** Các shell script (`api/docker/entrypoint.sh`, `api/docker/postgres/init-databases.sh`) phải có `LF` (và được xử lý an toàn trong Dockerfile) để không bị lỗi `exec: no such file or directory` trong Alpine Linux.
5. **CI `npm audit` & `composer audit` reliability:** Cấu hình security audit trên CI phải phản ánh đúng thực tế lockfile hiện tại, rõ ràng và không tạo false-green hay lỗi cú pháp YAML.

---

### Task 1: Tách `api/` và `miniapp/` thành 2 Git Sub-repo Độc lập

**Files:**
- Modify: `.gitignore` (root repo)
- Create: `api/.git` (standalone repo với full history S1–S5 trên `main`, nhánh làm việc `feat/docker-ci`)
- Create: `miniapp/.git` (standalone repo với full history S1–S5 trên `main`, nhánh làm việc `feat/docker-ci`)

**Interfaces:**
- Produces: 2 Git repo độc lập tại `api/` và `miniapp/` (cùng trên nhánh `feat/docker-ci` tách từ `main` sạch), repo cha trên nhánh `feat/subrepos-docker-ci` đã bỏ track `api/` và `miniapp/`.

- [ ] **Step 1:** Tại repo cha, tạo nhánh `feat/subrepos-docker-ci` từ `HEAD`, chạy `git subtree split --prefix=api -b split/api` và `git subtree split --prefix=miniapp -b split/miniapp`.
- [ ] **Step 2:** Khởi tạo Git độc lập tại `api/` (`git init -b main`, `git fetch .. split/api:refs/heads/main`, `git symbolic-ref HEAD refs/heads/main`, `git reset --mixed HEAD`, `git checkout -b feat/docker-ci`), kiểm chứng `git -C api status` sạch và `git -C api log --oneline -n 5` có lịch sử cũ, đồng thời `.env` và `vendor/` còn nguyên.
- [ ] **Step 3:** Khởi tạo Git độc lập tại `miniapp/` (quy trình tương tự), kiểm chứng `git -C miniapp status` sạch và `git -C miniapp log --oneline -n 5` có lịch sử cũ, đồng thời `.env` và `node_modules/` còn nguyên.
- [ ] **Step 4:** Tại repo cha: xóa nhánh tạm `split/api` và `split/miniapp`; chạy `git rm -r --cached api miniapp`; cập nhật `.gitignore` gốc để ignore `/api/` và `/miniapp/`; kiểm chứng `git ls-files api miniapp` rỗng và `FEATURES.md`, `.mp4` còn nguyên.
- [ ] **Step 5:** Commit trên nhánh `feat/subrepos-docker-ci` của repo cha (`chore(repo): detach api and miniapp into independent git sub-repos`).

---

### Task 2: Container hóa chuẩn Production-Ready cho Sub-repo `api/`

**Files:**
- Create: `api/.dockerignore`
- Create: `api/.env.docker.example`
- Create: `api/Dockerfile`
- Create: `api/compose.yaml`
- Create: `api/compose.prod.yaml`
- Create: `api/docker/entrypoint.sh`
- Create: `api/docker/php/php.ini`
- Create: `api/docker/php/opcache.ini`
- Create: `api/docker/php/www.conf`
- Create: `api/docker/nginx/Dockerfile`
- Create: `api/docker/nginx/default.conf`
- Create: `api/docker/postgres/init-databases.sh`

**Interfaces:**
- Consumes: `api/composer.json`, `api/composer.lock`, `api/bootstrap/app.php` (`health: '/up'`), `api/public/`
- Produces: Multi-stage Docker targets `dev` & `production`, Compose stack (`postgres` + `app` + `nginx`) expose cổng `${API_PORT:-8000}`

- [ ] **Step 1:** Tạo `api/.dockerignore`, `api/.env.docker.example` và các file cấu hình `api/docker/php/{php.ini,opcache.ini,www.conf}`, `api/docker/nginx/{default.conf,Dockerfile}`, `api/docker/postgres/init-databases.sh`, `api/docker/entrypoint.sh` (đảm bảo chuẩn LF cho script shell).
- [ ] **Step 2:** Tạo `api/Dockerfile` multi-stage (`composer-deps` → `runtime-base` → `dev` / `production`), non-root `appuser:1000`, healthcheck `/fpm-ping` qua `cgi-fcgi`, tự chuẩn hóa LF (`sed -i 's/\r$//'`) cho `entrypoint.sh`.
- [ ] **Step 3:** Tạo `api/compose.yaml` (dev/test) và `api/compose.prod.yaml` (production-like override không bind-mount).
- [ ] **Step 4:** Kiểm chứng `docker compose -f api/compose.yaml config` và `docker compose -f api/compose.yaml -f api/compose.prod.yaml config` (và build + chạy thử nếu Docker Engine hoạt động).
- [ ] **Step 5:** Commit trên nhánh `feat/docker-ci` của sub-repo `api/` (`feat(docker): add multi-stage production-ready Docker and Compose setup`).

---

### Task 3: Container hóa chuẩn Production-Ready cho Sub-repo `miniapp/`

**Files:**
- Create: `miniapp/.dockerignore`
- Create: `miniapp/Dockerfile`
- Create: `miniapp/compose.yaml`
- Create: `miniapp/compose.prod.yaml`
- Create: `miniapp/docker/nginx/default.conf`

**Interfaces:**
- Consumes: `miniapp/package.json`, `miniapp/package-lock.json`, `miniapp/vite.config.mts`, build output `www/`
- Produces: Multi-stage Docker targets `dev` (port `3000`, non-root `node`) & `production` (`nginxinc/nginx-unprivileged:1.27-alpine`, port `8080`, `/healthz`)

- [ ] **Step 1:** Tạo `miniapp/.dockerignore` và `miniapp/docker/nginx/default.conf` (SPA fallback, gzip, immutable asset cache, no-cache `index.html`, `/healthz`, security headers).
- [ ] **Step 2:** Tạo `miniapp/Dockerfile` multi-stage (`deps` → `dev` / `builder` → `production`) với `ARG VITE_API_BASE_URL`, chạy `npx tsc --noEmit && npm run build` ở `builder`, chạy non-root trên `production` kèm `HEALTHCHECK /healthz`.
- [ ] **Step 3:** Tạo `miniapp/compose.yaml` và `miniapp/compose.prod.yaml`.
- [ ] **Step 4:** Kiểm chứng `docker compose -f miniapp/compose.yaml config` và `docker compose -f miniapp/compose.prod.yaml config` (và `docker build` + smoke test nếu Docker Engine hoạt động) + chạy lại `npm test && npx tsc --noEmit && npm run build`.
- [ ] **Step 5:** Commit trên nhánh `feat/docker-ci` của sub-repo `miniapp/` (`feat(docker): add multi-stage production-ready Docker and Nginx setup`).

---

### Task 4: Thiết lập GitHub Actions CI Chuyên nghiệp cho Cả 2 Sub-repo

**Files:**
- Create: `api/.github/workflows/ci.yml`
- Create: `miniapp/.github/workflows/ci.yml`

**Interfaces:**
- Consumes: `api/Dockerfile`, `api/composer.lock`, `api/phpunit.xml`; `miniapp/Dockerfile`, `miniapp/package-lock.json`
- Produces: Independent CI pipelines cho mỗi sub-repo (Lint/Type-check, Automated Tests, Security Audit, Docker Buildx Verification)

- [ ] **Step 1:** Kiểm tra `npm audit` trên `miniapp/` để xác định ngưỡng audit thực tế không gây false failure.
- [ ] **Step 2:** Tạo `api/.github/workflows/ci.yml` với 4 job: `code-quality` (PHP 8.4, `composer validate --strict`, Pint), `tests` (PostgreSQL 17 service container + `php artisan test`), `security` (`composer audit --locked`), `docker` (Buildx target `dev` + `production` + smoke test `php artisan --version`).
- [ ] **Step 3:** Tạo `miniapp/.github/workflows/ci.yml` với 3 job: `quality-and-test` (Node 24, `npm ci`, `tsc --noEmit`, `npm test`, `npm run build`), `security` (`npm audit`), `docker` (Buildx target `production` + smoke test container `GET /healthz`).
- [ ] **Step 4:** Kiểm chứng cú pháp YAML hợp lệ cho cả 2 workflow file và rà soát không lộ secret.
- [ ] **Step 5:** Commit trên từng sub-repo (`ci(github): add GitHub Actions CI pipeline for lint, tests, audit and Docker build`).

---

### Task 5: Cập nhật Tài liệu Bàn giao, Code Review & Kiểm chứng Cuối cùng

**Files:**
- Modify: `api/README.md` (hướng dẫn Docker, CI, gắn remote GitHub riêng)
- Modify: `miniapp/README.md` (hướng dẫn Docker, CI, gắn remote GitHub riêng)
- Modify: `README.md` (cập nhật kiến trúc 2 sub-repo độc lập + Workspace Hub)
- Modify: `docs/PROJECT_STATE.md` (ghi nhận checkpoint Trục 1)

- [ ] **Step 1:** Cập nhật `api/README.md` và `miniapp/README.md`, commit vào từng sub-repo tương ứng.
- [ ] **Step 2:** Cập nhật `README.md` và `docs/PROJECT_STATE.md` ở repo cha, commit vào nhánh `feat/subrepos-docker-ci`.
- [ ] **Step 3:** Chạy toàn bộ Verification Quality Gates trên cả 3 repo (`api/`, `miniapp/`, root) bao gồm Git status/log checks, Docker compose config & Docker builds (nếu daemon sẵn sàng), YAML validation, Mini App tests/typecheck/build.
- [ ] **Step 4:** Thực hiện Fresh-context Code Review qua subagent (`requesting-code-review`) trên diff của cả 3 repo theo **Review Focus**; sửa toàn bộ vấn đề Critical/Important nếu có và kiểm chứng lại.
- [ ] **Step 5:** Trả báo cáo hoàn tất chuẩn Mục 17 kèm bằng chứng thực tế.
