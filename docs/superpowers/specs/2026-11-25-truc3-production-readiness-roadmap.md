# Phân tích Khoảng trống (Gap Analysis) & Lộ trình Kiến trúc Trục 3 — Nâng cấp Smart Canteen Chuẩn Production-Ready

- **Ngày lập:** 2026-11-25
- **Trạng thái:** `PROPOSED_FOR_APPROVAL`
- **Nền tảng hiện tại trên `main`:**
  - Trục 1 (Sub-repos + Docker + CI) + Trục 2 · Slice D1–D3 (`root @ fc93101`, `api @ c9caf98`, `miniapp @ b73ce24`) + bản đánh bóng `chore/d3-minor-polish`.
- **Nguồn đối chiếu:** `FEATURES.md` (Mục 7 — 16 nghiệp vụ Production-Ready & Mục 9 — 10 module Căn tin số), `AGENTS.md`, source code thực tế tại `api/` và `miniapp/`.

---

## 1. Đối chiếu 10/10 Module Căn tin Số (`FEATURES.md` Mục 9)

| # | Module (`FEATURES.md` Mục 9) | Hiện trạng trên `main` | Đánh giá | Khoảng trống cần bổ sung ở Trục 3 |
|---|---|---|---|---|
| 1 | **Authentication Module** | `POST /api/v1/auth/zalo` (access token / `zalo_user_id`), `POST /api/v1/auth/demo`, Admin Blade session login. `AuthBootstrap` tự gọi `ensureAuthenticated()` toàn app. | **Đáp ứng một phần (60%)** | Chưa có đăng nhập bằng **SĐT trên Zalo** (`zmp-sdk getPhoneNumber` → server-side Zalo Graph API), chưa áp dụng **Lazy Auth** (tách rõ màn hình Public không cần đăng nhập vs màn hình Protected). |
| 2 | **Student Management** | Bảng `students`, `StudentCardSeeder`, hiển thị trong Admin Blade & Mini App. | **Đáp ứng một phần (65%)** | Chưa có màn hình Admin CRUD học sinh & gắn/chuyển nhiều học sinh cho 1 tài khoản phụ huynh (hiện tạo qua seeder/DB). |
| 3 | **Student Card Module** | `CARD-01..07`, `ACTIVE`/`LOCKED`/`REVOKED`, Admin khóa/mở/thu hồi tự động unlink. | **Đáp ứng mạnh (90%)** | Bổ sung cấp phát thẻ mới / cấp lại thẻ trên Admin Blade. |
| 4 | **Banking Integration** | `BankingGatewayInterface` + `MockEpayBankingGateway`, AES-256 + masking, 7 ngân hàng VN. | **Đáp ứng một phần (55%)** | Mới chỉ có mock luôn thành công; cần **Sandbox Banking Gateway** có số dư giả lập, lỗi không đủ tiền/từ chối/timeout, Webhook HMAC-SHA256 và Adapter **VNPay Sandbox / PayOS (VietQR) Sandbox**. |
| 5 | **OTP Verification** | 6 chữ số băm bcrypt, TTL, cooldown cả route & service, lockout, `BANK_LINK` & `BANK_UNLINK`. | **Đáp ứng mạnh (80%)** | Chưa có tầng **`SmsGatewayInterface`** nối cổng SMS/ZNS thật (eSMS / SpeedSMS / Twilio / Zalo ZNS) để gửi OTP tới SĐT thật của phụ huynh. |
| 6 | **Spending Limit** | `LIMIT-01..13`, lazy daily reset `Asia/Ho_Chi_Minh`, audit log, Admin override. | **Đáp ứng 100%** | Giữ nguyên kiến trúc, bổ sung phát sự kiện cảnh báo khi chạm ngưỡng thấp (<=20%) sang Notification Module. |
| 7 | **Order Management** | Server pricing, idempotency, optimistic locking, status logs, hủy hoàn hạn mức, POS `#POS-...`. | **Đáp ứng mạnh (85%)** | Thiếu **Quản lý tồn kho (`stock_quantity`)** trừ/hoàn nguyên tử và giao diện **POS Quầy Căn tin** riêng cho Staff (`/admin/pos`). |
| 8 | **Payment Processing** | Trừ hạn mức + `chargeAutoDebit`/`refundAutoDebit` trong `OrderService`. | **Đáp ứng một phần (60%)** | Thiếu bảng sổ cái bất biến `payment_transactions`, chưa xử lý tường minh `INSUFFICIENT_FUNDS`, `PAYMENT_FAILED`, `GATEWAY_TIMEOUT`, và chưa có cổng thanh toán trực tuyến phổ biến (VNPay / VietQR). |
| 9 | **Transaction History** | `ORDER-01..07`, `DETAIL-01..14`, phân trang theo thẻ & toàn bộ đơn. | **Đáp ứng mạnh (85%)** | Thiếu module **Đối soát giao dịch ngân hàng (Reconciliation)** đối chiếu sổ cái nội bộ với sao kê Sandbox/Gateway. |
| 10 | **Notification Module** | **Chưa triển khai.** | **Chưa có (0%)** | Cần xây mới hoàn chỉnh: bảng `notifications`, Event/Listener, `NotificationChannelInterface` (In-App + Zalo OA/ZNS Push), API đọc/đánh dấu đã đọc & màn hình Thông báo trên Mini App. |

