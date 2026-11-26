# Cẩm nang Toàn cảnh Hệ thống & Hướng dẫn Trình bày Review Code / Test (`docs/architecture/system-deep-dive-presentation-guide.md`)

Tài liệu này được thiết kế riêng cho **buổi họp Review Code & Review Bài Test cùng Giám đốc và 2 anh Developer**, giúp bạn nắm chắc bức tranh tổng thể từ góc nhìn nghiệp vụ lẫn chiều sâu kỹ thuật, kèm bản đồ mã nguồn, bản đồ bài test và bộ câu hỏi phản biện thường gặp.

---

## Phần 1 — Góc nhìn Dành cho Giám đốc (Executive Summary — Trình bày trong 3 phút)

### 1.1. Dự án đã giải quyết trọn vẹn bài toán gì?
Từ yêu cầu ban đầu (MVP đặt món căn tin) mở rộng theo **10/10 phân hệ nghiệp vụ trong `FEATURES.md` §9**, hệ thống **Smart Canteen & Căn tin Số** hiện là một sản phẩm **Production-Ready** chạy độc lập 100% (Greenfield, không ảnh hưởng dữ liệu thật) gồm **3 giao diện** dùng chung 1 lõi nghiệp vụ Backend:

1. **Zalo Mini App (`miniapp/` — 11 màn hình, 5 tab đáy):**
   - **Phụ huynh:** Duyệt thực đơn tự do trước khi đăng nhập (Lazy Auth), đăng nhập 1 chạm bằng SĐT Zalo, quản lý nhiều thẻ con em, liên kết/hủy ngân hàng trích nợ tự động bằng OTP SMS 6 số, kiểm soát hạn mức chi tiêu từng ngày, đặt món Online (Thẻ / VNPay / Tiền mặt) và nhận thông báo giao dịch tức thời.
   - **Học sinh:** Đăng nhập xem thẻ, kiểm tra hạn mức còn lại trong ngày, tự đặt món trong hạn mức nhưng **tuyệt đối không thể tự sửa hạn mức hay can thiệp tài khoản ngân hàng**.
2. **Cổng Quản trị Admin (`api/` — `/admin/*`):**
   - Thống kê doanh thu hôm nay, điều phối đơn bếp, quản lý Thực đơn & **Tồn kho**, CRUD **Học sinh & Cấp phát/Khóa/Thu hồi thẻ**, **Đối soát giao dịch ngân hàng cuối ngày**, cấu hình Sandbox và **Nhật ký Thông báo**.
3. **Quầy Thu ngân POS (`api/` — `/admin/pos`):**
   - Giao diện 2 cột tốc độ cao cho nhân viên căn tin: gõ/quét mã thẻ học sinh → hiện ngay hạn mức còn lại → chọn món → quẹt thẻ hoặc thu tiền mặt → **in biên lai nhiệt**.

### 1.2. Giá trị Nổi bật để Nhấn mạnh với Giám đốc
- **100% Phủ đầy 10/10 Module Căn tin Số** (Authentication, Student, Card, Banking, OTP, Spending Limit, Order + POS + Inventory, Payment Ledger, Reconciliation, Notification).
- **An toàn tiền bạc & dữ liệu cá nhân (PII):** Số tài khoản ngân hàng & CCCD mã hóa **AES-256** trong CSDL, OTP băm **bcrypt**, mọi màn hình/API/log đều che mờ (`****5678`, `090***4567`). Khi ngân hàng lỗi hoặc thiếu tiền, đơn hàng/tồn kho/hạn mức được rollback sạch nhưng sổ cái `payment_transactions` **vẫn giữ bằng chứng** để kế toán đối soát.
- **Sẵn sàng cắm đối tác thật không cần sửa lõi:** 3 Adapter độc lập (`SmsGatewayInterface`, `BankingGatewayInterface`, `PushNotificationChannelInterface`) chỉ cần đổi biến `.env` là chuyển từ chế độ Sandbox/Log sang eSMS/Twilio, EPAY/VNPay, Zalo OA/ZNS thật.

---

## Phần 2 — Góc nhìn Chuyên sâu Dành cho 2 Anh Developer (Code Walkthrough — 10 phút)

