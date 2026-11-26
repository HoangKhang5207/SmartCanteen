# Ma trận Truy vết Tiêu chí Nghiệm thu — Smart Canteen & Căn tin Số (`docs/requirements/acceptance-criteria-matrix.md`)

Tài liệu này truy vết đầy đủ toàn bộ tiêu chí nghiệm thu từ giai đoạn **MVP (`AC-01..AC-16`)**, **Hạ tầng Docker/CI (`Trục 1`)**, **Căn tin Số (`Trục 2 · D1–D3`)** tới **Production-Readiness (`Trục 3 · P1–P4`)** và **10/10 Module (`FEATURES.md` §9)**.

---

## 1. Nhóm Tiêu chí Cốt lõi MVP (`AC-01` .. `AC-16`)

| Mã AC | Mô tả Tiêu chí Nghiệm thu | Giai đoạn | Thành phần hiện thực hóa chính | Bài kiểm thử tự động kiểm chứng | Trạng thái |
|---|---|---|---|---|---|
| **AC-01** | CSDL có ít nhất 10 sản phẩm giả lập và 3 danh mục | S1 | `MenuSeeder` (6 danh mục, 30 món có tồn kho & ảnh chuẩn) | `MenuSeederTest`, `EndToEndDemoWorkflowTest` | **PASS** |
| **AC-02** | Xem danh mục, lọc/tìm kiếm món tiếng Việt, phân trang và xem chi tiết món trên Mini App | S1 | `CategoryController`, `ProductController`, `menu.tsx`, `product-detail.tsx` | `MenuApiTest` (8 tests), `pagination-bar.test.ts` | **PASS** |
| **AC-03** | Thêm/xóa món và đổi số lượng (`1..20`, không vượt `stock_quantity`) trong giỏ hàng | S2 / P3 | `cart-logic.ts`, `cart/store.ts`, `cart.tsx` | `cart-logic.test.ts` | **PASS** |
| **AC-04** | Tạo đơn hàng trong `DB::transaction`; tính giá 100% ở backend (bỏ qua giá client) | S2 | `OrderService::createOrder`, `StoreOrderRequest` | `OrderApiTest`, `EndToEndDemoWorkflowTest` | **PASS** |
| **AC-05** | Từ chối đơn có món ngừng bán (`is_available = false`), hết kho (`stock_quantity = 0`), danh mục ẩn, hoặc SL ngoài `1..20` (`422`) | S2 / P3 | `OrderService::createOrder`, `InsufficientStockException` | `OrderApiTest`, `ProductInventoryTest` | **PASS** |
| **AC-06** | Chống gửi trùng đơn bằng `Idempotency-Key` (`201`/`200` cùng payload, `409` khác payload) | S2 | `OrderService`, `UNIQUE (user_id, idempotency_key)` | `OrderApiTest`, `EndToEndDemoWorkflowTest`, `StaffPosTerminalTest` | **PASS** |
| **AC-07** | Admin/Staff xem danh sách đơn, lọc trạng thái/mã đơn & thống kê hôm nay | S3 | `Admin\OrderController`, `DashboardController`, `StatisticsService` | `AdminOrderApiTest`, `AdminStatisticsApiTest`, `AdminWebPortalTest` | **PASS** |
| **AC-08** | Chuyển trạng thái theo State Machine, kiểm soát `version` (`409`), ghi `order_status_logs` | S3 / S4 | `OrderStatus::allowedTransitions`, `OrderService::transitionStatus` | `AdminOrderApiTest`, `OrderSchemaTest` | **PASS** |
| **AC-09** | Mini App tự động cập nhật trạng thái đơn (polling 7s/10s) và hiển thị lịch sử đơn | S2 / S4 | `order-detail.tsx`, `orders.tsx` | `EndToEndDemoWorkflowTest` | **PASS** |
| **AC-10** | Cô lập dữ liệu giữa các người dùng (`404` IDOR) & RBAC chặt (`403`) | S2–P4 | `OrderPolicy`, `EnsureUserHasRole`, `StudentCardService`, `NotificationController` | `OrderApiTest`, `FourRoleRbacTest`, `NotificationModuleTest` | **PASS** |
| **AC-11** | Đổi giá món sau khi đặt không ảnh hưởng đơn giá/tổng tiền đơn hàng cũ | S2 / S3 | `order_items.unit_price`, `order_items.subtotal`, `orders.total_amount` | `OrderApiTest`, `AdminWebPortalTest`, `EndToEndDemoWorkflowTest` | **PASS** |
| **AC-12** | Không chứa thông tin, khóa hay dữ liệu production của Căn Tin Số | Toàn dự án | `.gitignore`, `.env.example`, Seeders, Audit report | Kiểm toán mã nguồn & `truc3-security-e2e-audit.md` | **PASS** |
| **AC-13** | Đầy đủ `README.md`, tài liệu API, DB, Yêu cầu, Kiến trúc, Test Cases & `demo-script.md` | S4 / Polish | Toàn bộ thư mục `docs/` | Rà soát & cập nhật đầy đủ hậu Trục 3 | **PASS** |
| **AC-14** | Mini App mở được trên Zalo bằng QR/Deep Link bản Testing | S5 (Deploy) | `miniapp/app-config.json`, `docs/releases/s5-staging-zalo-deploy-runbook.md` | Chờ lệnh cấp quyền triển khai ngoài (`Deployment: NOT AUTHORIZED`) | **BLOCKED (By Policy)** |
| **AC-15** | Cấp quyền tài khoản kiểm thử Giám đốc trên Zalo Developer Center | S5 (Deploy) | `docs/releases/s5-staging-zalo-deploy-runbook.md` | Chờ lệnh cấp quyền triển khai ngoài | **BLOCKED (By Policy)** |
| **AC-16** | Backend HTTPS công khai vượt qua kiểm tra CORS từ `https://h5.zdn.vn` | S5 (Deploy) | `api/config/cors.php`, `CorsTest` | `CorsTest` (Local **PASS**; Public HTTPS chờ lệnh S5) | **BLOCKED (By Policy)** |