---

## 2. Phân tích Chi tiết 3 Yêu cầu Trọng tâm mới

### 2.1. Xác thực bằng Số điện thoại Zalo (Zalo Phone Auth) & Ma trận Public vs Protected

#### A. Luồng kỹ thuật chuẩn Zalo Mini App (Bảo mật phía Server)
1. Trên Mini App (`miniapp`), sử dụng `zmp-sdk`:
   - Gọi `getAccessToken({})` để lấy `access_token` của phiên Zalo hiện tại.
   - Gọi `getPhoneNumber({})` (yêu cầu người dùng nhấn xác nhận cấp quyền SĐT ngay trên popup chuẩn Zalo) → nhận **`code` (phone token tạm thời)**.
2. Mini App gửi `{ access_token, phone_code }` lên `POST /api/v1/auth/zalo-phone`.
3. Backend Laravel (`ZaloIdentityService::resolvePhoneNumber($accessToken, $phoneCode)`):
   - Gọi `GET https://graph.zalo.me/v2.0/me/info` với header `access_token`, `code: $phoneCode`, `secret_key: config('services.zalo.app_secret')` (tuyệt đối không lưu `ZALO_APP_SECRET` ở frontend!).
   - Chuẩn hóa SĐT Việt Nam (`84901234567` → `0901234567`).
   - Liên kết tài khoản phụ huynh theo thứ tự ưu tiên: khớp `phone` (hoặc `zalo_user_id`), cập nhật `phone`, `phone_verified_at = now()`, `zalo_user_id`, `role = parent` (nếu là tài khoản mới), cấp Sanctum Bearer Token.
4. **Môi trường Local/Dev/Testing (không chạy trong app Zalo thật):**
   - Cung cấp `POST /api/v1/auth/demo-phone` (nhập SĐT demo hợp lệ `0xxxxxxxxx`, chỉ hoạt động khi `APP_ENV in [local, testing]` và `DEMO_LOGIN_ENABLED=true`, fail-closed ở production) để kiểm thử trọn vẹn luồng SĐT + gửi SMS OTP thật ngay trên trình duyệt localhost!

#### B. Ma trận Phân quyền Truy cập — Màn hình/API nào Public vs Cần Đăng nhập?

