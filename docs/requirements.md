# Đặc tả Yêu cầu Nghiệp vụ & Kỹ thuật — Smart Canteen & Phân hệ Căn tin Số (`docs/requirements.md`)

- **Dự án:** Smart Canteen & Phân hệ Căn tin Số (Zalo Mini App × React 18 × Laravel 13 × PostgreSQL 17)
- **Kiến trúc Repository:** Workspace điều phối (`TestPart2/`) + 2 Git sub-repository độc lập (`api/`, `miniapp/`), Dockerized & GitHub Actions CI.
- **Loại dự án:** Greenfield (Độc lập 100%, không kết nối cơ sở dữ liệu hay dịch vụ của hệ thống production).
- **Trạng thái:** Hoàn tất 100% MVP (`S1–S4`) + Hạ tầng (`Trục 1`) + Căn tin Số (`Trục 2 · D1–D3`) + Production-Readiness (`Trục 3 · P1–P4`) — phủ đầy **10/10 Module Căn tin Số** theo `FEATURES.md` §9.

---

## 1. Mục tiêu Nghiệp vụ (Business Objective)

Xây dựng trọn vẹn hệ sinh thái **Căn tin Số thông minh trong trường học** kết hợp đặt món trực tuyến trên Zalo Mini App, quẹt thẻ học sinh thông minh tại quầy POS, liên kết tài khoản ngân hàng trích nợ tự động, kiểm soát hạn mức chi tiêu hằng ngày, đối soát kế toán và thông báo giao dịch tức thời:

1. **Phụ huynh (`parent` / alias `customer`):** Đăng nhập qua Số điện thoại Zalo (Lazy Auth — xem thực đơn tự do trước khi đăng nhập), quản lý một hoặc nhiều thẻ học sinh của con em, liên kết/hủy liên kết tài khoản ngân hàng qua OTP SMS 6 chữ số, thiết lập/theo dõi hạn mức chi tiêu hằng ngày, đặt món trước hoặc theo dõi toàn bộ giao dịch POS/Online và nhận thông báo tức thời.
2. **Học sinh (`student`):** Đăng nhập xem thẻ của chính mình, theo dõi hạn mức còn lại trong ngày, tự đặt món trong phạm vi hạn mức phụ huynh cấp; tuyệt đối không được phép thay đổi hạn mức hay liên kết/hủy ngân hàng.
3. **Nhân viên Bếp / Thu ngân POS (`staff`):** Sử dụng Cổng Quản trị (`/admin`) và màn hình **Quầy POS (`/admin/pos`)** để tra cứu thẻ học sinh, quẹt thẻ/thu tiền mặt, in biên lai, điều phối đơn đặt trước (`PENDING → PREPARING → READY → COMPLETED / CANCELLED`) và xem danh sách thẻ/đối soát/nhật ký thông báo.
4. **Quản trị viên (`admin`):** Toàn quyền của `staff`, cộng thêm quản trị thực đơn & tồn kho (`stock_quantity`), quản lý hồ sơ học sinh & cấp phát/khóa/thu hồi thẻ, điều chỉnh hạn mức khẩn cấp, cấu hình tài khoản Sandbox ngân hàng và thực hiện đối soát/xử lý chênh lệch kế toán.

---

## 2. Phân quyền Người dùng — RBAC 4 Vai trò

