# Đặc tả Thiết kế Chi tiết — Trục 3 · Slice P1: Xác thực bằng Số điện thoại Zalo (Lazy Auth) & Cổng SMS OTP Thực tế

- **Ngày lập:** 2026-11-25
- **Mã tác vụ:** `CANTEEN-P1-ZALO-PHONE-SMS-OTP`
- **Trạng thái:** `APPROVED (2026-11-25)`
- **Lộ trình cha:** `docs/superpowers/specs/2026-11-25-truc3-production-readiness-roadmap.md` (`APPROVED 2026-11-25`)
- **Nền tảng hiện tại trên `main`:** `root @ 9549812`, `api @ 22046ee`, `miniapp @ ca2e40d` (91 PHPUnit tests / 951 assertions, 33 Vitest tests).
- **Tài liệu chuẩn tham chiếu:** `FEATURES.md` (Mục 2.1, 3.3, 9), `DESIGN.md`, `miniapp/node_modules/zmp-sdk/apis/index.d.ts` (`zmp-sdk@^2.53.3`), `docs/api-contract.md`, `docs/database.md`.

---

## 1. Mục tiêu & Phạm vi

### 1.1. Trong phạm vi (In Scope)
1. **Xác thực Phụ huynh bằng Số điện thoại trên Zalo (`Zalo Phone Auth`):**
   - Backend đổi `phone_token` từ `zmp-sdk getPhoneNumber()` qua Zalo Graph API (`https://graph.zalo.me/v2.0/me/info`) bằng `ZALO_APP_SECRET` phía server, chuẩn hóa SĐT Việt Nam (`0xxxxxxxxx`), gắn định danh tài khoản phụ huynh (`users.phone`, `users.phone_verified_at`, `users.zalo_user_id`), cấp Sanctum Bearer Token.
   - Bổ sung `POST /api/v1/auth/demo-phone` (chỉ hoạt động ở `local`/`testing` khi `DEMO_LOGIN_ENABLED=true`, fail-closed `403` ở `production`) để kiểm thử trọn vẹn luồng SĐT phụ huynh trên trình duyệt localhost ngoài ứng dụng Zalo mobile.
   - Bổ sung `POST /api/v1/auth/logout` thu hồi token hiện tại.
   - Giữ tương thích ngược 100% với `POST /api/v1/auth/zalo` và `POST /api/v1/auth/demo` hiện có.
2. **Phân tách Public vs Protected bằng Lazy Auth trên Mini App:**
   - **Public (không yêu cầu đăng nhập):** Thực đơn (`/`), tìm kiếm & lọc danh mục, Chi tiết món (`/products/:id`), Giỏ hàng cục bộ (`/cart` bước thêm/xóa/tăng giảm món), Danh sách ngân hàng (`GET /api/v1/canteen/banks`).
   - **Protected (yêu cầu đăng nhập SĐT Zalo):** Đặt đơn (`POST /api/v1/orders`), Đơn của tôi (`/orders`, `/orders/:id`), toàn bộ phân hệ Căn tin số (`/canteen`, `/canteen/link-bank`, `/canteen/verify-otp`, `/canteen/unlink-otp`, `/canteen/limit-history`).
   - Khi người dùng chưa đăng nhập truy cập màn hình Protected hoặc bấm **"Đặt đơn"** tại Giỏ hàng, hiển thị giao diện **Đăng nhập bằng SĐT Zalo** gọn gàng (giữ nguyên giỏ hàng, đăng nhập xong đi tiếp mượt mà).
   - Tự động điền sẵn SĐT đã xác thực của phụ huynh vào biểu mẫu Liên kết ngân hàng (`BankLinkFormPage`) và cho phép đăng xuất/đổi SĐT tại màn hình Căn tin số.