| Màn hình / Chức năng | Yêu cầu đăng nhập SĐT Zalo? | Lý do nghiệp vụ & UX |
|---|---|---|
| **Thực đơn (`/`), Tìm kiếm, Lọc danh mục** | **KHÔNG (Public)** | Phụ huynh/khách vào Mini App xem ngay thực đơn, giá món, tình trạng còn/hết món không bị chặn màn hình login (tối ưu conversion & tuân thủ Zalo Mini App UX Guidelines). |
| **Chi tiết món ăn (`/products/:id`)** | **KHÔNG (Public)** | Xem mô tả, hình ảnh, giá bán công khai. |
| **Giỏ hàng (`/cart` — thêm/xóa/tăng giảm món)** | **KHÔNG (Public, lưu cục bộ)** | Cho phép khách chọn món tự do trước; chỉ bật **Zalo Phone Login Sheet** khi nhấn **"Đặt đơn"** hoặc chọn thanh toán **Thẻ học sinh**. |
| **Danh sách ngân hàng hỗ trợ (`GET /api/v1/canteen/banks`)** | **KHÔNG (Public)** | Dữ liệu danh mục công khai. |
| **Đặt đơn hàng (`POST /api/v1/orders`)** | **CÓ (Bắt buộc)** | Gắn chủ sở hữu đơn hàng, chống trùng đơn (`Idempotency-Key` theo `user_id`), trừ hạn mức thẻ học sinh. |
| **Đơn hàng của tôi (`/orders`, `/orders/:id`)** | **CÓ (Bắt buộc)** | Dữ liệu giao dịch cá nhân (IDOR-protected). |
| **Phân hệ Căn tin số (`/canteen`, Liên kết ngân hàng, OTP, Hạn mức, Lịch sử hạn mức, Hủy liên kết)** | **CÓ (Bắt buộc — SĐT đã xác thực)** | Truy cập hồ sơ học sinh, thông tin tài khoản ngân hàng che mờ, gửi OTP SMS về SĐT phụ huynh, cấu hình tài chính. |
| **Thông báo giao dịch (`/notifications`)** | **CÓ (Bắt buộc)** | Dữ liệu thông báo cá nhân theo tài khoản phụ huynh. |

> **Thay đổi quan trọng ở Frontend:** Bỏ việc ép `AuthBootstrap` tự đăng nhập ngay khi mở `/`. Thay bằng cơ chế **Lazy Auth**: giữ sẵn token nếu đã đăng nhập, còn khi người dùng chưa đăng nhập mà vào tab **Căn tin số**, **Đơn của tôi**, **Thông báo**, hoặc bấm **Đặt đơn** tại Giỏ hàng → hiển thị **Zalo Phone Login Prompt** gọn gàng ("Xác thực bằng SĐT Zalo để quản lý thẻ con & đặt món").

---

### 2.2. Nối cổng SMS OTP gửi về Số điện thoại thật (`SmsGatewayInterface`)

Thiết kế chuẩn **Strategy Pattern** qua `config/services.php` (`SMS_DRIVER=log|esms|twilio|zns`):
1. **`SmsGatewayInterface::sendOtp(string $phoneNumber, string $otpCode, OtpPurpose $purpose, int $ttlSeconds): SmsDeliveryResult`**
2. **Các Driver triển khai:**
   - **`LogSmsGateway` (`SMS_DRIVER=log`, mặc định cho `testing`/CI):** Không gọi mạng thật, ghi log có che SĐT (`090****567`) và không log plaintext OTP khi `APP_ENV=production`.
   - **`EsmsSmsGateway` (`SMS_DRIVER=esms` — khuyên dùng số 1 cho SĐT Việt Nam):** Kết nối REST API chính thức của **eSMS.vn** (hỗ trợ `Sandbox=1` để test miễn phí không tốn cước và `Sandbox=0` để gửi tin nhắn SMS thật về SĐT Việt Nam `09x/03x/07x/08x` với Brandname/Đầu số CSKH). Có timeout `8s`, retry `1` lần khi lỗi mạng tạm thời, map mã lỗi nhà cung cấp thành `SmsDeliveryException` (`503 SERVICE_UNAVAILABLE`, không tạo phiên OTP "mồ côi" khi gửi SMS thất bại!).
   - **`TwilioSmsGateway` (tùy chọn chuẩn quốc tế `SMS_DRIVER=twilio`):** Sử dụng Twilio Messages REST API (hỗ trợ E.164 `+84...` và Twilio Test Credentials).
3. **Liên thông với Zalo Phone Auth:** Khi phụ huynh đã đăng nhập bằng SĐT Zalo (`$user->phone`), form liên kết ngân hàng tự động điền sẵn SĐT đã xác thực; mã OTP liên kết & hủy liên kết được chuyển qua `SmsGatewayInterface` tới đúng SĐT thật của phụ huynh, kèm ghi nhận trạng thái gửi (`delivery_provider`, `delivery_reference`) trong `otp_verifications` (không lưu plaintext OTP).

