# Đặc tả Thiết kế Chi tiết — Trục 3 · Slice P4: Notification Module (In-App + Push Zalo OA/ZNS), Security & Full-System E2E Audit

- **Ngày lập:** 2026-11-26
- **Mã tác vụ:** `CANTEEN-P4-NOTIFICATIONS-SECURITY-E2E`
- **Trạng thái:** `APPROVED (2026-11-26)`
- **Nền tảng cơ sở (`main`):** Sau khi đã merge cục bộ Trục 3 · Slice P3 (`root @ 42e1146`, `api @ f2a5625`, `miniapp @ 3044130`).
- **Nguồn đối chiếu:** [2026-11-25-truc3-production-readiness-roadmap.md](file:///d:/Coding/TitKul_ProjectTest/TestPart2/docs/superpowers/specs/2026-11-25-truc3-production-readiness-roadmap.md) (`APPROVED 2026-11-25`), `FEATURES.md` (Mục 7 #14, #16 & Mục 9 Module #10), `AGENTS.md`.
- **Phạm vi bao gồm:** Notification Module hoàn chỉnh (In-App + Push Zalo OA/ZNS Strategy), Security Hardening + Full-System E2E Audit toàn bộ Trục 3 (P1→P4), và 3 điểm Deferred Minors từ Slice P3. **Ngoài phạm vi:** Push lên remote Git, triển khai production/staging hoặc gọi mạng thật trong CI.

---

## 1. Mục tiêu & Kiến trúc Tổng thể

Slice P4 là lát cắt cuối cùng khép kín **10/10 module Căn tin số (`FEATURES.md` §9)** và **16/16 nghiệp vụ Production-Ready (`FEATURES.md` §7)**:

1. **Notification Module chuẩn Production (In-App + Push Channel Strategy):**
   - Lưu lịch sử thông báo bền vững tại bảng `canteen_notifications`, phát sự kiện tự động sau khi giao dịch tài chính commit thành công (hoặc sau khi thanh toán thất bại bên ngoài transaction rollback) — **tuyệt đối không để lỗi kênh push làm ảnh hưởng/rollback giao dịch thanh toán hoặc đặt món**.
   - Tầng trừu tượng `PushNotificationChannelInterface` với 3 driver (`NOTIFICATION_PUSH_DRIVER=log|zalo_oa|zalo_zns`): `LogPushNotificationChannel` (mặc định local/testing/CI, che PII), `ZaloOaPushChannel` (Zalo OA Open API v3), và `ZaloZnsPushChannel` (Zalo ZNS Template API hỗ trợ sandbox mode).
2. **Giao diện Trung tâm Thông báo trên Mini App (`/notifications`) & Nhật ký Admin Blade (`/admin/canteen/notifications`):**
   - Mini App: thêm tab **Thông báo** kèm badge số chưa đọc trên `BottomNav`, trang `/notifications` (Lazy Auth) hỗ trợ lọc `Tất cả` / `Chưa đọc`, đánh dấu đã đọc từng mục hoặc toàn bộ, và điều hướng an toàn tới đơn hàng/thẻ tương ứng (`action_url` chỉ chấp nhận đường dẫn nội bộ `/...`).
   - Admin Blade: trang chỉ-đọc `/admin/canteen/notifications` (`role:staff,admin`) theo dõi trạng thái gửi In-App & Push (`push_channel`, `push_status`).
3. **Security Hardening & Full-System E2E Audit (P1 → P2 → P3 → P4):**
   - Thêm `SecurityHeadersMiddleware` (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`).
   - Bộ kiểm thử E2E toàn hệ thống (`ProductionReadinessEndToEndTest.php`) kiểm chứng liên hoàn từ Zalo Phone Auth → Liên kết ngân hàng OTP SMS → Đặt hạn mức → Staff POS trừ kho/hạn mức/ngân hàng → Cảnh báo hạn mức thấp → Chặn Học sinh 403 → Lỗi thiếu số dư ngân hàng rollback kho → Hủy đơn hoàn kho/tiền → VNPay Sandbox IPN + Đối soát → Notification Center & IDOR/PII Audit.
   - Hồ sơ báo cáo kiểm toán bảo mật & nghiệm thu tổng thể [truc3-security-e2e-audit.md](file:///d:/Coding/TitKul_ProjectTest/TestPart2/docs/testing/truc3-security-e2e-audit.md).
4. **Dọn dẹp trọn vẹn 3 Deferred Minors từ Slice P3:**
   - Roving `tabIndex` cho `CartPage` radiogroup & tinh chỉnh câu chữ phù hợp vai trò `student` ở `LimitHistoryPage`/`CartPage`/`CanteenHubPage`.
   - Snackbar cảnh báo khi bấm `+ Thêm` tại `MenuPage` đã chạm trần `stock_quantity` & prepend động đơn POS mới vào sidebar `"Đơn POS gần đây hôm nay"` trên `/admin/pos`.
   - Chống ghi đè tồn kho đồng thời khi Admin chỉnh sửa metadata món ăn ở `ProductManagementController::update()`.

---

## 2. Phương án Thiết kế Đã Cân nhắc (Approaches)

| Quyết định | Phương án chọn | Phương án loại bỏ & Lý do |
|---|---|---|
| **Bảng lưu thông báo** | Tạo bảng chuyên biệt **`canteen_notifications`** (PK `bigint`, cột rõ ràng `type`, `severity`, `title`, `body`, `meta jsonb`, `action_url`, `push_channel`, `push_status`, `push_reference`, `read_at`). | *Dùng bảng `notifications` đa hình mặc định của Laravel (UUID + JSON `data`):* Khó truy vấn lọc theo `type`/`push_status` và không có cột theo dõi trạng thái gửi Zalo OA/ZNS tường minh. |
| **Kỹ thuật phát thông báo (Isolation)** | Dispatch qua `NotificationService` sử dụng **`DB::afterCommit()`** cho các luồng thành công và gọi trực tiếp ngoài `DB::transaction()` cho lỗi thanh toán (`FAILED`/`TIMEOUT`), bọc `try/catch` toàn diện. | *Gọi HTTP Zalo OA/ZNS ngay trong `DB::transaction()`:* Giữ khóa PostgreSQL hàng giây khi mạng chậm và làm hỏng đơn hàng nếu Zalo OA tạm thời 503. |
| **Vị trí vào Thông báo trên Mini App** | Nâng cấp `BottomNav` thành **5 mục gọn** (`Thực đơn`, `Căn tin`, `Giỏ hàng`, `Đơn hàng`, `Thông báo`) kèm badge số chưa đọc (ẩn khi chưa đăng nhập) + route `/notifications`. | *Chỉ đặt biểu tượng chuông ở 1 trang duy nhất:* Người dùng ở trang Đơn hàng hoặc Thực đơn không thấy ngay cảnh báo giao dịch/hạn mức mới. |

---

## 3. Thiết kế Cơ sở Dữ liệu (100% Additive Migration)

### 3.1. `2026_11_26_600001_create_canteen_notifications_table.php`
Bảng `canteen_notifications`:
- `id` (`bigIncrements`)
- `user_id` (`foreignId` → `users.id` `cascadeOnDelete`)
- `type` (`string(40)`): `PURCHASE_SUCCESS`, `LOW_SPENDING_LIMIT`, `PAYMENT_FAILED`, `ORDER_REFUNDED`, `ORDER_STATUS_UPDATED`, `BANK_LINK_CHANGED`, `SPENDING_LIMIT_UPDATED`
- `severity` (`string(16)`, default `'INFO'`): `INFO`, `SUCCESS`, `WARNING`, `ERROR`
- `title` (`string(160)`)
- `body` (`string(500)`)
- `action_url` (`string(160)`, nullable — chỉ chấp nhận internal route dạng `/orders/123`, `/canteen`, `/canteen/limit-history`)
- `meta` (`jsonb`, nullable — chỉ chứa ID/mã đơn/số tiền/tên che mờ; **tuyệt đối không lưu STK/CCCD/OTP/SĐT thô**)
- `push_channel` (`string(24)`, default `'log'`): `log`, `zalo_oa`, `zalo_zns`, `in_app`
- `push_status` (`string(24)`, default `'SKIPPED'`): `SKIPPED`, `SIMULATED`, `SENT`, `FAILED`
- `push_reference` (`string(120)`, nullable)
- `read_at` (`timestamp`, nullable)
- `created_at`, `updated_at`
- Indexes:
  - `(user_id, read_at, id)` — tối ưu đếm `unread_count` và lọc chưa đọc.
  - `(user_id, type, created_at)` — tối ưu lọc theo loại thông báo.

---

## 4. Hợp đồng Sự kiện, Push Drivers & API

### 4.1. Danh sách Sự kiện Phát Thông báo Tự động

| Sự kiện (`type`) | Điểm kích hoạt | Người nhận | Điều kiện đặc biệt |
|---|---|---|---|
| `PURCHASE_SUCCESS` | Sau commit `OrderService::createOrder` (thẻ), `createPosPurchase`, `createStaffPosOrder` (thẻ), hoặc `VnpayGatewayService::handleIpn` (thành công) | Phụ huynh (`student.user_id` hoặc `order.user_id`) + Học sinh (`student.student_user_id` nếu khác) | `action_url = "/orders/{id}"`, hiển thị số tiền & hạn mức còn lại. |
| `LOW_SPENDING_LIMIT` | Ngay sau `PURCHASE_SUCCESS` bằng thẻ học sinh | Phụ huynh (`student.user_id`) | Chỉ phát khi `daily_limit > 0` và `remaining_limit_snapshot <= floor(daily_limit * 0.2)`. `severity = WARNING`, `action_url = "/canteen"`. |
| `PAYMENT_FAILED` | Ngoài transaction rollback tại `OrderService` (`INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`) hoặc `VnpayGatewayService::handleIpn` (thất bại) | Phụ huynh (`user_id` của thẻ/đơn) | `severity = ERROR`, không lộ thông tin nhạy cảm, `action_url = "/canteen"` hoặc `"/orders/{id}"`. |
| `ORDER_REFUNDED` | Sau commit `OrderService::transitionStatus(..., CANCELLED)` khi đơn đã thanh toán thẻ | Phụ huynh (`student.user_id` / `order.user_id`) | `severity = INFO`, ghi rõ số tiền hoàn & mã đơn. |
| `ORDER_STATUS_UPDATED` | Sau commit `OrderService::transitionStatus` khi đơn chuyển `READY` hoặc `CANCELLED` (chưa thanh toán) | Chủ đơn (`order.user_id`) | `action_url = "/orders/{id}"`. |
| `BANK_LINK_CHANGED` | Sau commit `BankLinkingService::verifyOtp` (liên kết thành công), `verifyUnlinkOtp` (hủy liên kết), hoặc `StudentCardService::updateCardStatusByAdmin` (`LOCKED`/`REVOKED`) | Phụ huynh (`student.user_id`) | `severity = SUCCESS` hoặc `WARNING`, chỉ hiển thị STK che mờ (`****1234`). |
| `SPENDING_LIMIT_UPDATED` | Sau commit `SpendingLimitService::updateDailyLimit` / `updateDailyLimitByAdmin` | Phụ huynh (`student.user_id`) + Học sinh (nếu có) | `action_url = "/canteen/limit-history"`. |

### 4.2. Push Drivers (`config/services.php` → `services.notifications`)
- `NOTIFICATION_PUSH_DRIVER`: `log` (mặc định), `zalo_oa`, `zalo_zns`.
- **`LogPushNotificationChannel`:** Ghi log có cấu trúc (không chứa PII thô), trả `status = SIMULATED`, `reference = LOG-PUSH-...`.
- **`ZaloOaPushChannel`:** Gọi `POST https://openapi.zalo.me/v3.0/oa/message/cs` với header `access_token: ZALO_OA_ACCESS_TOKEN` và `recipient.user_id = $user->zalo_user_id`. Nếu thiếu token hoặc user chưa có `zalo_user_id` → trả `SKIPPED` (hoặc `FAILED` khi HTTP/API lỗi) mà không ném ngoại lệ ra luồng nghiệp vụ.
- **`ZaloZnsPushChannel`:** Gọi `POST https://business.zalo.me/api/template/message` với `phone` chuẩn hóa `84xxxxxxxxx`, `template_id: ZALO_ZNS_TEMPLATE_ID`, `mode: development` (khi sandbox=true). Nếu thiếu cấu hình hoặc user chưa có SĐT → trả `SKIPPED`/`FAILED` an toàn.

### 4.3. REST API Thông báo (`api/routes/api.php` — `auth:sanctum`)
- `GET /api/v1/notifications?page=1&per_page=15&unread_only=0&type=...`: Trả danh sách phân trang của riêng `$request->user()` kèm `meta.unread_count`.
- `GET /api/v1/notifications/unread-count`: Trả `{ data: { unread_count: int } }`.
- `PATCH /api/v1/notifications/{id}/read`: Đánh dấu đã đọc (IDOR `404` nếu không thuộc user).
- `POST /api/v1/notifications/read-all`: Đánh dấu toàn bộ chưa đọc của user là đã đọc, trả `{ data: { marked_count: int, unread_count: 0 } }`.

### 4.4. Admin Blade (`api/routes/web.php`)
- `GET /admin/canteen/notifications` (`auth`, `role:staff,admin`): Bảng nhật ký thông báo đã phát sinh (lọc theo `type`, `push_status`, phân trang) + liên kết trên thanh điều hướng Admin.

---

## 5. Thiết kế Frontend Mini App (`miniapp/`) & Xử lý 3 Deferred Minors Slice P3

1. **Notification Center (`/notifications`):**
   - Module `src/features/notifications/{types,api,helpers,helpers.test}.ts`:
     - `isSafeInternalRoute(path?: string | null): boolean` (chỉ chấp nhận đường dẫn bắt đầu bằng `/` và không bắt đầu bằng `//`, chống open redirect).
     - `getNotificationTone(severity)`: màu sắc WCAG-safe cho `INFO`, `SUCCESS`, `WARNING`, `ERROR`.
   - `BottomNav`: 5 tab gọn (`Thực đơn`, `Căn tin`, `Giỏ`, `Đơn`, `Thông báo`), tự query `unread-count` khi `currentUser !== null` (`refetchInterval: 20_000`) và hiển thị badge số chưa đọc (`99+`).
   - `NotificationsPage` (`/notifications`): Protected Lazy Auth (`ZaloPhoneLoginPanel` khi chưa đăng nhập), bộ lọc `Tất cả` / `Chưa đọc`, nút `"Đọc tất cả"`, danh sách thông báo có dấu chấm chưa đọc, click để đánh dấu đã đọc và mở `action_url` an toàn.
2. **Xử lý 3 Deferred Minors từ Slice P3:**
   - **P3-Minor #1:** Roving `tabIndex={isChecked ? 0 : -1}` trên 3 nút `role="radio"` ở `CartPage`; ẩn nút `"Đổi hạn mức"` / đổi chữ `"Thiết lập hạn mức"` thành `"Quay lại Căn tin số"` khi `currentUser.role === "student"` trên `CartPage`, `LimitHistoryPage`, `CanteenHubPage`.
   - **P3-Minor #2:** Trên `MenuPage`, kiểm tra trước khi gọi `addCartItem()` xem số lượng trong giỏ đã đạt `getMaxAddableQuantity(product)` chưa để báo snackbar `"Đã đạt số lượng tồn kho tối đa..."` thay vì `"Đã thêm..."`; trên `/admin/pos`, prepend động dòng đơn mới vào danh sách `"Đơn POS gần đây hôm nay"` ngay sau khi checkout thành công.
   - **P3-Minor #3:** Trên `ProductManagementController::update()` + `admin/products/form.blade.php`, gửi kèm hidden `original_stock_quantity` và khóa `Product::lockForUpdate()` trong `DB::transaction()` — chỉ cập nhật `stock_quantity` khi Admin thực sự thay đổi giá trị, tránh ghi đè lượt trừ kho đồng thời từ đơn hàng.

---

## 6. Tiêu chí Nghiệm thu (Acceptance Criteria — Slice P4)

1. **NOTIF-01 (Event-Driven Dispatch):** Mua hàng thẻ thành công sinh `PURCHASE_SUCCESS` (kèm `LOW_SPENDING_LIMIT` khi còn `<= 20%`); thanh toán lỗi (`INSUFFICIENT_BANK_BALANCE`/`BANK_DECLINED`/`GATEWAY_TIMEOUT`/VNPay fail) sinh `PAYMENT_FAILED`; hủy đơn hoàn tiền sinh `ORDER_REFUNDED`; liên kết/hủy liên kết ngân hàng sinh `BANK_LINK_CHANGED`; đổi hạn mức sinh `SPENDING_LIMIT_UPDATED`.
2. **NOTIF-02 (Push Resilience & Drivers):** `ZaloOaPushChannel` & `ZaloZnsPushChannel` hoạt động đúng chuẩn API (kiểm thử qua `Http::fake()`); khi kênh push lỗi 500/timeout, bản ghi In-App vẫn lưu thành công (`push_status = FAILED`) và **không bao giờ** làm lỗi hay rollback giao dịch tài chính.
3. **NOTIF-03 (Notification API & IDOR):** `GET /notifications`, `unread-count`, `PATCH /{id}/read`, `POST /read-all` hoạt động chính xác; user khác truy cập thông báo của người khác nhận `404`; không có STK/CCCD/OTP thô trong `title`/`body`/`meta`.
4. **NOTIF-04 (Mini App & Admin UI):** Mini App có `/notifications` (Lazy Auth), badge unread trên `BottomNav`, lọc chưa đọc, đọc tất cả, điều hướng an toàn `isSafeInternalRoute`; Admin Blade `/admin/canteen/notifications` hiển thị nhật ký sạch (`staff`/`admin` xem được, `parent`/`student` bị `403`).
5. **SEC-E2E-01 (Security Hardening & Full-System E2E):** `SecurityHeadersMiddleware` gắn đủ 3 header bảo mật; bộ test `ProductionReadinessEndToEndTest` kiểm chứng liên hoàn P1→P2→P3→P4 PASS 100%; hoàn tất báo cáo kiểm toán [truc3-security-e2e-audit.md](file:///d:/Coding/TitKul_ProjectTest/TestPart2/docs/testing/truc3-security-e2e-audit.md).
6. **P3-MINOR-01..03:** Đã xử lý trọn vẹn cả 3 Deferred Minors từ Slice P3.
7. **Quality Gates:** `vendor/bin/pint --test`, `php artisan test --compact`, `migrate:fresh --force --seed`, `npm test`, `npx tsc --noEmit`, `npm run build` và Independent Code Review đều PASS sạch.

---

## 7. Xác nhận Phê duyệt

Nếu bạn đồng ý với đặc tả chi tiết **Slice P4** trên, hãy phản hồi **Phê duyệt (Approve)** để tôi lập kế hoạch chi tiết (`superpowers:writing-plans`) và triển khai trọn vẹn lát cắt cuối cùng của Trục 3 (implementation → testing → independent code review → verification → handoff).