### 2.1. Cấu trúc Multi-Repo & Vận hành
```text
TestPart2/                  ← Workspace điều phối + toàn bộ docs/ (spec, plan, audit, runbook)
├── api/                    ← Git repo độc lập: Laravel 13 + PHP 8.4 + PostgreSQL 17 + Docker + CI
│   ├── app/
│   │   ├── Contracts/      ← 3 Strategy Interfaces (Banking, SMS, Push Notification)
│   │   ├── Enums/          ← 10 Backed Enums (OrderStatus, UserRole, PaymentMethod, ...)
│   │   ├── Exceptions/     ← Domain Exceptions có error_code rõ ràng (InsufficientStock, ...)
│   │   ├── Http/
│   │   │   ├── Controllers/Api/V1/  ← 12 REST Controllers (Auth, Menu, Order, Canteen, Webhook, VNPay, Notifications)
│   │   │   ├── Controllers/Admin/   ← 10 Blade/POS Controllers (Dashboard, POS, Orders, Products, Students, Cards, Reconciliation, Notifications)
│   │   │   ├── Middleware/          ← EnsureUserHasRole (4 roles), SecurityHeadersMiddleware
│   │   │   └── Requests/ & Resources/
│   │   ├── Models/         ← 18 Eloquent Models (encrypted casts, Hidden PII attributes)
│   │   └── Services/       ← Lõi nghiệp vụ tập trung (API & Blade gọi chung 100%)
│   │       ├── OrderService.php              ← Trái tim hệ thống: Đặt đơn, POS, Trừ/Hoàn kho & Hạn mức & Tiền
│   │       ├── BankLinkingService.php        ← Link/Unlink Ngân hàng + OTP + SMS rollback
│   │       ├── SpendingLimitService.php      ← Hạn mức ngày + lazy reset Asia/Ho_Chi_Minh + audit log
│   │       ├── StudentCardService.php        ← Quyền sở hữu thẻ (parent vs student) + thu hồi thẻ
│   │       ├── PaymentLedgerService.php      ← Sổ cái bất biến (ghi bền vững cả FAILED/TIMEOUT ngoài rollback)
│   │       ├── ReconciliationService.php     ← Đối soát 5 trạng thái idempotent theo ngày
│   │       ├── NotificationService.php       ← 7 sự kiện afterCommit + lọc PII + chống Open Redirect
│   │       ├── DataMaskingService.php        ← Che mờ STK / CCCD / SĐT thống nhất
│   │       ├── ZaloIdentityService.php       ← Xác minh access_token + phone_token với graph.zalo.me
│   │       ├── Banking/ · Sms/ · Payment/ · Notification/  ← Các Adapter cụ thể
│   └── database/migrations/ ← 27 migrations tuần tự, additive, không destructive
└── miniapp/                ← Git repo độc lập: React 18 + TS + ZaUI + Jotai + TanStack Query + Docker + CI
    └── src/
        ├── lib/            ← api-client.ts (in-memory token, ApiError), format.ts, use-focus-trap.ts
        ├── features/       ← auth/ · cart/ · menu/ · orders/ · canteen/ · notifications/
        ├── components/     ← bottom-nav.tsx (5 tabs + aria-live), student-card-view.tsx, otp-code-input.tsx, ...
        └── pages/          ← 11 trang ứng dụng (Lazy Auth + 4 states Loading/Error/Empty/Success)
```

### 2.2. Giải phẫu 5 Cơ chế Kỹ thuật Khó nhất (Điểm "ăn tiền" khi Review Code)

#### Cơ chế 1 — Chống Deadlock & Chống Oversell trong `OrderService`
Khi 2 người cùng mua 1 món cuối cùng hoặc cùng quẹt 1 thẻ học sinh song song, hệ thống bảo vệ bằng **Thứ tự Khóa dòng Cố định (`SELECT ... FOR UPDATE`)** trong `DB::transaction()`:
1. `student_cards` → 2. `spending_limits` (kèm lazy reset ngày `Asia/Ho_Chi_Minh`) → 3. `sandbox_bank_accounts` → 4. `products` (**luôn `ORDER BY id ASC`** để loại bỏ AB-BA deadlock khi 2 đơn đặt cùng 2 món theo thứ tự ngược nhau!) → 5. `orders` → 6. `payment_transactions`.
- **Hủy đơn (`CANCELLED`):** Mở đúng thứ tự khóa đó → hoàn `stock_quantity` → hoàn `spent_today` (nếu cùng ngày) → hoàn số dư `sandbox_bank_accounts` → ghi dòng `REFUND` vào `payment_transactions` → nâng `orders.version + 1` → bắn thông báo trong `DB::afterCommit()`.