---

### 2.3. Giải pháp Cổng Ngân hàng & Thanh toán Chuẩn Production-Ready (Local/Dev/Testing Sandbox)

Vì ngân hàng thật tại VN (MB Bank / EPay auto-debit) yêu cầu ký hợp đồng pháp nhân doanh nghiệp, giải pháp **Production-Ready nhất cho Local/Dev/Testing** là kiến trúc **Dual-Adapter Sandbox**:
1. **`SandboxBankingGateway` (cho Trích nợ Tự động Thẻ học sinh — chuẩn hóa mô phỏng MB Bank / EPay):**
   - Lưu số dư & trạng thái tài khoản ngân hàng giả lập trong bảng `sandbox_bank_accounts` (có thể điều chỉnh số dư, bật/tắt trạng thái khóa/timeout trực tiếp từ Admin Blade hoặc quy ước hậu tố STK khi test: vd STK kết thúc `...0000` luôn trả `INSUFFICIENT_FUNDS`, `...9999` trả `GATEWAY_TIMEOUT`, `...8888` trả `BANK_DECLINED`).
   - Hỗ trợ **trừ tiền thật trên số dư sandbox**, **hoàn tiền về số dư sandbox**, **ký & xác thực Webhook HMAC-SHA256** (`X-Gateway-Signature` + `X-Gateway-Timestamp` chống replay 5 phút), và xuất **sao kê ngày (`fetchSettlementStatement($date)`)** phục vụ đối soát!
2. **`VnpaySandboxGateway` (hoặc `PayOsSandboxGateway` — Cổng Thanh toán Phổ biến tại VN có Sandbox công khai miễn phí):**
   - **VNPay Sandbox** (`https://sandbox.vnpayment.vn/paymentv2/vpcpay.html`) có môi trường test công khai miễn phí (chỉ cần đăng ký `TMN_CODE` & `HASH_SECRET` sandbox miễn phí), hỗ trợ thẻ ATM nội địa NCB test, ký `HMAC-SHA512`, Return URL & IPN Webhook chuẩn Production.
   - Dùng cho đơn hàng Mini App không dùng thẻ học sinh (hoặc thanh toán QR/VNPay thay tiền mặt tại quầy), đồng thời minh chứng tích hợp cổng thanh toán bên thứ ba thật sự chạy end-to-end!

---

## 3. Rà soát Đầy đủ 16/16 Nghiệp vụ Production-Ready (`FEATURES.md` Mục 7)

