# Lộ trình Triển khai & Các Lát cắt Dọc — Smart Canteen (`docs/plans/roadmap-and-slices.md`)

Tài liệu này tổng hợp chiến lược phân rã dự án **Smart Canteen** theo phương pháp **Vertical Slices (Lát cắt chức năng hoàn chỉnh theo chiều dọc)** kết hợp với lộ trình 7 ngày trong `BaiTestPart2.md`.

---

## 1. Bảng Tổng hợp Các Lát cắt Dọc (Vertical Slices S1 → S5)

| Lát cắt | Phạm vi chức năng (Backend + Frontend + Database + Tests) | Kế hoạch chi tiết | Trạng thái |
|---|---|---|---|
| **S1 — Monorepo & Menu Slice** | Khởi tạo Git monorepo (`api/` Laravel 13 + `miniapp/` React 18/ZaUI), PostgreSQL 17.5 (`titkul_canteen_demo`, `titkul_canteen_test`), bảng `categories` & `products`, `MenuSeeder` (4 danh mục, 12 món), API thực đơn công khai, cấu hình CORS, màn hình Thực đơn & Chi tiết món trên Mini App | `docs/superpowers/plans/2026-11-14-s1-menu-slice.md` | **COMPLETED & MERGED (`6f44c9c`)** |
| **S2 — Brand Theme, Auth, Cart & Checkout Slice** | Tích hợp Logo Smart Canteen (`canteen_logo.jpg`) & bảng màu xanh lá chuẩn WCAG AA (`#5DBE46` / `#2E7D32`), xác thực Zalo Open API (`appsecret_proof`) + Demo Login local, giỏ hàng trong bộ nhớ (`Jotai`), đặt đơn trong `DB::transaction` + `lockForUpdate()` kèm `Idempotency-Key`, tính tiền phía server, màn hình Giỏ hàng / Đơn của tôi / Chi tiết đơn | `docs/superpowers/plans/2026-11-14-s2-checkout-slice.md` | **COMPLETED & MERGED (`d45ca09`)** |
| **S3 — Admin/Staff Portal, RBAC, Dispatch & Statistics Slice** | Bảng `order_status_logs`, middleware `role:staff,admin`, `StaffSeeder`, chuyển trạng thái đơn theo State Machine + Optimistic Locking (`version`), thống kê doanh thu hôm nay (`StatisticsService`), Cổng Quản trị Laravel Blade (`/admin/*`) kèm CRUD Danh mục & Món ăn | `docs/superpowers/plans/2026-11-14-s3-admin-slice.md` | **COMPLETED & MERGED (`51601fc`)** |
| **S4 — RBAC/Validation Polish, E2E Verification & Full Documentation** | Xử lý triệt để 3 điểm tinh chỉnh từ S3 (`status`/`limit` validation 422, tự động dọn session `customer` tại `/admin/login`, tách quyền `role:admin` cho thao tác ghi/xóa thực đơn), bài test tích hợp tự động 8 bước `EndToEndDemoWorkflowTest`, và hoàn thiện toàn bộ hồ sơ tài liệu kỹ thuật | `docs/superpowers/plans/2026-11-14-s4-docs-polish-slice.md` | **COMPLETED (`feature/s4-docs-polish`)** |
| **S5 — Staging HTTPS & Zalo Mini App Testing Deployment** | Đưa Laravel API lên HTTPS public (Cloudflare Tunnel / VPS), cấu hình `VITE_API_BASE_URL` HTTPS, deploy bản Zalo Mini App **Development/Testing**, cấp quyền tài khoản Giám đốc và xuất QR Code / Deep Link | `docs/releases/s5-staging-zalo-deploy-runbook.md` | **READY TO EXECUTE (Chờ cấp quyền Deployment)** |

---

## 2. Ánh xạ với Lộ trình 7 Ngày (`BaiTestPart2.md` Mục 8)

- **Ngày 1 (Khảo sát nghiệp vụ & khởi tạo Zalo Mini App):** Hoàn thành trong S1.
- **Ngày 2 (Laravel, PostgreSQL & API Thực đơn):** Hoàn thành trong S1.
- **Ngày 3 (Shopping Cart & Transactional Checkout):** Hoàn thành trong S2.
- **Ngày 4 (Admin Portal & Order Workflow + Polling):** Hoàn thành trong S3.
- **Ngày 5 (Authentication, RBAC, Idempotency & Automated Tests):** Hoàn thành trong S2, S3 và S4 (49 tests / 344 assertions trên PostgreSQL + 11 tests Vitest).
- **Ngày 6 (Deploy HTTPS & Zalo Testing QR):** Đã chuẩn bị sẵn toàn bộ cấu hình và quy trình thực thi từng lệnh trong `docs/releases/s5-staging-zalo-deploy-runbook.md` (chờ lệnh phê duyệt triển khai).
- **Ngày 7 (Ổn định, Tài liệu hóa & Kịch bản Demo cho Giám đốc):** Hoàn thành trong S4 (`docs/requirements.md`, `docs/api-contract.md`, `docs/database.md`, `docs/test-cases.md`, `docs/demo-script.md`).