#### Cơ chế 2 — Rollback Nghiệp vụ nhưng Giữ Bằng chứng Kế toán (`PaymentLedgerService`)
- Một câu hỏi kinh điển khi review thanh toán: *"Nếu tài khoản ngân hàng không đủ tiền hoặc gateway timeout, ta phải rollback đơn hàng & tồn kho, vậy làm sao bảng `payment_transactions` không bị rollback mất dấu?"*
- **Cách hệ thống xử lý (`OrderService::createOrder` & `createPosPurchase`):**
  - Toàn bộ bước tạo đơn + trừ kho + trừ hạn mức + gọi `BankingGatewayInterface::charge()` chạy bên trong `DB::transaction()`.
  - Khi ngân hàng ném `BankingGatewayException` (`INSUFFICIENT_BANK_BALANCE` / `BANK_DECLINED` / `GATEWAY_TIMEOUT`), Laravel **rollback 100%** transaction (không tạo đơn rỗng, không trừ kho, không trừ hạn mức).
  - Khối `catch (BankingGatewayException)` nằm **BÊN NGOÀI** `DB::transaction()` ngay lập tức ghi 1 bản ghi bất biến (`FAILED` hoặc `TIMEOUT` kèm `gateway_txn_ref`) qua `PaymentLedgerService` và gửi thông báo `PAYMENT_FAILED` cho phụ huynh!

#### Cơ chế 3 — Idempotency + Optimistic Locking
- **Chống tạo đơn trùng (`Idempotency-Key`):** `UNIQUE (user_id, idempotency_key)` + `payload_hash` (SHA-256). Cùng key + cùng payload → trả `200 OK` đơn cũ; cùng key + đổi payload → trả `409 Conflict`.
- **Chống nhân viên bếp ghi đè trạng thái (`version`):** `PATCH /admin/orders/{id}/status` yêu cầu gửi kèm `version`; nếu lệch trong `lockForUpdate()` → `409 Conflict`.
- **Chống ghi đè tồn kho Admin (`original_stock_quantity`):** Khi Admin sửa thông tin món ăn lúc POS vừa bán trừ kho, `ProductManagementController::update()` kiểm tra xem Admin có thực sự đổi ô tồn kho hay không trước khi ghi đè.