| # | Nghiệp vụ | Trạng thái hiện tại | Chi tiết hiện có vs Khoảng trống | Slice xử lý |
|---|---|---|---|---|
| 1 | **Tích hợp API thực tế với Bank** | **Một phần (45%)** | Có `BankingGatewayInterface` + `MockEpayBankingGateway`. Thiếu `SandboxBankingGateway` có số dư, mã lỗi chuẩn, webhook HMAC-SHA256 & HTTP client adapter cấu hình sẵn cho Production. | **Slice P2** |
| 2 | **Tích hợp API thanh toán EPay / Cổng phổ biến** | **Một phần (35%)** | Mới có mock EPay nội bộ. Cần bổ sung adapter **VNPay Sandbox** (hoặc **PayOS VietQR Sandbox**) chuẩn checksum + IPN callback + sổ cái `payment_transactions`. | **Slice P2** |
| 3 | **Trừ tiền tự động khi học sinh mua hàng** | **Đáp ứng tốt (85%)** | Đã trừ `spent_today` & gọi `chargeAutoDebit` nguyên tử trong transaction ở cả Cart & POS. Cần nâng cấp trừ trên số dư `sandbox_bank_accounts` & ghi sổ cái `payment_transactions`. | **Slice P2** |
| 4 | **Tự động làm mới hạn mức mỗi ngày** | **Đáp ứng 100%** | Đã có lazy daily reset theo `Asia/Ho_Chi_Minh` (`spent_date`) trên cả đọc & ghi, đã có test chuyển giao 23:59:59 → 00:00:00 VN. (Có thể thêm console command snapshot nếu cần báo cáo cuối ngày.) | *Đã hoàn tất (D2)* |
| 5 | **Xử lý khi chi tiêu vượt hạn mức** | **Đáp ứng 100%** | Chặn `422 SPENDING_LIMIT_EXCEEDED` kèm `remaining_today`, kể cả khi hạ hạn mức thấp hơn `spent_today`, có test đầy đủ. | *Đã hoàn tất (D2)* |
| 6 | **Xử lý khi tài khoản ngân hàng không đủ tiền** | **Chưa có (0%)** | `MockEpayBankingGateway` luôn trả `charged: true`. Cần `InsufficientFundsException` (`422 INSUFFICIENT_BANK_BALANCE`), rollback transaction không trừ hạn mức, ghi `payment_transactions.status = FAILED` & gửi thông báo cho phụ huynh. | **Slice P2** |
| 7 | **Xử lý giao dịch thất bại** | **Chưa có (0%)** | Chưa xử lý `GATEWAY_TIMEOUT` / `BANK_DECLINED` / trạng thái `PENDING_RECONCILE` khi timeout, chưa lưu lịch sử giao dịch lỗi để đối soát/thử lại an toàn. | **Slice P2** |
| 8 | **Hoàn tiền và hủy đơn hàng** | **Đáp ứng mạnh (90%)** | Đã hoàn `spent_today` (nếu cùng ngày VN), đổi `PAID → REFUNDED`, gọi `refundAutoDebit` với thứ tự khóa chuẩn. Cần ghi thêm dòng `REFUND` vào `payment_transactions` & hoàn số dư sandbox + hoàn tồn kho. | **Slice P2 & P3** |
| 9 | **Đối soát giao dịch ngân hàng** | **Chưa có (0%)** | Chưa có bảng `reconciliation_batches`/`reconciliation_items`, chưa có job so khớp giao dịch nội bộ với sao kê ngân hàng & màn hình Admin xử lý lệch. | **Slice P2** |
| 10 | **Hệ thống POS tại căn tin** | **Một phần (45%)** | Đã có API `POST /api/v1/canteen/cards/{cardId}/pos-purchases` (`#POS-...`) và nút demo trên Mini App. Thiếu giao diện **POS Quầy Căn tin chuyên dụng cho Staff** (`/admin/pos`: quét/nhập mã thẻ, xem hạn mức tức thì, chọn nhiều món, in/xem biên lai). | **Slice P3** |
| 11 | **Quản lý sản phẩm và tồn kho** | **Một phần (55%)** | Đã có CRUD sản phẩm/danh mục & bật/tắt `is_available`. Thiếu **số lượng tồn kho (`stock_quantity`)**, trừ kho nguyên tử chống oversell, tự hoàn kho khi hủy đơn & cảnh báo sắp hết hàng. | **Slice P3** |
| 12 | **Quản lý nhiều học sinh trên một tài khoản** | **Đáp ứng mạnh (75%)** | Schema `users 1-N students` & UI tab chọn đa thẻ trên Mini App đã hoạt động đồng bộ (`selectedCardIdAtom`). Thiếu màn hình Admin CRUD thêm/gắn nhiều học sinh & phát hành thẻ cho 1 phụ huynh. | **Slice P3** |
| 13 | **Phân quyền Admin, Parent, Student, Staff** | **Một phần (75%)** | Đã có `admin`, `staff`, `customer` với RBAC nghiêm ngặt. Thiếu vai trò `student` (chỉ xem thẻ & giao dịch của mình, không được chỉnh hạn mức/liên kết ngân hàng) và chuẩn hóa ngữ nghĩa `parent` (tương thích `customer`). | **Slice P3** |
| 14 | **Push Notification khi phát sinh giao dịch** | **Chưa có (0%)** | Chưa có Notification Module (bảng `notifications`, event-driven dispatch, In-App Notification Center + adapter Zalo OA/ZNS Push). | **Slice P4** |
| 15 | **Backend API và Database** | **Đáp ứng mạnh (90%)** | 17 migrations PostgreSQL 17, REST API v1 chuẩn hóa, Docker multi-stage + CI. Sẽ bổ sung thêm các bảng mới theo từng slice P1–P4 (100% additive). | **Slices P1–P4** |
| 16 | **Cơ chế mã hóa và bảo mật thực tế** | **Đáp ứng mạnh (85%)** | Đã có AES-256 cho STK/CCCD, bcrypt OTP, data masking, Sanctum, IDOR protection, rate limiting, fail-closed debug OTP. Cần bổ sung chữ ký Webhook HMAC-SHA256 + anti-replay, bảo vệ PII log sanitizer & xác thực SĐT Zalo bằng `ZALO_APP_SECRET` phía server. | **Slices P1–P4** |