---

## 2. Nhóm Tiêu chí Hạ tầng & Phân hệ Căn tin Số (`Trục 1 + Trục 2 · D1–D3`)

| Mã AC | Mô tả Tiêu chí Nghiệm thu | Lát cắt | Bài kiểm thử tự động kiểm chứng | Trạng thái |
|---|---|---|---|---|
| **AC-T1** | Tách 2 Git sub-repo độc lập (`api/`, `miniapp/`), Docker multi-stage non-root & GitHub Actions CI | Trục 1 | `docker compose config`, build non-root image, CI YAML, PHPUnit & Vitest trong Docker | **PASS** |
| **AC-D1** | Hồ sơ học sinh, Thẻ thông minh (`CARD-01..07`), Liên kết ngân hàng AES-256 + OTP 6 số (`BANK-01..15`, `OTP-01..07`), tối đa 1 liên kết `ACTIVE`/thẻ | Slice D1 | `DigitalCanteenBankLinkTest`, `DataMaskingServiceTest`, `validation.test.ts` | **PASS** |
| **AC-D2** | Hạn mức chi tiêu ngày (`LIMIT-01..13`), lazy reset `Asia/Ho_Chi_Minh`, thanh toán trừ hạn mức, POS `#POS-...`, hoàn hạn mức khi hủy, chi tiết 14 trường (`DETAIL-01..14`) | Slice D2 | `SpendingLimitAndOrderPaymentTest`, `spending-limit.test.ts` | **PASS** |
| **AC-D3** | Hủy liên kết ngân hàng qua OTP (`UNLINK-01..07`, `OTP-08`), Quản trị Blade Căn tin (`/admin/canteen/cards`), 4 luồng E2E Căn tin số | Slice D3 | `BankUnlinkOtpTest`, `AdminCanteenManagementTest`, `DigitalCanteenEndToEndFlowsTest` | **PASS** |

---

## 3. Nhóm Tiêu chí Production-Readiness (`Trục 3 · P1–P4`)

| Mã AC | Mô tả Tiêu chí Nghiệm thu | Lát cắt | Bài kiểm thử tự động kiểm chứng | Trạng thái |
|---|---|---|---|---|
| **AC-P1** | Xác thực SĐT Zalo (`POST /api/v1/auth/zalo-phone`), Lazy Auth trên Mini App, Demo Phone local, `SmsGatewayInterface` (`log`/`esms`/`twilio`) rollback sạch khi SMS lỗi | Slice P1 | `ZaloPhoneAuthTest`, `SmsOtpGatewayTest`, `phone.test.ts` | **PASS** |
| **AC-P2** | `SandboxBankingGateway` (số dư, `INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`, hoàn tiền), sổ cái `payment_transactions`, Webhook `HMAC-SHA256` chống replay, `VNPay Sandbox` `HMAC-SHA512`, Đối soát `/admin/canteen/reconciliation` & `canteen:reconcile` | Slice P2 | `SandboxBankingPaymentTest`, `BankingWebhookTest`, `VnpaySandboxTest`, `AdminReconciliationTest`, `payment.test.ts` | **PASS** |
| **AC-P3** | Tồn kho nguyên tử `stock_quantity` (trừ/hoàn + chống oversell & chống ghi đè kho), Quầy POS Staff `/admin/pos` + biên lai in, CRUD Học sinh `/admin/canteen/students`, RBAC 4 vai trò (`admin`/`staff`/`parent`/`student`) | Slice P3 | `ProductInventoryTest`, `StaffPosTerminalTest`, `AdminStudentCrudTest`, `FourRoleRbacTest` | **PASS** |
| **AC-P4** | Module Thông báo (`canteen_notifications` + 7 sự kiện `afterCommit` + `log`/`zalo_oa`/`zalo_zns` fail-safe + Mini App `/notifications` + 5-tab `BottomNav` badge `aria-live="polite"` + Admin `/admin/canteen/notifications`), `SecurityHeadersMiddleware`, Kiểm toán bảo mật & E2E liên hoàn P1→P4, Demo Showcase Seeder idempotent | Slice P4 + Polish | `NotificationModuleTest`, `ProductionReadinessEndToEndTest`, `DemoShowcaseSeederTest`, `helpers.test.ts` | **PASS** |
