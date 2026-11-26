# Lộ trình Triển khai & Các Lát cắt Dọc — Smart Canteen & Căn tin Số (`docs/plans/roadmap-and-slices.md`)

Tài liệu này tổng hợp đầy đủ chiến lược triển khai theo **Vertical Slices** xuyên suốt 4 giai đoạn lớn của dự án: **MVP (`S1–S4`) → Hạ tầng (`Trục 1`) → Căn tin Số (`Trục 2 · D1–D3`) → Production-Readiness (`Trục 3 · P1–P4`) → S5 Deployment**.

---

## 1. Giai đoạn 1 — Nền tảng Smart Canteen MVP (`S1 → S4`)

| Lát cắt | Phạm vi chức năng chính | Kế hoạch chi tiết | Trạng thái |
|---|---|---|---|
| **S1 — Monorepo & Menu Slice** | Khởi tạo Laravel 13 + React 18/ZaUI + PostgreSQL 17, bảng `categories`/`products`, `MenuSeeder`, API thực đơn công khai, CORS, màn hình Thực đơn & Chi tiết | `docs/superpowers/plans/2026-11-14-s1-menu-slice.md` | **MERGED (`6f44c9c`)** |
| **S2 — Brand, Auth, Cart & Checkout** | Nhận diện thương hiệu WCAG AA, Zalo OAuth (`appsecret_proof`) + Demo Login, giỏ hàng Jotai, đặt đơn `DB::transaction` + `Idempotency-Key` + Server Pricing | `docs/superpowers/plans/2026-11-14-s2-checkout-slice.md` | **MERGED (`d45ca09`)** |
| **S3 — Admin Portal, RBAC & Dispatch** | Bảng `order_status_logs`, State Machine + Optimistic Locking (`version`), `StatisticsService`, Cổng Blade `/admin/*` & CRUD Thực đơn | `docs/superpowers/plans/2026-11-14-s3-admin-slice.md` | **MERGED (`51601fc`)** |
| **S4 — Polish, E2E 8 Bước & Tài liệu** | Tinh chỉnh validation & tách quyền `role:admin`, `EndToEndDemoWorkflowTest`, bộ tài liệu kỹ thuật nền tảng | `docs/superpowers/plans/2026-11-14-s4-docs-polish-slice.md` | **MERGED** |

---

## 2. Giai đoạn 2 — Hạ tầng Multi-Repo, Docker & CI (`Trục 1`)

| Lát cắt | Phạm vi chức năng chính | Đặc tả / Kế hoạch | Trạng thái |
|---|---|---|---|
| **Trục 1 — Sub-repos, Docker & CI** | Tách `api/` và `miniapp/` thành 2 Git repository độc lập (`git subtree split`), bổ sung `Dockerfile` multi-stage non-root, `compose.yaml`/`compose.prod.yaml` (PHP 8.4 + Nginx unprivileged + PostgreSQL 17) & GitHub Actions CI độc lập | `docs/superpowers/specs/2026-11-25-subrepos-docker-ci-design.md` | **MERGED** |

---

## 3. Giai đoạn 3 — Phân hệ Căn tin Số Cốt lõi (`Trục 2 · D1–D3`)

| Lát cắt | Phạm vi chức năng chính | Đặc tả / Kế hoạch | Trạng thái |
|---|---|---|---|
| **Slice D1 — Thẻ Học sinh & Liên kết Ngân hàng + OTP** | Bảng `students`, `student_cards`, `bank_links` (AES-256 + Partial Unique Index), `otp_verifications` (bcrypt), `BankingGatewayInterface`, `DataMaskingService`, Mini App `/canteen`, `/canteen/link-bank`, `/canteen/verify-otp` | `docs/superpowers/specs/2026-11-25-d1-student-card-bank-link-design.md` | **MERGED (`4c76e83` / `fa9418a` / `01c8db3`)** |
| **Slice D2 — Hạn mức Ngày, Thanh toán Trừ Hạn mức & POS** | `spending_limits` (lazy daily reset `Asia/Ho_Chi_Minh`), `spending_limit_logs`, mở rộng `orders` hỗ trợ thẻ & `#POS-...`, hoàn hạn mức khi hủy, `SpendingLimitSheet`, `LimitHistoryPage`, `OrderDetailPage` 14 trường | `docs/superpowers/specs/2026-11-25-d2-spending-limit-orders-design.md` | **MERGED (`941014c` / `89d76d1` / `14a433b`)** |
| **Slice D3 — Hủy Liên kết OTP, Admin Căn tin & E2E 4 Luồng** | 3 API `/unlink/{initiate,verify-otp,resend-otp}`, Admin Blade `/admin/canteen/cards`, `UnlinkConfirmDialog`, `BankUnlinkOtpPage`, 7 Deferred Minors, `DigitalCanteenEndToEndFlowsTest` + Minor Polish | `docs/superpowers/specs/2026-11-25-d3-unlink-admin-e2e-design.md` | **MERGED (`fc93101` / `c9caf98` / `b73ce24`)** |