| Vai trò (`UserRole`) | Định danh & Xác thực | Quyền hạn được phép | Rào chắn bảo mật |
|---|---|---|---|
| **`parent`** *(alias tương thích ngược: `customer`)* | Xác thực SĐT Zalo server-side (`POST /api/v1/auth/zalo-phone`), Zalo OAuth (`POST /api/v1/auth/zalo`), hoặc Demo Phone/Key cục bộ; nhận Sanctum Bearer Token | Xem thực đơn công khai; quản lý nhiều thẻ con em; liên kết & hủy liên kết ngân hàng qua OTP; đặt/sửa hạn mức hằng ngày; đặt đơn (`STUDENT_CARD`, `VNPAY`, `COUNTER_CASH`); theo dõi lịch sử & thông báo | Chặn `403` mọi tuyến `/admin/*` & `/api/v1/admin/*`; IDOR trả `404 Not Found` khi truy cập thẻ/đơn/thông báo của phụ huynh khác |
| **`student`** | Xác thực SĐT Zalo (`students.student_user_id = users.id`); nhận Sanctum Bearer Token | Xem thẻ học sinh gắn với mình, hạn mức còn lại, lịch sử hạn mức & đơn hàng; tự đặt đơn bằng thẻ trong hạn mức cho phép; xem thông báo của mình | Chặn `403 PARENT_ROLE_REQUIRED` ở mọi API tài chính phụ huynh (`bank-links`, `verify-otp`, `unlink`, `spending-limit`); chặn `403` trên `/admin/*` |
| **`staff`** | Xác thực Session Web (`/admin/login`) + CSRF hoặc Sanctum API | Xem Dashboard hôm nay; vận hành **POS Quầy Căn tin (`/admin/pos`)** & in biên lai; điều phối trạng thái đơn hàng; xem (read-only) Danh mục, Món ăn, Học sinh, Thẻ, Đối soát, Nhật ký Thông báo | Chặn `403` (`role:admin`) khi thêm/sửa/xóa Danh mục, Món ăn, Tồn kho, Học sinh, Trạng thái thẻ, Hạn mức Admin, Đối soát |
| **`admin`** | Xác thực Session Web (`/admin/login`) + CSRF hoặc Sanctum API | Toàn quyền của `staff` + CRUD Danh mục/Món ăn/Tồn kho, CRUD Học sinh & cấp phát/khóa/thu hồi thẻ, Admin Override hạn mức, Chạy đối soát & xác nhận xử lý chênh lệch, Cấu hình tài khoản Sandbox | Ràng buộc toàn vẹn DB (`restrictOnDelete`), không xóa danh mục có món, không xóa món/học sinh có lịch sử giao dịch, không chuyển phụ huynh khi còn liên kết ngân hàng đang mở |

---

## 3. Phủ Đầy 10/10 Module Căn tin Số (`FEATURES.md` §9)

