# Sổ tay Triển khai Lát cắt S5 — Staging HTTPS & Zalo Mini App Testing (`docs/releases/s5-staging-zalo-deploy-runbook.md`)

> **Trạng thái Cấp quyền Hiện tại:** `Deployment: NOT AUTHORIZED` (Theo quy tắc dự án, Agent không tự ý đẩy mã nguồn hoặc triển khai ra môi trường ngoài khi chưa có lệnh phê duyệt tường minh). Tài liệu này cung cấp quy trình chuẩn từng bước để thực thi ngay khi bạn cấp quyền triển khai S5.

---

## 1. Kiến trúc Triển khai Demo cho Giám đốc (Zero-Cost & Tương thích IP Việt Nam)

Để Giám đốc có thể mở ứng dụng bằng Zalo trên điện thoại cá nhân thông qua mã QR (đáp ứng **AC-14, AC-15, AC-16**), cần 2 thành phần:
1. **Backend Laravel API qua HTTPS công khai:** Sử dụng **Cloudflare Tunnel (`cloudflared`)** xuất phát trực tiếp từ máy tính của bạn tại Việt Nam (đảm bảo chứng chỉ SSL hợp lệ, miễn phí, không bị cold-start, và gọi `graph.zalo.me` từ IP Việt Nam).
2. **Frontend Mini App trên hạ tầng Zalo Testing:** Đóng gói và đẩy bản build từ `miniapp/` lên **Zalo Mini App Testing** thông qua Zalo Mini App Studio / VS Code Extension hoặc `zmp-cli`, sinh mã QR / Deep Link phiên bản kiểm thử.

---

## 2. Quy trình Thực thi Chi tiết (Step-by-Step Runbook)

### Bước 1 — Tạo Mini App ID & Lấy thông tin ứng dụng trên Zalo Developer Center
1. Đăng nhập vào [Zalo Mini App Developer Center](https://mini.zalo.me/developers/).
2. Tạo một Mini App mới đặt tên **`Smart Canteen Demo`**, ghi nhận:
   - `ZALO_APP_ID` (Mini App ID)
   - `ZALO_APP_SECRET` (App Secret)
3. Trong mục **Quản lý người dùng thử nghiệm (Testers)** trên Zalo Developer Center:
   - Thêm số điện thoại / tài khoản Zalo của **Giám đốc** và tài khoản của bạn vào danh sách được phép mở phiên bản **Development / Testing** (**AC-15**).

### Bước 2 — Mở kết nối HTTPS cho Laravel API bằng Cloudflare Tunnel (`api/`)
1. Khởi chạy Laravel API ở cổng `8000`:
   ```powershell
   cd api
   php artisan migrate --seed
   php artisan serve --host=127.0.0.1 --port=8000
   ```
2. Mở một cửa sổ PowerShell thứ hai và chạy Cloudflare Quick Tunnel:
   ```powershell
   cloudflared tunnel --url http://127.0.0.1:8000
   ```
3. Sao chép địa chỉ HTTPS được cấp (ví dụ: `https://smart-canteen-demo.trycloudflare.com`).
4. Cập nhật file `api/.env`:
   ```env
   APP_URL=https://smart-canteen-demo.trycloudflare.com
   CORS_ALLOWED_ORIGINS=http://localhost:3000,https://h5.zdn.vn
   ZALO_APP_ID=<ZALO_APP_ID_CUA_BAN>
   ZALO_APP_SECRET=<ZALO_APP_SECRET_CUA_BAN>
   ```
   Sau đó chạy `php artisan config:clear`.

### Bước 3 — Cấu hình & Deploy Zalo Mini App bản Testing (`miniapp/`)
1. Cập nhật file `miniapp/.env`:
   ```env
   VITE_API_BASE_URL=https://smart-canteen-demo.trycloudflare.com/api/v1
   ```
2. Kiểm tra bản build production:
   ```powershell
   cd miniapp
   npm run build
   ```
3. Mở **Zalo Mini App VS Code Extension** (hoặc dùng `npx zmp login` & `npx zmp deploy`):
   - Liên kết với `ZALO_APP_ID` vừa tạo.
   - Chọn **Deploy Testing** với ghi chú phiên bản: `v0.1.0 - Smart Canteen Full Workflow Demo`.
   - Nhận **Mã QR Code** và **Deep Link** từ kết quả deploy (**AC-14**).

### Bước 4 — Kiểm tra Trước khi Gửi Giám đốc (Pre-Flight Checklist)
- [ ] Mở URL `https://.../api/v1/categories` trên trình duyệt 4G điện thoại (ngoài mạng Wi-Fi nội bộ) thấy trả về JSON 4 danh mục (**AC-16**).
- [ ] Dùng điện thoại thứ hai quét mã QR Testing để xác nhận mở được ứng dụng, đặt thử 1 đơn và thấy đơn xuất hiện ngay trên trang `http://127.0.0.1:8000/admin` (**AC-14**, **AC-15**).