3. **Cổng gửi SMS OTP chuẩn Production-Ready (`SmsGatewayInterface` — Strategy Pattern):**
   - Hỗ trợ 3 driver cấu hình qua `SMS_DRIVER`:
     - `log` (mặc định Local/Testing/CI — mô phỏng gửi SMS, che SĐT trong log, không bao giờ log plaintext OTP ở `production`).
     - `esms` (`EsmsSmsGateway` — eSMS.vn REST API chuẩn cho SĐT Việt Nam, hỗ trợ cả `ESMS_SANDBOX=true` và gửi SMS thật khi điền `ESMS_API_KEY` / `ESMS_SECRET_KEY`).
     - `twilio` (`TwilioSmsGateway` — Twilio REST API chuẩn quốc tế, tự chuyển đổi E.164 `+84...`).
   - Tích hợp `SmsGatewayInterface` vào cả 4 điểm phát hành OTP của `BankLinkingService` (`initiateLink`, `resendOtp`, `initiateUnlink`, `resendUnlinkOtp`), lưu vết metadata giao nhận (`delivery_provider`, `delivery_status`, `delivery_reference`) và rollback sạch transaction nếu cổng SMS thất bại (`503 SMS_DELIVERY_FAILED`).

### 1.2. Ngoài phạm vi (Out of Scope cho Slice P1)
- Không triển khai `SandboxBankingGateway` / `VNPay Sandbox` / đối soát (thuộc **Slice P2**).
- Không triển khai POS Staff Terminal / tồn kho / RBAC 4 vai trò `student` (thuộc **Slice P3**).
- Không triển khai Notification Center / Zalo OA Push (thuộc **Slice P4**).
- Không tự ý commit API key thật, không `git push`, không deploy.

---

## 2. Thiết kế Cơ sở Dữ liệu (2 Migrations Additive)

### 2.1. `2026_11_25_300001_add_phone_to_users_table.php`
Bổ sung cột vào bảng `users`:
- `phone VARCHAR(20) NULL UNIQUE` (lưu chuẩn SĐT VN 10 chữ số `0xxxxxxxxx`, vd `0901234567`).
- `phone_verified_at TIMESTAMP NULL`.
- Cập nhật `StudentCardSeeder`: gán `phone = '0901234567'` và `phone_verified_at = now()` cho tài khoản mẫu `demo_customer1` (idempotent, chỉ cập nhật khi `phone` đang `null`) để test trực tiếp luồng đăng nhập SĐT khớp ngay thẻ `F156BA`.

### 2.2. `2026_11_25_300002_add_sms_delivery_metadata_to_otp_verifications_table.php`
Bổ sung cột audit giao nhận SMS vào bảng `otp_verifications` (**tuyệt đối không lưu plaintext OTP**):
- `delivery_provider VARCHAR(32) NULL` (`log`, `esms`, `twilio`).
- `delivery_status VARCHAR(24) NULL` (`SIMULATED`, `SENT`, `FAILED`).
- `delivery_reference VARCHAR(120) NULL` (mã tin nhắn / SID do nhà cung cấp trả về).

---

## 3. Thiết kế Backend (`api/`)

### 3.1. Chuẩn hóa Số điện thoại Việt Nam (`App\Services\PhoneNormalizer`)
- `normalizeVietnamMobile(string $raw): ?string`:
  - Loại bỏ khoảng trắng, dấu gạch ngang, dấu chấm.
  - Chuyển `+84xxxxxxxxx` hoặc `84xxxxxxxxx` → `0xxxxxxxxx`.
  - Xác thực chặt regex `^0[35789]\d{8}$`; nếu không khớp trả `null`.
- `toE164(string $normalizedVietnamPhone): string`:
  - Chuyển `0901234567` → `+84901234567` (phục vụ `TwilioSmsGateway`).

### 3.2. Xác thực SĐT Zalo (`ZaloIdentityService` & `AuthController`)

#### A. Mở rộng `ZaloIdentityService`
- Thêm `resolvePhoneNumber(string $accessToken, string $phoneToken): string`:
  - Kiểm tra `config('services.zalo.app_secret')` không rỗng; nếu rỗng ném `ZaloAuthException('Zalo app secret is not configured.', 503)`.
  - Gọi `Http::timeout(10)->withHeaders(['access_token' => $accessToken, 'code' => $phoneToken, 'secret_key' => $secret])->get('https://graph.zalo.me/v2.0/me/info')`.
  - Xác minh `error === 0` và `data.number` tồn tại; chuẩn hóa bằng `PhoneNormalizer::normalizeVietnamMobile()`. Nếu token hết hạn (2 phút) / sai → ném `ZaloAuthException('Mã xác thực số điện thoại Zalo không hợp lệ hoặc đã hết hạn.', 401)`.