---

## 4. Lộ trình Triển khai Trục 3 Đề xuất (4 Slices Độc lập, Gia tăng Chuẩn Production-Ready)

| Slice | Tên & Phạm vi | Phụ thuộc | Đầu ra kiểm chứng được |
|---|---|---|---|
| **Slice P1** | **Zalo Phone Auth (Lazy Auth) & Real SMS OTP Gateway** | `main` | Đăng nhập SĐT Zalo + fallback demo SĐT local, tách Public vs Protected trên Mini App, `SmsGatewayInterface` (`log` + `esms` thật + `twilio`), gửi OTP qua SĐT phụ huynh. |
| **Slice P2** | **Sandbox Banking & VNPay/PayOS Gateway, Payment Ledger, Lỗi Giao dịch & Đối soát** | Slice P1 | Bảng `payment_transactions`, `sandbox_bank_accounts`, `reconciliation_batches/items`, xử lý `INSUFFICIENT_BANK_BALANCE` / `GATEWAY_TIMEOUT` / `BANK_DECLINED`, Webhook HMAC-SHA256, adapter VNPay Sandbox, màn hình Admin Đối soát. |
| **Slice P3** | **POS Quầy Căn tin (Staff), Quản lý Tồn kho, CRUD Đa Học sinh & RBAC 4 Vai trò** | Slice P2 | Trừ/hoàn tồn kho nguyên tử, màn hình `/admin/pos` quét/nhập mã thẻ thanh toán nhiều món, CRUD Học sinh & Thẻ trên Admin Blade, RBAC 4 vai trò (`admin`, `staff`, `parent`, `student`). |
| **Slice P4** | **Notification Module (In-App + Push Zalo OA/ZNS) & Security/E2E Audit toàn hệ thống** | Slice P3 | Event-driven notifications khi mua hàng/hoàn tiền/lỗi thanh toán/chạm ngưỡng hạn mức/liên kết ngân hàng, UI Thông báo trên Mini App, kiểm chứng E2E + bảo mật tổng thể. |

---

## 5. Các Điểm Cần Làm rõ / Chốt Quyết định trước khi Thiết kế Chi tiết Slice P1

1. **Nhà cung cấp SMS OTP thật cho Slice P1:**
   - **Đề xuất:** Cài đặt chuẩn `SmsGatewayInterface` với driver mặc định `log` (cho test tự động/CI) + **`esms` (eSMS.vn — tối ưu cho SĐT Việt Nam, có Sandbox mode & gửi SMS thật)** + **`twilio` (quốc tế)**. Anh/chị đã có tài khoản/API key nhà cung cấp SMS nào (như eSMS, SpeedSMS, Vietguys, Twilio, hay Zalo ZNS) muốn ưu tiên gắn driver chính không?
2. **Cổng thanh toán Sandbox phổ biến cho Slice P2:**
   - **Đề xuất:** Kết hợp **`SandboxBankingGateway`** (mô phỏng trích nợ tự động MB Bank/EPay có số dư, lỗi không đủ tiền, webhook HMAC-SHA256, đối soát) **+** **`VnpaySandboxGateway`** (cổng thanh toán phổ biến nhất VN, có Sandbox công khai miễn phí) hoặc **PayOS (VietQR)**. Anh/chị ưu tiên **VNPay Sandbox**, **PayOS (VietQR)**, hay chỉ cần **SandboxBankingGateway + VNPay Sandbox**?
3. **Thứ tự triển khai:**
   - **Đề xuất:** Đi tuần tự **Slice P1 → P2 → P3 → P4** (mỗi Slice có Spec → Plan → Implement → Test → Independent Code Review → Verify → Merge giống chuẩn Trục 2).
