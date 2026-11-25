# Đặc tả Thiết kế — Trục 2 · Slice D3: Hủy Liên kết Ngân hàng qua OTP, Quản trị Blade Căn tin Số, Dọn 7 Deferred Minors & Kiểm chứng E2E 4 Luồng Nghiệp vụ

- **Ngày lập:** 2026-11-25
- **Mã tác vụ:** `CANTEEN-D3-UNLINK-ADMIN-E2E`
- **Trạng thái:** `APPROVED (2026-11-25)`
- **Tiền đề đã merge vào `main`:**
  - Trục 1 (`api/` & `miniapp/` sub-repos + Docker + CI)
  - Slice D1 (`CARD-01..07`, `BANK-01..16`, `OTP-01..07`) — `root @ 4c76e83`, `api @ fa9418a`, `miniapp @ 01c8db3`
  - Slice D2 (`LIMIT-01..13`, `ORDER-01..07`, `DETAIL-01..14`) — `root @ 941014c`, `api @ 89d76d1`, `miniapp @ 14a433b`
- **Nguồn yêu cầu chuẩn:** `FEATURES.md` (Mục 3.3 `OTP-08`, Mục 3.7 `UNLINK-01..07`, Mục 4 màn hình #13–#15, Mục 9 — 4 luồng nghiệp vụ trọng tâm), `DESIGN.md`, `docs/api-contract.md`, `docs/database.md`, `AGENTS.md`.

---

## 1. Mục tiêu & Phạm vi

### 1.1. Trong phạm vi (In Scope)
1. **Hủy liên kết tài khoản ngân hàng qua OTP 6 chữ số (`UNLINK-01..07`, `OTP-08`):**
   - Backend API khởi tạo hủy liên kết, xác thực OTP hủy liên kết và gửi lại OTP hủy liên kết với khóa hàng PostgreSQL, mã băm OTP, TTL, cooldown và chống brute-force.
   - Frontend Mini App: nút **Hủy liên kết ngân hàng** trên `CanteenHubPage`, hộp thoại xác nhận (`UnlinkConfirmDialog`), trang xác thực OTP hủy liên kết (`/canteen/unlink-otp`) và màn hình xác nhận **Hủy liên kết thành công** (Screen #15).
2. **Màn hình Quản trị Blade cho Phân hệ Căn tin Số (Học sinh / Thẻ / Hạn mức / Giao dịch):**
   - Bổ sung mục điều hướng **Căn tin số** trên `admin.layout`.
   - Màn hình danh sách thẻ học sinh (`GET /admin/canteen/cards`) có tìm kiếm/lọc theo trạng thái thẻ và trạng thái liên kết ngân hàng, phân trang.
   - Màn hình chi tiết thẻ (`GET /admin/canteen/cards/{studentCard}`) hiển thị hồ sơ học sinh, lịch sử liên kết ngân hàng (chỉ hiển thị dữ liệu đã che mờ — tuyệt đối không giải mã PII ra Blade), trạng thái hạn mức hôm nay (`Asia/Ho_Chi_Minh`), lịch sử điều chỉnh hạn mức và danh sách giao dịch gần đây.
   - Tác vụ quản trị dành riêng cho vai trò `admin`: khóa/mở khóa/thu hồi thẻ (`PATCH /admin/canteen/cards/{studentCard}/status`) và cập nhật hạn mức chi tiêu hằng ngày kèm audit log (`PUT /admin/canteen/cards/{studentCard}/spending-limit`).
   - Nâng cấp `admin.orders.index`, `admin.orders.show` và `admin.dashboard` hiển thị thông tin thanh toán thẻ học sinh (`student_card`, tên/lớp học sinh, `payment_method`, `payment_status`, `store_name`, `limit_remaining_after`, và khối thống kê nhanh Căn tin số hôm nay).
3. **Khắc phục trọn vẹn 7 Deferred Minors đã ghi nhận từ Slice D2:**
   - 3 mục Backend (`Admin\OrderController` eager-load, `BankLinkingService` eager-load `spendingLimit`, `StorePosPurchaseRequest` kiểm tra kiểu chuỗi).
   - 3 mục Frontend (`SpendingLimitSheet` xóa lỗi khi sửa + tiêu đề thẻ + trả focus, `CanteenHubPage` nâng tương phản thanh tiến độ + xóa `posError` khi đổi thẻ + khóa nút khi thẻ không `ACTIVE`, chuẩn hóa `timeZone: "Asia/Ho_Chi_Minh"` khi format thời gian).
   - 1 mục Tài liệu (`docs/database.md` cập nhật tiêu đề Mục 4 và sơ đồ ERD đầu file).
4. **Kiểm chứng E2E trọn vẹn 4 Luồng Nghiệp vụ Trọng tâm (`FEATURES.md` Mục 9) + RBAC Admin Blade:**
   - Test suite E2E xuyên suốt 4 luồng (Liên kết → Hạn mức → Đặt đơn/Quẹt thẻ POS/Hủy hoàn hạn mức → Hủy liên kết & Liên kết lại) trên PostgreSQL thật trong Docker.
   - Test suite RBAC & bảo mật PII cho toàn bộ tuyến Admin Blade Căn tin số.

### 1.2. Ngoài phạm vi (Out of Scope)
- Không tích hợp cổng ngân hàng/SMS thật (giữ nguyên `MockEpayBankingGateway` đứng sau `BankingGatewayInterface`).
- Không tự ý `git push`, `deploy` hay thay đổi cấu hình production bên ngoài phạm vi Local/Development/Testing.
- Không thêm migration cơ sở dữ liệu mới (schema hiện có đã đầy đủ cho Slice D3).

---

## 2. Thiết kế Cơ sở Dữ liệu & Tính toàn vẹn

**Không cần thêm migration mới** vì schema Slice D1–D2 đã chuẩn bị sẵn toàn bộ cột và chỉ mục:
- `bank_links`: đã có `status` (`ACTIVE → UNLINKED`), `unlinked_at`, và partial unique index `bank_links_one_active_per_card_idx (student_card_id) WHERE status = 'ACTIVE'`. Khi chuyển sang `UNLINKED`, khe `ACTIVE` được giải phóng ngay để phụ huynh có thể liên kết ngân hàng khác sau đó.
- `otp_verifications`: đã có `purpose` (`OtpPurpose::BANK_UNLINK`), `bank_link_id`, `phone_masked`, `code_hash`, `attempts`, `max_attempts`, `resend_count`, `max_resends`, `last_sent_at`, `expires_at`, `verified_at`, `invalidated_at`, và index `['bank_link_id', 'purpose', 'invalidated_at']`.

### Bất biến nghiệp vụ (Domain Invariants)
1. **Thứ tự khóa hàng PostgreSQL chuẩn (Deadlock-Free):** `student_cards → bank_links → otp_verifications` (và `student_cards → spending_limits → products → orders` cho luồng thanh toán/hoàn tiền).
2. **Bảo toàn `spent_today` trong ngày khi hủy liên kết:** Khi `bank_links.status` chuyển thành `UNLINKED`, bản ghi `spending_limits`, `spending_limit_logs` và `orders` được giữ nguyên để đối soát và ngăn hành vi hủy liên kết rồi liên kết lại trong cùng ngày nhằm reset `spent_today`. Trong lúc chưa liên kết, mọi thao tác thanh toán thẻ hoặc chỉnh hạn mức đều bị chặn (`422 CARD_NOT_LINKED`).
3. **Cho phép hủy liên kết kể cả khi thẻ đang `LOCKED`:** Phụ huynh luôn có quyền ngắt trích nợ tự động khi thẻ bị mất/tạm khóa (chỉ yêu cầu `student.is_active = true` và `bank_link.status = ACTIVE`).
4. **Nhập sai OTP hủy liên kết quá số lần (`attempts >= max_attempts`):** Chỉ vô hiệu hóa bản ghi `otp_verifications` của phiên hủy liên kết (`invalidated_at = now()`, trả `429 OTP_MAX_ATTEMPTS_EXCEEDED`), **không** chuyển `bank_links.status` sang `FAILED` để tránh phá hỏng liên kết đang hoạt động của phụ huynh.
5. **Quy tắc chuyển trạng thái thẻ của Admin:**
   - `ACTIVE ↔ LOCKED` được phép chuyển đổi qua lại.
   - `ACTIVE → REVOKED` và `LOCKED → REVOKED` được phép.
   - `REVOKED` là trạng thái kết thúc (terminal), không mở lại được để đảm bảo tính toàn vẹn kiểm toán. Khi Admin thu hồi thẻ đang có liên kết `ACTIVE`, hệ thống cảnh báo rõ trên UI hoặc tự động ngắt liên kết (`UNLINKED`) trong cùng transaction — **quyết định thiết kế:** khi chuyển thẻ sang `REVOKED`, tự động gọi `gateway->unlinkAutoDebit()` và chuyển `activeBankLink` (nếu có) sang `UNLINKED` (`unlinked_at = now()`) trong cùng transaction để không tồn tại thẻ đã thu hồi nhưng vẫn còn ủy quyền trích nợ `ACTIVE`.

---

## 3. Thiết kế API Hợp đồng (Backend `api/`)

### 3.1. Nhóm API Hủy liên kết Ngân hàng (`/api/v1/canteen/...`, `auth:sanctum`)

| Method | Endpoint | Middleware | Mô tả | Status Codes |
|---|---|---|---|---|
| `POST` | `/api/v1/canteen/bank-links/{bankLinkId}/unlink/initiate` | `auth:sanctum`, `throttle:10,1` | Khởi tạo yêu cầu hủy liên kết & phát hành OTP 6 số (`purpose = BANK_UNLINK`) | `201`, `404`, `409` (`BANK_LINK_NOT_ACTIVE`), `429` |
| `POST` | `/api/v1/canteen/bank-links/{bankLinkId}/unlink/verify-otp` | `auth:sanctum`, `throttle:10,1` | Xác thực OTP 6 số, gọi `unlinkAutoDebit`, chuyển `bank_links.status = UNLINKED`, trả về `StudentCardResource` | `200`, `404`, `409` (`BANK_LINK_NOT_ACTIVE`), `422` (`OTP_INVALID`, `OTP_EXPIRED`), `429` (`OTP_MAX_ATTEMPTS_EXCEEDED`) |
| `POST` | `/api/v1/canteen/bank-links/{bankLinkId}/unlink/resend-otp` | `auth:sanctum`, `throttle:10,1` | Gửi lại OTP `BANK_UNLINK` (tôn trọng cooldown `30s`, tối đa `3` lần, chặn nếu OTP đã bị khóa do nhập sai) | `200`, `404`, `409`, `422` (`OTP_EXPIRED`), `429` (`OTP_RESEND_COOLDOWN`, `OTP_MAX_RESENDS_EXCEEDED`, `OTP_MAX_ATTEMPTS_EXCEEDED`) |

- **Payload `POST .../unlink/initiate` & `POST .../unlink/resend-otp`:**
  ```json
  {
    "data": {
      "bank_link_id": 12,
      "student_card_id": 3,
      "purpose": "BANK_UNLINK",
      "status": "ACTIVE",
      "bank_code": "MB",
      "bank_name": "MB Bank",
      "account_number_masked": "****6789",
      "phone_masked": "090***5678",
      "otp": {
        "expires_in_seconds": 180,
        "resend_cooldown_seconds": 30,
        "max_attempts": 5,
        "remaining_attempts": 5,
        "max_resends": 3,
        "remaining_resends": 3
      }
    }
  }
  ```
  *(Trường `debug_otp` chỉ xuất hiện khi `APP_ENV in [local, testing]` và `BANKING_EXPOSE_DEBUG_OTP=true` — giữ nguyên nguyên tắc fail-closed từ Slice D2.)*
- **Payload `POST .../unlink/verify-otp`:** Request `{ "otp_code": "123456" }` (tái sử dụng `VerifyBankLinkOtpRequest`); Response `200 OK` chuẩn `StudentCardResource` (`is_bank_linked: false`, `bank_link: null`, vẫn kèm `spending_limit`).

### 3.2. Nhóm Route Quản trị Blade (`/admin/...`, session `auth`)

| Method | Route | Role | Chức năng |
|---|---|---|---|
| `GET` | `/admin/canteen/cards` (`admin.canteen.cards.index`) | `staff,admin` | Danh sách học sinh, thẻ, trạng thái liên kết (masked), hạn mức hôm nay (lọc/tìm kiếm/phân trang) |
| `GET` | `/admin/canteen/cards/{studentCard}` (`admin.canteen.cards.show`) | `staff,admin` | Chi tiết thẻ, lịch sử liên kết (masked), hạn mức & lịch sử đổi hạn mức, đơn hàng gắn thẻ |
| `PATCH` | `/admin/canteen/cards/{studentCard}/status` (`admin.canteen.cards.status`) | `admin` | Cập nhật trạng thái thẻ (`ACTIVE`, `LOCKED`, `REVOKED`); nếu `REVOKED` thì tự động `UNLINKED` ngân hàng |
| `PUT` | `/admin/canteen/cards/{studentCard}/spending-limit` (`admin.canteen.cards.limit`) | `admin` | Điều chỉnh hạn mức chi tiêu hằng ngày (qua `SpendingLimitService`, ghi `SpendingLimitLog` với `changed_by = admin.id`) |

---

## 4. Thiết kế Giao diện Frontend (`miniapp/` & Admin Blade)

### 4.1. Zalo Mini App (`miniapp/`, tuân thủ `DESIGN.md`)
1. **`CanteenHubPage` (`miniapp/src/pages/canteen.tsx`):**
   - Trong khối **Tài khoản ngân hàng liên kết**, bổ sung nút **Hủy liên kết ngân hàng** (`UNLINK-01`) màu cảnh báo đỏ chuẩn tương phản (`text-red-700 bg-red-50 border-red-200 hover:bg-red-100`).
   - Khi nhấn mở **`UnlinkConfirmDialog`** (`miniapp/src/components/unlink-confirm-dialog.tsx`): modal `role="dialog"`, `aria-modal="true"`, Escape đóng modal, trả focus về nút kích hoạt (`UNLINK-02..04`).
   - Áp dụng luôn các Deferred Minors trên `CanteenHubPage`:
     - Thanh tiến độ hạn mức thấp đổi sang `bg-amber-600` (và `bg-red-600` khi `remainingPct === 0`).
     - Chuyển tab chọn thẻ thì xóa `posError`.
     - Chặn thao tác Thiết lập hạn mức / Quẹt thẻ POS / Liên kết ngân hàng khi `activeCard.status !== "ACTIVE"` kèm thông báo trạng thái thẻ rõ ràng (riêng nút Hủy liên kết vẫn mở nếu thẻ `LOCKED` để phụ huynh ngắt ủy quyền).
     - Chuẩn hóa `timeZone: "Asia/Ho_Chi_Minh"` khi hiển thị thời gian.
2. **`BankUnlinkOtpPage` (`/canteen/unlink-otp`, `miniapp/src/pages/bank-unlink-otp.tsx`):**
   - Tái sử dụng `OtpCodeInput` và bộ đếm `formatCountdown` + wall-clock deadline sync (`UNLINK-05`, `UNLINK-06`, `OTP-08`).
   - Hỗ trợ đầy đủ: nhập 6 số, nút "Điền nhanh" ở chế độ Local Demo (`debug_otp`), trạng thái "Đang xác thực...", đếm ngược hết hạn & cooldown gửi lại OTP, thông báo khóa phiên và nút quay lại Căn tin số.
   - Sau khi xác thực thành công: hiển thị **Màn hình Hủy liên kết thành công** (`UNLINK-07`, Screen #15 trong `FEATURES.md`) với thông tin thẻ/ngân hàng vừa hủy và CTA **"Quay về Căn tin số"** (đã invalidate cache `canteen-cards`, `canteen-card-orders`, `canteen-limit-logs`).
3. **Các file Frontend sửa cho Deferred Minors:**
   - `miniapp/src/components/spending-limit-sheet.tsx`: xóa `serverError` khi chỉnh input/preset, hiển thị tên học sinh + mã thẻ trên tiêu đề sheet, trả focus về nút mở khi đóng.
   - `miniapp/src/pages/order-detail.tsx` & `miniapp/src/pages/limit-history.tsx`: truyền `timeZone: "Asia/Ho_Chi_Minh"` vào `toLocaleString("vi-VN", ...)`.

### 4.2. Admin Blade (`api/resources/views/admin/`)
- Dùng chung hệ thống CSS biến `--brand-*`, `.card`, `.badge`, `.btn`, `admin.partials.pagination` trong `admin/layout.blade.php`.
- Bổ sung badge ngữ nghĩa cho trạng thái thẻ (`ACTIVE`, `LOCKED`, `REVOKED`), trạng thái thanh toán (`PAID`, `REFUNDED`, `UNPAID`), trạng thái liên kết (`LINKED`, `UNLINKED`) với độ tương phản WCAG 2.1 AA.
- Tuyệt đối không render số tài khoản/CCCD/SĐT nguyên bản ra HTML (chỉ render các trường `*_masked`).

---

## 5. Kế hoạch Kiểm chứng & Tiêu chí Chấp nhận

| Hạng mục kiểm chứng | Công cụ / Test Suite | Tiêu chí đạt |
|---|---|---|
| Hủy liên kết OTP (`UNLINK-01..07`, `OTP-08`) | `api/tests/Feature/Canteen/BankUnlinkOtpTest.php` | Đủ happy path, sai OTP giảm lượt, khóa phiên (không làm hỏng `ACTIVE` bank link), hết hạn, cooldown/max resends, IDOR (`404`), thẻ `LOCKED` vẫn hủy được, không reset `spent_today` khi liên kết lại trong ngày |
| Quản trị Blade Căn tin số & RBAC | `api/tests/Feature/Admin/AdminCanteenManagementTest.php` | Đủ kiểm tra phân quyền (`guest`/`customer`/`staff`/`admin`), lọc/tìm kiếm/phân trang, không rò rỉ PII plaintext, đổi trạng thái thẻ (`REVOKED` tự hủy bank link), admin đổi hạn mức ghi log |
| E2E 4 Luồng Nghiệp vụ Trọng tâm | `api/tests/Feature/Canteen/DigitalCanteenEndToEndFlowsTest.php` | Chạy liên hoàn Flow 1 → Flow 2 → Flow 3 (đặt đơn + POS + hủy hoàn hạn mức) → Flow 4 (hủy liên kết + chặn chi tiêu + liên kết lại) trên PostgreSQL thật |
| Dọn 7 Deferred Minors | Cập nhật test tương ứng + `SpendingLimitAndOrderPaymentTest` + `miniapp` unit tests | 100% minor được phủ kiểm thử hoặc kiểm tra tĩnh |
| Cổng chất lượng toàn hệ thống | `vendor/bin/pint --test`, `php artisan test --compact`, `php artisan migrate --force --seed`, `npm test`, `npx tsc --noEmit`, `npm run build` | 0 lỗi, 0 warning phá vỡ, toàn bộ test xanh |

---

## 6. Rủi ro & Biện pháp Giảm thiểu
- **Rủi ro trùng lặp logic OTP giữa `initiateLink`/`verifyOtp`/`resendOtp` và `initiateUnlink`/`verifyUnlinkOtp`/`resendUnlinkOtp`:** Tách các helper nội bộ gọn gàng trong `BankLinkingService` (format payload theo `OtpPurpose`, kiểm tra cooldown/resend/attempt) để giữ code tối giản, dễ bảo trì và nhất quán.
- **Rủi ro lộ PII trên trang quản trị Blade:** Dùng `BankLink` model `#[Hidden]` + chỉ đọc `account_number_masked`, `national_id_masked`, `phone_masked` trong view, đồng thời khóa bằng assertion `assertDontSee` trong `AdminCanteenManagementTest`.
- **Rủi ro phá vỡ backward compatibility của Admin Orders/Dashboard:** Mọi cột/khối mới trên Blade và JSON API đều bổ sung theo kiểu additive; toàn bộ 80 test hiện có phải tiếp tục PASS 100%.