#### B. Endpoints mới trong `routes/api.php`
| Method | Endpoint | Middleware | Mô tả & Quy tắc |
|---|---|---|---|
| `POST` | `/api/v1/auth/zalo-phone` | `throttle:20,1` | Body: `{ access_token, phone_token, zalo_user_id?, name? }`. Đổi SĐT từ Zalo Graph API phía server, đối khớp/tạo `User` trong `DB::transaction`, trả `{ data: { token, user } }`. |
| `POST` | `/api/v1/auth/demo-phone` | `throttle:20,1` | Chỉ bật khi `APP_ENV in [local, testing]` và `DEMO_LOGIN_ENABLED=true` (ngoài ra trả `403`). Body: `{ phone_number, name? }`. Trả `{ data: { token, user } }`. |
| `POST` | `/api/v1/auth/logout` | `auth:sanctum` | Thu hồi Sanctum token hiện tại, trả `200 OK` `{ "message": "Đã đăng xuất." }`. |

#### C. Quy tắc Liên kết Tài khoản (Account Resolution) an toàn trong `loginZaloPhone`
Trong `DB::transaction()`:
1. Lấy `$phone = $zaloIdentity->resolvePhoneNumber($accessToken, $phoneToken)`.
2. Thử lấy `$zaloProfile` qua `verifyAccessToken($accessToken)` (fallback `$validated['zalo_user_id']` nếu token chỉ có scope SĐT).
3. Tìm `$userByPhone = User::where('phone', $phone)->lockForUpdate()->first()` và (nếu có `$resolvedZaloId`) `$userByZalo = User::where('zalo_user_id', $resolvedZaloId)->lockForUpdate()->first()`.
4. **Chống xung đột tài khoản (`409 ACCOUNT_CONFLICT`):** Nếu cả `$userByPhone` và `$userByZalo` cùng tồn tại nhưng khác `id` (tức Zalo ID đang gắn với SĐT khác) → từ chối với `409 Conflict` (`error_code: ACCOUNT_CONFLICT`) thay vì ghi đè chéo dữ liệu!
5. Nếu tìm thấy `$userByPhone`: giữ nguyên `role` hiện có (không hạ quyền `staff`/`admin`), cập nhật `zalo_user_id` (nếu đang null), `phone_verified_at = now()`, cập nhật `name` nếu trước đó đang là tên mặc định.
6. Nếu chỉ tìm thấy `$userByZalo`: gán `phone = $phone`, `phone_verified_at = now()`.
7. Nếu chưa có tài khoản nào: tạo `User` mới (`name`, `phone`, `phone_verified_at = now()`, `zalo_user_id`, `role = UserRole::CUSTOMER`).
8. Cập nhật `UserResource` (additive): thêm `phone` (trả cho chính chủ đã đăng nhập để tự điền form liên kết ngân hàng), `phone_masked` (`090****567`), `has_verified_phone` (`bool`).

### 3.3. Kiến trúc Cổng SMS OTP (`SmsGatewayInterface`)

#### A. Contract & Config (`api/config/services.php`)
```php
'sms' => [
    'driver' => env('SMS_DRIVER', 'log'), // log | esms | twilio
    'brandname' => env('SMS_BRANDNAME', 'SmartCanteen'),
    'esms' => [
        'endpoint' => env('ESMS_ENDPOINT', 'https://rest.esms.vn/MainService.svc/json/SendMultipleMessage_V4_post_json/'),
        'api_key' => env('ESMS_API_KEY', ''),
        'secret_key' => env('ESMS_SECRET_KEY', ''),
        'sms_type' => (int) env('ESMS_SMS_TYPE', 2),
        'sandbox' => filter_var(env('ESMS_SANDBOX', true), FILTER_VALIDATE_BOOL),
    ],
    'twilio' => [
        'account_sid' => env('TWILIO_ACCOUNT_SID', ''),
        'auth_token' => env('TWILIO_AUTH_TOKEN', ''),
        'from' => env('TWILIO_FROM', ''),
    ],
],
```

