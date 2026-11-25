# Kế hoạch Triển khai — Trục 3 · Slice P1: Xác thực bằng Số điện thoại Zalo (Lazy Auth) & Cổng SMS OTP Thực tế

- **Đặc tả đã phê duyệt:** `docs/superpowers/specs/2026-11-25-p1-zalo-phone-auth-sms-otp-design.md`
- **Nhánh làm việc:** `feat/p1-zalo-phone-auth-sms-otp` (độc lập trên `root`, `api/`, `miniapp/`)
- **Chiến lược kiểm chứng:** Feature + Unit Tests trên PostgreSQL 17 thật qua Docker `sc-p1` (`Http::fake` cho Zalo Graph API, eSMS, Twilio), Laravel Pint, Vitest, TypeScript, Vite Build, và Independent Code Review.

## Task 1 — Khởi tạo nhánh & phê duyệt đặc tả
- Đánh dấu đặc tả Slice P1 `APPROVED (2026-11-25)`.
- Tạo nhánh `feat/p1-zalo-phone-auth-sms-otp` từ `main` trên cả 3 repo (`root`, `api/`, `miniapp/`).

## Task 2 — Backend Migrations & Zalo Phone Authentication
1. Migrations additive:
   - `2026_11_25_300001_add_phone_to_users_table.php` (`phone VARCHAR(20) NULL UNIQUE`, `phone_verified_at TIMESTAMP NULL`).
   - `2026_11_25_300002_add_sms_delivery_metadata_to_otp_verifications_table.php` (`delivery_provider`, `delivery_status`, `delivery_reference`).
2. Cập nhật Models & Seeder:
   - `User.php`, `UserResource.php` (bổ sung `phone`, `phone_masked`, `has_verified_phone`), `OtpVerification.php`, `StudentCardSeeder.php` (gán `phone = 0901234567` cho `demo_customer1`).
3. `PhoneNormalizer.php`, `ZaloAuthException.php` (thêm `statusCode`), `ZaloIdentityService::resolvePhoneNumber()`.
4. `AuthController.php` & `routes/api.php`:
   - `POST /api/v1/auth/zalo-phone` (`throttle:20,1`, account resolution trong `DB::transaction`, chống `409 ACCOUNT_CONFLICT`, giữ role `staff`/`admin`).
   - `POST /api/v1/auth/demo-phone` (`throttle:20,1`, fail-closed `403` ngoài `local`/`testing` hoặc khi `DEMO_LOGIN_ENABLED=false`).
   - `POST /api/v1/auth/logout` (`auth:sanctum`).
5. Test `api/tests/Feature/Auth/ZaloPhoneAuthTest.php`.

## Task 3 — Backend SMS OTP Gateway (`SmsGatewayInterface` — 3 Drivers)
1. `App\Contracts\SmsGatewayInterface` & `App\Exceptions\SmsDeliveryException`.
2. 3 Drivers:
   - `App\Services\Sms\LogSmsGateway` (mask SĐT, không bao giờ log plaintext OTP).
   - `App\Services\Sms\EsmsSmsGateway` (`eSMS.vn` REST API, timeout 8s, retry 1 lần, fail-closed khi thiếu key).
   - `App\Services\Sms\TwilioSmsGateway` (Twilio REST API, E.164 `+84...`, timeout 8s, retry 1 lần).
3. Đăng ký trong `config/services.php`, `AppServiceProvider.php`, cập nhật `api/.env.example`.
4. Tích hợp `SmsGatewayInterface` vào 4 điểm phát hành OTP của `BankLinkingService` (`initiateLink`, `resendOtp`, `initiateUnlink`, `resendUnlinkOtp`), lưu `delivery_provider/status/reference`, rollback sạch khi `SmsDeliveryException` (`503 SMS_DELIVERY_FAILED` trong `CanteenBankLinkController`).
5. Test `api/tests/Feature/Canteen/SmsOtpGatewayTest.php`.

## Task 4 — Frontend Mini App: Lazy Auth (Public vs Protected) & Zalo Phone Login UI
1. `miniapp/src/features/auth/phone.ts` + `phone.test.ts`: chuẩn hóa & kiểm tra SĐT VN, helper kiểm tra public/protected route.
2. `miniapp/src/features/auth/api.ts` & `store.ts`:
   - Thêm `AuthUser.phone`, `phone_masked`, `has_verified_phone`, `authHydratedAtom`.
   - Thêm `hydrateAuthSession()`, `loginWithZaloPhone()`, `loginWithDemoPhone()`, `logoutSession()`.
   - Cập nhật `layout.tsx` (`AuthBootstrap`) chỉ khôi phục phiên có sẵn, **không** ép login tự động khi mở `/` hay `/products/:id`.
3. `miniapp/src/components/zalo-phone-login.tsx`: cung cấp `ZaloPhoneLoginPanel` (inline cho các trang Protected) & `ZaloPhoneLoginModal` (accessible với `useFocusTrap`, cho nút Đặt đơn tại `/cart`).
4. Cập nhật các trang:
   - `canteen.tsx` (hiển thị `ZaloPhoneLoginPanel` khi chưa đăng nhập; hiển thị dải phụ huynh + nút **Đăng xuất** khi đã đăng nhập).
   - `orders.tsx`, `order-detail.tsx`, `limit-history.tsx`, `bank-link-form.tsx` (hiển thị `ZaloPhoneLoginPanel` nếu chưa đăng nhập; auto-fill `phone_number` từ `currentUser.phone` ở `BankLinkFormPage`).
   - `cart.tsx` (giỏ hàng Public tự do; chỉ bật `ZaloPhoneLoginModal` khi bấm Đặt đơn hoặc chọn dùng Thẻ học sinh lúc chưa đăng nhập).

## Task 5 — Cập nhật tài liệu & Code Review độc lập
- Cập nhật `docs/api-contract.md`, `docs/database.md`, `docs/PROJECT_STATE.md`.
- Chạy Docker `sc-p1` (Pint, PHPUnit, migrations/seeders) & Frontend (`npm test`, `tsc --noEmit`, `vite build`).
- Gọi `Slice P1 Code Reviewer` kiểm tra độc lập toàn bộ diff của 3 repo.

## Task 6 — Khắc phục Review, Kiểm chứng cuối, Commit & Handoff
- Sửa các phát hiện Critical/Important, chạy lại toàn bộ cổng chất lượng sau lần sửa cuối, dọn Docker `sc-p1`, commit sạch trên `feat/p1-zalo-phone-auth-sms-otp` ở 3 repo và viết Báo cáo Mục 17.
