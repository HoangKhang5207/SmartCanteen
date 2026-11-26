# Ghi chú Phát hành — Smart Canteen & Căn tin Số `v1.0.0-digital-canteen` (`docs/releases/release-v1.0.0-digital-canteen.md`)

- **Phiên bản:** `v1.0.0-digital-canteen` *(kế thừa `v0.1.0-mvp`)*
- **Ngày hoàn thiện nội bộ:** `2026-11-26`
- **Phạm vi hoàn thành:** Toàn bộ **MVP (`S1–S4`) + Hạ tầng (`Trục 1`) + Căn tin Số (`Trục 2 · D1–D3`) + Production-Readiness (`Trục 3 · P1–P4`) + Final Demo Polish** — phủ đầy **10/10 Module Căn tin Số** theo `FEATURES.md` §9.

---

## 1. Các Hạng mục Bàn giao Chính

1. **Hạ tầng Multi-Repo, Docker & CI (`Trục 1`):**
   - 2 Git repository triển khai độc lập (`api/`, `miniapp/`) với `Dockerfile` multi-stage non-root, `compose.yaml`, `compose.prod.yaml` và GitHub Actions CI riêng.
2. **Xác thực SĐT Zalo (Lazy Auth), RBAC 4 Vai trò & Bảo mật (`P1`, `P3`, `P4`):**
   - Xác thực SĐT Zalo phía server (`POST /api/v1/auth/zalo-phone`), Demo Phone local, Logout thu hồi Sanctum token.
   - Phân quyền 4 vai trò (`admin`, `staff`, `parent`/`customer`, `student`), mã hóa AES-256 cho STK/CCCD, OTP băm `bcrypt`, `DataMaskingService`, và `SecurityHeadersMiddleware`.
3. **Quản lý Học sinh, Thẻ Thông minh, Ngân hàng Sandbox, VNPay & Đối soát (`D1–D3`, `P2`, `P3`):**
   - CRUD Học sinh & cấp phát/khóa/thu hồi thẻ (`/admin/canteen/students`, `/admin/canteen/cards`).
   - Liên kết & hủy liên kết ngân hàng qua SMS OTP (`SmsGatewayInterface`: `log`/`esms`/`twilio`), hạn mức chi tiêu hằng ngày reset theo `Asia/Ho_Chi_Minh`.
   - `SandboxBankingGateway` (số dư & chế độ lỗi), sổ cái bất biến `payment_transactions`, Webhook `HMAC-SHA256` chống replay, `VNPay Sandbox` `HMAC-SHA512`, và Đối soát cuối ngày (`/admin/canteen/reconciliation`, CLI `canteen:reconcile`).
4. **Quầy POS Căn tin, Tồn kho Nguyên tử & Trung tâm Thông báo (`P3`, `P4`):**
   - Tồn kho `stock_quantity` trừ/hoàn nguyên tử theo thứ tự khóa dòng chuẩn PostgreSQL; Quầy POS `/admin/pos` + biên lai in `@media print`.
   - Module Thông báo `canteen_notifications` (7 sự kiện `DB::afterCommit()`), Push Adapter (`log`/`zalo_oa`/`zalo_zns`) fail-safe, Mini App `/notifications` + `BottomNav` 5 tab (`aria-live="polite"`), và Admin `/admin/canteen/notifications`.
5. **Bộ Demo Seeders & Hồ sơ Review (`Final Polish`):**
   - 4 Seeder idempotent (`MenuSeeder`, `StaffSeeder`, `StudentCardSeeder`, `DemoShowcaseSeeder`) sẵn sàng trình diễn toàn bộ 12 bước trong `docs/demo-script.md`.
   - Cẩm nang trình bày Review Code/Test: `docs/architecture/system-deep-dive-presentation-guide.md`.

---

## 2. Bằng chứng Cổng Chất lượng (Verified Quality Gates)

- **Backend (`api/`):** `vendor/bin/pint --test` **PASS (`200 files`)** · `php artisan test --compact` **PASS (`126 passed`, `1,387 assertions`)** trên PostgreSQL 17 · `migrate:fresh --seed` + `db:seed` lần 2 **PASS (`27 migrations`, `4 seeders`)**.
- **Frontend (`miniapp/`):** `npm test` **PASS (`11 files`, `44 tests`)** · `npx tsc --noEmit` **PASS (`0 errors`)** · `npm run build` **PASS (`608 modules`)**.
- **An toàn môi trường:** Chưa triển khai/push ra hạ tầng ngoài (`Deployment: NOT AUTHORIZED`); sẵn sàng chạy `docs/releases/s5-staging-zalo-deploy-runbook.md` ngay khi được cấp quyền.
