# Ghi chú Phát hành — Smart Canteen `v0.1.0-mvp` (`docs/releases/release-v0.1.0-mvp.md`)

> **Lưu ý lịch sử:** Đây là ghi chú phát hành của mốc **MVP (`S1–S4`)**. Để xem phiên bản đầy đủ mới nhất sau khi hoàn thiện Trục 1 + Trục 2 (`D1–D3`) + Trục 3 (`P1–P4`) với **10/10 Module Căn tin Số**, xem [release-v1.0.0-digital-canteen.md](./release-v1.0.0-digital-canteen.md).

- **Phiên bản:** `v0.1.0-mvp`
- **Ngày phát hành nội bộ:** `2026-11-14`
- **Phạm vi hoàn thành:** Lát cắt S1, S2, S3 và S4 (100% tính năng P0 và P1 trên môi trường Local/Development).

---

## 1. Các Hạng mục Bàn giao trong Phiên bản `v0.1.0-mvp`

1. **Ứng dụng Khách hàng Zalo Mini App (`miniapp/`):**
   - Nhận diện thương hiệu Smart Canteen (`canteen-logo.jpg`, bảng màu xanh lá `#5DBE46` / `#2E7D32` chuẩn WCAG 2.1 AA).
   - Màn hình Thực đơn (`/`), Chi tiết món (`/products/:id`), Giỏ hàng (`/cart`), Đơn của tôi (`/orders`), Chi tiết & Theo dõi đơn (`/orders/:id`).
   - Tự động xác thực qua Zalo SDK (`getAccessToken()` → `POST /api/v1/auth/zalo`) hoặc fallback sang Demo Login (`POST /api/v1/auth/demo`) khi chạy trình duyệt local.
   - Polling tự động trạng thái đơn hàng mỗi 7 giây.

2. **Hệ thống Backend REST API & Cổng Quản trị Bếp (`api/`):**
   - REST API `/api/v1` đầy đủ cho Thực đơn, Xác thực Zalo/Demo, Đặt đơn Idempotent & Tính tiền Server, Lịch sử đơn cá nhân, và Quản trị đơn hàng / Thống kê.
   - Cổng Quản trị Laravel Blade (`/admin/login`, `/admin`, `/admin/orders`, `/admin/categories`, `/admin/products`) với phân quyền RBAC 3 cấp (`customer`, `staff`, `admin`), kiểm soát xung đột `version` (`409 Conflict`), và nhật ký `order_status_logs`.

3. **Cơ sở Dữ liệu & Kiểm thử Tự động:**
   - 10 migrations chuẩn hóa trên PostgreSQL 17.5 + 2 seeders idempotent (`MenuSeeder`, `StaffSeeder`).
   - **49 bài kiểm thử PHPUnit** (`344 assertions`) trên PostgreSQL + **11 bài kiểm thử Vitest** trên Frontend đạt **100% PASS**.

4. **Bộ Hồ sơ Tài liệu Kỹ thuật (`docs/`):**
   - Yêu cầu & Ma trận nghiệm thu: `docs/requirements.md`, `docs/requirements/acceptance-criteria-matrix.md`
   - Kiến trúc & Thiết kế: `docs/architecture/system-architecture.md`, `docs/design/ui-ux-brand-guidelines.md`, `DESIGN.md`
   - Hợp đồng API & Cơ sở dữ liệu: `docs/api-contract.md`, `docs/database.md`
   - Kiểm thử & Kịch bản trình diễn: `docs/test-cases.md`, `docs/testing/test-report.md`, `docs/demo-script.md`
   - Hướng dẫn triển khai S5: `docs/releases/s5-staging-zalo-deploy-runbook.md`