---

## 4. Giai đoạn 4 — Production-Readiness Toàn Diện (`Trục 3 · P1–P4` + Final Demo Polish)

| Lát cắt | Phạm vi chức năng chính | Đặc tả / Kế hoạch | Trạng thái |
|---|---|---|---|
| **Slice P1 — Zalo Phone Auth (Lazy Auth) & SMS OTP** | `POST /api/v1/auth/{zalo-phone,demo-phone,logout}`, `PhoneNormalizer`, `SmsGatewayInterface` (`log`/`esms`/`twilio`) rollback sạch khi SMS lỗi, Mini App Lazy Auth + `ZaloPhoneLoginModal` | `docs/superpowers/specs/2026-11-25-p1-zalo-phone-auth-sms-otp-design.md` | **MERGED (`8bd8340` / `63e1d02` / `c7889be`)** |
| **Slice P2 — Sandbox Banking, VNPay, Webhook & Đối soát** | `SandboxBankingGateway`, sổ cái `payment_transactions` (ghi bền vững `FAILED`/`TIMEOUT`), Webhook `HMAC-SHA256` chống replay, `VNPay Sandbox` `HMAC-SHA512`, `ReconciliationService` + CLI `canteen:reconcile` + `/admin/canteen/reconciliation` | `docs/superpowers/specs/2026-11-25-p2-sandbox-banking-vnpay-reconciliation-design.md` | **MERGED (`120e480` / `3ab9af5` / `7a2d4e5`)** |
| **Slice P3 — POS Staff Terminal, Tồn kho, CRUD Học sinh & RBAC 4 Vai trò** | Trừ/hoàn `stock_quantity` nguyên tử, Quầy POS `/admin/pos` + biên lai in, CRUD Học sinh `/admin/canteen/students`, RBAC 4 vai trò (`admin`/`staff`/`parent`/`student`), chế độ Học sinh chỉ-xem trên Mini App | `docs/superpowers/specs/2026-11-25-p3-pos-inventory-students-rbac-design.md` | **MERGED (`42e1146` / `f2a5625` / `3044130`)** |
| **Slice P4 — Notification Module & Security/E2E Audit** | `canteen_notifications`, 7 sự kiện `DB::afterCommit()`, Push `log`/`zalo_oa`/`zalo_zns` fail-safe, Mini App `/notifications` + 5-tab `BottomNav`, Admin `/admin/canteen/notifications`, `SecurityHeadersMiddleware`, `ProductionReadinessEndToEndTest`, `truc3-security-e2e-audit.md` | `docs/superpowers/specs/2026-11-26-p4-notifications-security-e2e-design.md` | **MERGED (`8418b46` / `de322f3` / `dd6cac2`)** |
| **Final Demo Polish & Review Kit** | Hoàn tất 2 Minor cuối của P4 (index lọc Admin & `aria-live="polite"`), `DemoShowcaseSeeder` idempotent cho demo 10/10 module, đồng bộ toàn bộ `docs/`, và cẩm nang Deep-Dive Review cuộc họp kỹ thuật | Nhánh `chore/p4-polish-demo-docs` | **COMPLETED_VERIFIED** |

---

## 5. Giai đoạn 5 — Triển khai Staging HTTPS & Zalo Testing (`S5`)

- **Tài liệu:** `docs/releases/s5-staging-zalo-deploy-runbook.md`
- **Trạng thái:** `READY TO EXECUTE` — chờ lệnh phê duyệt triển khai chính thức (`Deployment: NOT AUTHORIZED`).
