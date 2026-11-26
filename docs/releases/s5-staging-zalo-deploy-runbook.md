# Sổ tay Triển khai Lát cắt S5 — Staging HTTPS & Zalo Mini App Testing (`docs/releases/s5-staging-zalo-deploy-runbook.md`)

> **Trạng thái Cấp quyền Hiện tại:** `Deployment: NOT AUTHORIZED` (Agent không tự ý đẩy mã nguồn hay triển khai ra môi trường ngoài khi chưa có lệnh phê duyệt tường minh). Tài liệu này hướng dẫn quy trình chuẩn từng bước để đưa trọn bộ phiên bản **`v1.0.0-digital-canteen`** lên Staging HTTPS & Zalo Mini App Testing.

---

## 1. Kiến trúc Triển khai Demo / Staging

1. **Backend Laravel API & Admin/POS (`api/`):**
   - Khuyến nghị chạy qua Docker Production (`api/compose.prod.yaml` — PHP 8.4 FPM non-root `appuser` + Nginx unprivileged + PostgreSQL 17) kết hợp **Cloudflare Tunnel (`cloudflared`)** hoặc Reverse Proxy HTTPS có chứng chỉ SSL hợp lệ.
2. **Frontend Zalo Mini App (`miniapp/`):**
   - Build production (`npm run build`) và đẩy lên **Zalo Mini App Testing** (qua Zalo Mini App Studio / VS Code Extension / `zmp-cli`) để sinh mã QR Code / Deep Link kiểm thử trên điện thoại thật.

---

## 2. Danh mục Biến Môi trường Bắt buộc cho Staging / Production (`api/.env`)

```env
APP_ENV=staging
APP_DEBUG=false
APP_URL=https://<ten-mien-https-cua-ban>

# Bảo mật & CORS
CORS_ALLOWED_ORIGINS=https://h5.zdn.vn
DEMO_LOGIN_ENABLED=false
BANKING_EXPOSE_DEBUG_OTP=false

# Xác thực SĐT Zalo (Bắt buộc trên Zalo Mini App thật)
ZALO_APP_ID=<ZALO_MINI_APP_ID>
ZALO_APP_SECRET=<ZALO_APP_SECRET>

# Cổng SMS OTP (Chọn log khi staging chưa mua SMS; esms hoặc twilio khi thật)
SMS_DRIVER=log
# ESMS_API_KEY=...
# ESMS_SECRET_KEY=...

# Cổng Ngân hàng & VNPay Sandbox
BANKING_GATEWAY_DRIVER=sandbox
BANKING_WEBHOOK_SECRET=<CHUOI_BI_MAT_NGẪU_NHIEN_64_KY_TU>
VNPAY_TMN_CODE=<VNPAY_SANDBOX_TMN_CODE>
VNPAY_HASH_SECRET=<VNPAY_SANDBOX_HASH_SECRET>
VNPAY_RETURN_URL=https://<ten-mien-https-cua-ban>/api/v1/payments/vnpay/return

# Thông báo Push Zalo OA / ZNS (mặc định log; bật zalo_oa hoặc zalo_zns khi có OA Token)
NOTIFICATION_PUSH_DRIVER=log
# ZALO_OA_ACCESS_TOKEN=...
# ZALO_ZNS_ACCESS_TOKEN=...
# ZALO_ZNS_TEMPLATE_ID=...
```

> **Lưu ý an toàn (Fail-Closed):** Trên `APP_ENV=production`:
> - `POST /api/v1/auth/demo` và `POST /api/v1/auth/demo-phone` tự động bị khóa (`403`).
> - `debug_otp` không bao giờ được trả về trong JSON API.
> - `VnpayGatewayService` tự từ chối nếu phát hiện `VNPAY_HASH_SECRET` trống hoặc dùng khóa demo mặc định.

---

## 3. Quy trình Triển khai 4 Bước

### Bước 1 — Cấu hình Zalo Developer Center
1. Tạo / chọn Mini App trên [Zalo Mini App Developer Center](https://mini.zalo.me/developers/), lấy `ZALO_APP_ID` và `ZALO_APP_SECRET`.
2. Bật quyền `getPhoneNumber` (xin cấp SĐT người dùng) và thêm số điện thoại của **Giám đốc** + **Nhóm kiểm thử** vào mục **Testers** (**AC-15**).

### Bước 2 — Khởi chạy Backend HTTPS (`api/`)
```powershell
cd d:\Coding\TitKul_ProjectTest\TestPart2\api
docker compose -f compose.prod.yaml up -d --wait
docker compose -f compose.prod.yaml exec app php artisan migrate --force
# Nếu cần dữ liệu trình diễn:
docker compose -f compose.prod.yaml exec app php artisan db:seed --force

# Mở tunnel HTTPS (nếu chạy trên máy cá nhân):
cloudflared tunnel --url http://127.0.0.1:8000
```

### Bước 3 — Cấu hình & Deploy Zalo Mini App (`miniapp/`)
1. Cập nhật `miniapp/.env`:
   ```env
   VITE_API_BASE_URL=https://<ten-mien-https-cua-ban>/api/v1
   VITE_DEMO_PHONE_LOGIN=false
   ```
2. Build & deploy Testing:
   ```powershell
   cd d:\Coding\TitKul_ProjectTest\TestPart2\miniapp
   npm run build
   # Dùng Zalo Mini App VS Code Extension hoặc zmp-cli để Deploy Testing lấy mã QR
   ```

### Bước 4 — Checklist Nghiệm thu Trước khi Gửi Giám đốc (Pre-Flight)
- [ ] `GET https://<domain>/up` trả `200 OK` và header bảo mật (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`).
- [ ] `GET https://<domain>/api/v1/categories` từ mạng 4G trả JSON 6 danh mục (**AC-16**).
- [ ] Quét mã QR Zalo trên điện thoại: duyệt thực đơn công khai → xác thực SĐT Zalo thật → đặt thử 1 đơn → thấy đơn hiển thị trên `/admin` và quầy `/admin/pos` (**AC-14**, **AC-15**).