#### B. 3 Driver & Hành vi an toàn
1. **`LogSmsGateway` (`log`):**
   - Không gọi mạng ngoài; trả `['provider' => 'log', 'status' => 'SIMULATED', 'reference' => 'LOG-SMS-...']`.
   - Ghi `Log::info(...)` với `phone_masked` (không bao giờ ghi SĐT nguyên bản hay mã OTP khi `production`; ở `local` chỉ ghi thông điệp trạng thái).
2. **`EsmsSmsGateway` (`esms`):**
   - Fail-closed: thiếu `api_key` hoặc `secret_key` → ném `SmsDeliveryException`.
   - Nội dung tin nhắn chuẩn ngắn gọn không dấu (tránh chia mảnh GSM-7):
     - `BANK_LINK`: `Ma OTP lien ket ngan hang Smart Canteen cua ban la {CODE}. Hieu luc {MINUTES} phut. Khong chia se ma nay.`
     - `BANK_UNLINK`: `Ma OTP huy lien ket ngan hang Smart Canteen cua ban la {CODE}. Hieu luc {MINUTES} phut. Khong chia se ma nay.`
   - Timeout `8s`, retry `1` lần trên lỗi kết nối/HTTP 5xx; kiểm tra `CodeResult === '100'`, trả `SMSID` làm `reference`.
3. **`TwilioSmsGateway` (`twilio`):**
   - Chuyển SĐT VN sang E.164 (`+84...`), Basic Auth `account_sid:auth_token`, timeout `8s`, retry `1` lần trên lỗi kết nối/5xx, trả `sid` làm `reference`.
4. **Tích hợp vào `BankLinkingService`:**
   - Gọi `$delivery = $this->smsGateway->sendOtp($phoneNumber, $otpCode, $purpose, $ttlSeconds)` trong 4 luồng `initiateLink`, `resendOtp`, `initiateUnlink`, `resendUnlinkOtp`.
   - Nếu `SmsDeliveryException` xảy ra → transaction rollback toàn bộ (không lưu `PENDING_OTP` rác, không tăng `resend_count`), Controller trả `503 Service Unavailable` (`error_code: 'SMS_DELIVERY_FAILED'`).
   - Lưu `delivery_provider`, `delivery_status`, `delivery_reference` vào `otp_verifications`.
   - Response JSON `data.otp` bổ sung additive: `delivery_channel: 'SMS'`, `delivery_provider: $delivery['provider']`.

---

## 4. Thiết kế Frontend Mini App (`miniapp/`, tuân thủ `DESIGN.md`)

### 4.1. Kiến trúc Lazy Auth (Không ép đăng nhập khi xem Thực đơn)
- **`AuthBootstrap` trong `layout.tsx`:**
  - Khi mở app: chỉ kiểm tra nếu đã có Bearer token hợp lệ (`getAuthToken()`) thì gọi `fetchCurrentUser()` để khôi phục `currentUserAtom`.
  - **Không** gọi `ensureAuthenticated()` tự động trên `/` hay `/products/:id`! Khách vãng lai lướt thực đơn, tìm kiếm, xem chi tiết món và thêm món vào giỏ hàng hoàn toàn tức thì (0 request auth không cần thiết).
- **Ma trận bảo vệ tuyến & thao tác:**
  - **Public:** `/` (`MenuPage`), `/products/:id` (`ProductDetailPage`), `/cart` (`CartPage` — bước xem/chỉnh giỏ hàng).
  - **Protected (hiển thị `ZaloPhoneLoginPanel` inline khi `currentUser === null`):** `/canteen`, `/canteen/link-bank`, `/canteen/verify-otp`, `/canteen/unlink-otp`, `/canteen/limit-history`, `/orders`, `/orders/:id`.
  - **Protected action tại `/cart`:** Khi `currentUser === null`, phần chọn Thẻ học sinh hiện gợi ý ngắn "Đăng nhập SĐT Zalo để dùng Thẻ học sinh", và khi bấm **"Đặt đơn"** sẽ mở **`ZaloPhoneLoginModal`** (accessible, có `useFocusTrap`, Escape dismiss) ngay trên trang Giỏ hàng — đăng nhập xong đóng modal và cho tiếp tục đặt đơn mà không mất món đã chọn!