#### Cơ chế 4 — Bảo mật Xác thực, Chữ ký số & PII
- **Zalo Phone Auth (`ZaloIdentityService`):** Server không bao giờ tin `zalo_user_id` hay `phone` do client tự gửi; luôn xác minh `access_token` kèm `appsecret_proof = HMAC-SHA256(access_token, ZALO_APP_SECRET)` và đổi `phone_token` trực tiếp phía server.
- **Webhook Ngân hàng (`BankingWebhookController`):** Kiểm tra `X-Gateway-Timestamp` (chặn lệch quá `±300s`), so khớp chữ ký `HMAC-SHA256(timestamp + "." + rawBody, secret)` bằng `hash_equals()`, và chống replay bằng `UNIQUE (provider, event_id)` trên bảng `gateway_webhook_events`.
- **VNPay (`VnpayGatewayService`):** Ký `HMAC-SHA512` chuẩn v2.1.0, IPN kiểm tra chặt `vnp_TmnCode`, `vnp_Amount === order.total_amount * 100`, `vnp_TransactionStatus`, xử lý idempotent, và **fail-closed** nếu production dùng khóa mặc định.
- **Thông báo (`NotificationService`):** Chặn Open Redirect (`isSafeInternalRoute()` chặn `https://`, `//`, `/\`, `javascript:`), đệ quy xóa khóa PII khỏi `meta`, và bọc `try/catch` quanh Push Channel để lỗi mạng Zalo OA/ZNS không bao giờ làm hỏng giao dịch tiền.

#### Cơ chế 5 — Kiến trúc Frontend Mini App (`miniapp/`)
- **Lazy Auth:** Token Sanctum chỉ lưu trong RAM (không dùng `localStorage`/`cookie` vốn hay bị hạn chế trên WebView Zalo). Người dùng duyệt Menu & Giỏ hàng tự do; chỉ bật `ZaloPhoneLoginModal` khi chạm vào tính năng cá nhân/tài chính.
- **State Separation:** Jotai quản lý state cục bộ đồng bộ (`currentUserAtom`, `cartItemsAtom`, `selectedCardIdAtom`, `bankLinkDraftAtom`); TanStack Query quản lý toàn bộ server state kèm polling thông minh (`7s` chi tiết đơn, `10s` lịch sử đơn, `20s` badge thông báo chưa đọc) và tự động xóa sạch cache khi đăng xuất hoặc đổi tài khoản.

---

## Phần 3 — Bản đồ Bài Test để Review Nhanh cùng 2 Anh Developer

Toàn bộ bài test chạy trên **PostgreSQL 17 thật** (không dùng SQLite in-memory giả lập — nhờ đó kiểm chứng được cả `SELECT ... FOR UPDATE`, Partial Unique Index, `jsonb`, và `pg_indexes`!).

### 3.1. Nhóm Test Backend ưu tiên mở ra xem trong cuộc họp (`api/tests/`)
1. **`ProductionReadinessEndToEndTest.php`** — *Bài test quan trọng nhất để mở đầu tiên!* Chạy liền mạch 1 vòng từ Slice P1 → P2 → P3 → P4: đăng nhập SĐT → liên kết ngân hàng + SMS OTP → đặt hạn mức → mua Online trừ kho/hạn mức/tiền → kiểm tra thông báo & security headers → lỗi thiếu tiền ghi sổ cái → đối soát → hủy đơn hoàn tiền/kho → kiểm chứng không lộ PII.
2. **`OrderApiTest.php` & `ProductInventoryTest.php`** — Chứng minh server-side pricing, `Idempotency-Key` (`201/200/409`), bảo toàn giá lịch sử, trừ/hoàn kho nguyên tử và chống ghi đè kho Admin.
3. **`SandboxBankingPaymentTest.php` & `BankingWebhookTest.php` & `VnpaySandboxTest.php`** — Chứng minh rollback sạch + sổ cái bền vững khi `INSUFFICIENT_BANK_BALANCE`/`BANK_DECLINED`/`GATEWAY_TIMEOUT`, chữ ký Webhook HMAC-SHA256 + anti-replay, và VNPay HMAC-SHA512 IPN idempotent.
4. **`FourRoleRbacTest.php` & `StaffPosTerminalTest.php` & `AdminStudentCrudTest.php`** — Chứng minh ma trận phân quyền 4 vai trò (`admin`/`staff`/`parent`/`student`), POS quẹt thẻ & biên lai, và CRUD học sinh/thẻ.
5. **`NotificationModuleTest.php`** — Chứng minh 7 sự kiện thông báo, `ZaloOaPushChannel` + `ZaloZnsPushChannel` resilient khi provider trả HTTP 500, chặn `error: "invalid_token"`, chặn Open Redirect `/\evil.example`, và lọc sạch PII `meta`.
6. **`DemoShowcaseSeederTest.php`** — Chứng minh bộ seed demo idempotent, giữ thống kê hôm nay = `0`, và xác nhận đủ chỉ mục lọc Admin trong `pg_indexes`.

### 3.2. Nhóm Test Frontend (`miniapp/src/**/*.test.ts` — 11 files)
- Tập trung vào các hàm nghiệp vụ thuần túy & bảo mật: `cart-logic.test.ts`, `phone.test.ts`, `validation.test.ts`, `spending-limit.test.ts`, `payment.test.ts` (`isSafeHttpUrl`), `notifications/helpers.test.ts` (`isSafeInternalRoute`, `formatUnreadBadge`), `api-client.test.ts`, `use-focus-trap.test.ts`.

---

## Phần 4 — Bộ Câu hỏi Phản biện Thường gặp & Cách Trả lời Chắc tay (Q&A Cheat Sheet)

| Câu hỏi có thể gặp trong buổi họp | Câu trả lời ngắn gọn, đúng trọng tâm |
|---|---|
| **1. "Tại sao không dùng Queue (Redis/Horizon) cho Push Notification / SMS ngay bây giờ?"** | Hiện kiến trúc đã tách chuẩn qua `PushNotificationChannelInterface` & `SmsGatewayInterface` và chỉ chạy trong `DB::afterCommit()` + timeout HTTP ngắn (`5s`) + fail-safe `try/catch` nên không thể làm lỗi giao dịch. Khi lưu lượng lớn trên production, chỉ cần đổi `NotificationService::send()` sang dispatch Laravel Job (`ShouldQueue`) với Redis mà **không cần sửa bất kỳ dòng code nào** ở `OrderService`, `BankLinkingService` hay API contract. |
| **2. "Nếu 2 giao dịch cùng quẹt 1 thẻ hoặc cùng mua phần ăn cuối cùng trong 1 mili-giây thì sao?"** | Cả hai đều vào `DB::transaction()` và khóa `SELECT ... FOR UPDATE` trên PostgreSQL theo đúng thứ tự cố định `student_cards → spending_limits → sandbox_bank_accounts → products (id ASC)`. Giao dịch thứ 2 phải chờ giao dịch thứ 1 commit xong rồi đọc số dư/hạn mức/tồn kho mới nhất — nếu không đủ sẽ trả ngay `422` (`INSUFFICIENT_STOCK` / `DAILY_LIMIT_EXCEEDED` / `INSUFFICIENT_BANK_BALANCE`), đảm bảo tuyệt đối không bao giờ âm kho hay âm tiền. |
| **3. "Làm sao đảm bảo hệ thống test không 'ăn gian' bằng SQLite hay mock quá đà?"** | Toàn bộ bộ PHPUnit chạy trực tiếp trên **PostgreSQL 17 thật** trong Docker (`titkul_canteen_test`), kiểm chứng thực tế khóa ngoại `restrictOnDelete`, Partial Unique Index, `jsonb`, và khóa dòng. Ta chỉ dùng `Http::fake()` ở đúng ranh giới mạng ngoài (Zalo Graph API, eSMS/Twilio, Zalo OA/ZNS) để test deterministic và không phụ thuộc internet. |
| **4. "Khi triển khai thật cho nhà trường, cần làm gì để nối ngân hàng/SMS/Zalo thật?"** | Chỉ cần cấu hình biến môi trường trong `api/.env` (`ZALO_APP_ID/SECRET`, `SMS_DRIVER=esms|twilio`, `VNPAY_TMN_CODE/HASH_SECRET`, `NOTIFICATION_PUSH_DRIVER=zalo_oa|zalo_zns`) và viết thêm 1 class triển khai `BankingGatewayInterface` cho đối tác thu hộ thực tế (ví dụ EPAY production) rồi đăng ký trong `AppServiceProvider` — toàn bộ logic đơn hàng, hạn mức, đối soát, POS và Mini App giữ nguyên 100%. |
| **5. "Dữ liệu nhạy cảm của phụ huynh (STK, CCCD, SĐT) được bảo vệ ra sao?"** | STK và CCCD được mã hóa **AES-256** (`encrypted` cast) bằng `APP_KEY`, khai báo `#[Hidden]` trên Eloquent Model; OTP chỉ lưu mã băm `bcrypt`; tất cả API Resource, màn hình Admin Blade, POS Lookup, Notification `meta` và log hệ thống đều đi qua `DataMaskingService` để che mờ và chặn rò rỉ PII (đã có test tự động quét chuỗi nhạy cảm trong `ProductionReadinessEndToEndTest`). |
| **6. "Tại sao tách thành 2 sub-repo `api/` và `miniapp/`?"** | Để Frontend Mini App và Backend Laravel có vòng đời CI/CD, Docker image và triển khai độc lập (khi sửa UI Mini App và đẩy lên Zalo Testing không cần build/deploy lại Backend API và ngược lại), đồng thời workspace cha vẫn giữ thống nhất tài liệu thiết kế và hợp đồng API. |