| # | Module Nghiệp vụ | Phạm vi Yêu cầu Đã Hoàn thiện |
|---|---|---|
| 1 | **Authentication Module** | Lazy Auth trên Mini App; Zalo Phone Auth (`getPhoneNumber` → server gọi `graph.zalo.me/v2.0/me/info` bằng `ZALO_APP_SECRET`); Demo Phone local (`APP_ENV in [local, testing]`); Logout thu hồi Sanctum token. |
| 2 | **Student Management** | CRUD Học sinh trên Admin (`/admin/canteen/students`); 1 phụ huynh quản lý N học sinh; liên kết 1-1 tài khoản `student_user_id`; khóa chuyển phụ huynh khi còn `bank_links` mở. |
| 3 | **Student Card Module** | `CARD-01..07`: cấp thẻ mới/thay thế (tự thu hồi thẻ cũ), đổi trạng thái `ACTIVE ↔ LOCKED → REVOKED`, tự động ngắt `UNLINKED` ngân hàng & vô hiệu hóa OTP khi thu hồi. |
| 4 | **Banking Integration** | Strategy `BankingGatewayInterface`; `SandboxBankingGateway` mô phỏng số dư thực, chế độ lỗi (`INSUFFICIENT_FUNDS`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`) & hoàn tiền; Webhook `HMAC-SHA256` chống replay; `VNPay Sandbox` chuẩn v2.1.0 `HMAC-SHA512`. |
| 5 | **OTP Verification** | `OTP-01..08`: mã 6 chữ số băm `bcrypt`, TTL `180s`, cooldown gửi lại `30s`, tối đa `5` lần sai (`429`), tối đa `3` lần gửi lại; `SmsGatewayInterface` (`log`, `esms`, `twilio`) rollback sạch khi gửi SMS thất bại (`503`). |
| 6 | **Spending Limit** | `LIMIT-01..13`: hạn mức `0..5.000.000 ₫` (bội số `1.000 ₫`), reset hằng ngày tự động theo `Asia/Ho_Chi_Minh`, kiểm tra & hoàn hạn mức nguyên tử, nhật ký `spending_limit_logs`, cảnh báo chạm ngưỡng thấp (`<= 20%`). |
| 7 | **Order Management** | Server-side pricing (VND nguyên dương), bảo toàn giá lịch sử, `Idempotency-Key` (`201/200/409`), State Machine + Optimistic Locking (`version`), trừ/hoàn tồn kho nguyên tử `stock_quantity`, **POS Quầy Căn tin (`/admin/pos`)** + biên lai in. |
| 8 | **Payment Processing** | Sổ cái bất biến `payment_transactions` ghi nhận cả `SUCCEEDED`, `FAILED`, `TIMEOUT`, `REFUND` (ghi bền vững ngoài transaction rollback khi lỗi ngân hàng); chống double-charge & VNPay IPN idempotent. |
| 9 | **Transaction History & Reconciliation** | `ORDER-01..07`, `DETAIL-01..14`; `ReconciliationService` idempotent theo `(reconciliation_date, provider)` phát hiện 5 trạng thái (`MATCHED`, `AMOUNT_MISMATCH`, `MISSING_LOCAL`, `MISSING_IN_BANK`, `STATUS_MISMATCH`), lệnh `canteen:reconcile`, giao diện Admin `/admin/canteen/reconciliation`. |
| 10 | **Notification Module** | Bảng `canteen_notifications`, 7 sự kiện tự động sau `DB::afterCommit()` / post-catch, `PushNotificationChannelInterface` (`log`, `zalo_oa`, `zalo_zns`) lỗi push không ảnh hưởng thanh toán, Mini App `/notifications` + badge `BottomNav`, Admin log `/admin/canteen/notifications`. |

---

## 4. Yêu cầu Phi chức năng (Non-Functional Requirements)

1. **Bảo mật & Quyền riêng tư (Security & PII Protection):**
   - Mã hóa AES-256 (`encrypted` cast) cho `account_number_encrypted` và `national_id_encrypted`; OTP băm `bcrypt`.
   - `DataMaskingService` che mờ toàn bộ STK/CCCD/SĐT trước khi trả API, hiển thị Blade/POS, lưu `meta` thông báo hoặc ghi log.
   - Chữ ký số nghiêm ngặt: `appsecret_proof` (`HMAC-SHA256` Zalo), Webhook (`HMAC-SHA256` + cửa sổ `±300s` + anti-replay `UNIQUE (provider, event_id)`), VNPay (`HMAC-SHA512` + `vnp_TmnCode` + `vnp_Amount` + fail-closed trên production).
   - `SecurityHeadersMiddleware` gắn `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`; chặn open redirect qua `isSafeInternalRoute()` và `isSafeHttpUrl()`.
   - Không lưu `ZALO_APP_SECRET` hay token vào `localStorage`/`cookie` trên Mini App (in-memory only).
2. **Toàn vẹn Đồng thời & Chống Deadlock:**
   - Thứ tự khóa dòng PostgreSQL (`SELECT ... FOR UPDATE`) thống nhất toàn hệ thống: `student_cards → spending_limits → sandbox_bank_accounts → products (id ASC) → orders → payment_transactions`.
   - Partial Unique Index PostgreSQL đảm bảo tối đa 1 liên kết ngân hàng `ACTIVE` trên mỗi thẻ.
3. **Khả năng Tiếp cận & UI/UX (WCAG 2.1 AA):**
   - Bảng màu thương hiệu Smart Canteen (`#2E7D32` `brand-700` tỷ lệ tương phản `5.14:1` trên trắng), 4 trạng thái chuẩn (`Loading`, `Error`, `Empty`, `Success`), focus trap modal, roving `tabIndex` cho `radiogroup`, và `aria-live="polite"` cho badge thông báo chưa đọc.
4. **Hạ tầng & Kiểm thử Tự động:**
   - Docker multi-stage non-root (`api/` PHP 8.4 FPM + Nginx + PostgreSQL 17; `miniapp/` Nginx unprivileged) kèm GitHub Actions CI độc lập cho cả 2 repo.
   - Toàn bộ quy tắc nghiệp vụ cốt lõi được bảo vệ bằng bộ kiểm thử tự động PHPUnit trên PostgreSQL 17 và Vitest + TypeScript trên Frontend.