### 4.2. Component `ZaloPhoneLoginCard` & `ZaloPhoneLoginModal`
- **Nút chính (Zalo Mobile):** **"Đăng nhập bằng số điện thoại Zalo"**
  - Gọi `zmp-sdk` `getAccessToken({})` + `getPhoneNumber({})`.
  - Lấy `token` trả về gửi lên `POST /api/v1/auth/zalo-phone`.
  - Nếu người dùng từ chối cấp quyền hoặc chạy trên trình duyệt web ngoài Zalo (`@zaloOnly` lỗi) → hiển thị thông báo tiếng Việt thân thiện, không crash app.
- **Chế độ Demo SĐT cho Local/Dev/Testing (ẩn tự động khi `VITE_DEMO_PHONE_LOGIN === "false"` hoặc backend trả `403`):**
  - Ô nhập SĐT VN (`0901234567` kèm nút điền nhanh tài khoản phụ huynh mẫu `0901234567` có sẵn thẻ `F156BA`) gọi `POST /api/v1/auth/demo-phone`.
- **Tích hợp mượt vào `BankLinkFormPage` & `CanteenHubPage`:**
  - `BankLinkFormPage`: khi mở form mới mà `currentUser.phone` có giá trị, tự động điền sẵn `phone_number = currentUser.phone` (vẫn cho phép chỉnh sửa nếu cần).
  - `CanteenHubPage`: hiển thị dải thông tin phụ huynh (`currentUser.name` · `currentUser.phone_masked`) kèm nút **"Đăng xuất"** gọn gàng (gọi `POST /api/v1/auth/logout`, xóa token & React Query cache cá nhân).

---

## 5. Kế hoạch Kiểm chứng & Tiêu chí Chấp nhận Slice P1

| Hạng mục | Test Suite / Lệnh | Tiêu chí đạt |
|---|---|---|
| Zalo Phone Auth + Account Resolution + Logout | `api/tests/Feature/Auth/ZaloPhoneAuthTest.php` | Đổi `phone_token` qua `Http::fake` Zalo Graph API (`8490... → 090...`), khớp user cũ theo SĐT, bảo toàn role `admin`/`staff`, chặn conflict `409`, thiếu `ZALO_APP_SECRET` fail-closed, `/auth/demo-phone` chỉ bật ở `local`/`testing` & `403` ở `production`, `/auth/logout` thu hồi token |
| SMS OTP Gateway (3 drivers + Rollback on failure) | `api/tests/Feature/Canteen/SmsOtpGatewayTest.php` | `LogSmsGateway` (mask SĐT, không lộ OTP ở production), `EsmsSmsGateway` (`Http::fake` `CodeResult=100` & lỗi -> `503` rollback sạch không tạo `bank_link`/không mất lượt resend), `TwilioSmsGateway` (`Http::fake` `+84...`), ghi đủ `delivery_provider/status/reference` |
| Chuẩn hóa SĐT & Lazy Auth Frontend | `miniapp/src/features/auth/phone.test.ts` + `vitest run` + `tsc --noEmit` + `vite build` | 100% pass unit tests, 0 lỗi TypeScript, build sạch `www/` |
| Hồi quy toàn hệ thống | `vendor/bin/pint --test`, `php artisan test --compact`, `php artisan migrate --force --seed` | 100% test cũ (91) + test mới đều PASS trên PostgreSQL 17 trong Docker |

---

## 6. Rủi ro & Biện pháp Giảm thiểu
- **Rủi ro vỡ test cũ khi thêm cột `phone` hoặc thay đổi `AuthBootstrap`:** Cột `phone`/`phone_verified_at` là `nullable`, `POST /api/v1/auth/zalo` & `/auth/demo` giữ nguyên hoạt động; toàn bộ 91 test cũ phải xanh 100%.
- **Rủi ro lộ SĐT hoặc OTP plaintext qua log:** `LogSmsGateway` chỉ log `phone_masked` và không bao giờ ghi `$otpCode` vào log hệ thống; `UserResource` không lộ `phone` trên các endpoint Admin/Order của người khác.
- **Rủi ro giữ khóa DB lâu khi gọi HTTP SMS:** Gọi `SmsGatewayInterface` với timeout ngắn `8s` và retry tối đa `1` lần (`200ms` backoff), và chỉ khi gửi thành công mới commit bản ghi OTP/BankLink để đảm bảo trải nghiệm phụ huynh không bị mất lượt resend hay kẹt trạng thái.
