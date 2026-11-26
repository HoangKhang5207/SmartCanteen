# Kịch bản Trình diễn Trực tiếp Toàn Hệ thống — Smart Canteen & Căn tin Số (`docs/demo-script.md`)

Tài liệu này hướng dẫn chi tiết cách khởi chạy và trình diễn trọn vẹn **10/10 phân hệ nghiệp vụ Căn tin Số** (từ Zalo Mini App React 18/TypeScript qua Laravel 13 REST API + PostgreSQL 17 tới Cổng Quản trị Admin/Staff Blade & Quầy POS).

---

## 1. Chuẩn bị Môi trường & Bộ Dữ liệu Mẫu (Pre-Demo Setup — 2 phút)

### 1.1. Khởi chạy Backend (`api/`) bằng Docker (Khuyến nghị — 1 lệnh duy nhất)
```powershell
cd d:\Coding\TitKul_ProjectTest\TestPart2\api
docker compose up -d --wait
docker compose exec app php artisan migrate:fresh --force --seed
```
- **API Base URL:** `http://127.0.0.1:8000/api/v1`
- **Admin & POS Portal:** `http://127.0.0.1:8000/admin/login`
- *(Hoặc chạy local PHP + PostgreSQL nếu máy cài sẵn PHP 8.4+: `php artisan migrate:fresh --seed && php artisan serve --host=127.0.0.1 --port=8000`)*

### 1.2. Khởi chạy Zalo Mini App (`miniapp/`)
```powershell
cd d:\Coding\TitKul_ProjectTest\TestPart2\miniapp
npm run dev
```
- **Mini App URL:** `http://localhost:3000/` (khuyến nghị mở DevTools ở khung di động `390×844` bên trái màn hình; mở Admin Portal bên phải màn hình).

### 1.3. Bảng Tài khoản & Dữ liệu Demo có sẵn sau `migrate:fresh --seed`

> Toàn bộ 4 Seeder (`MenuSeeder`, `StaffSeeder`, `StudentCardSeeder`, `DemoShowcaseSeeder`) đều **idempotent** (chạy lại nhiều lần không trùng dữ liệu). Các đơn hàng lịch sử mẫu được đánh dấu **ngày hôm qua** để thống kê **Hôm nay** trên Dashboard bắt đầu sạch tại `0 ₫` trước khi thao tác trực tiếp.

| Nhóm | Tài khoản / Mã | Thông tin đăng nhập / Trạng thái | Mục đích Demo |
|---|---|---|---|
| **Admin** | `admin@smartcanteen.local` | Mật khẩu: `Admin@123456` (`role: admin`) | Toàn quyền Dashboard, POS, Đơn hàng, Thực đơn/Tồn kho, Học sinh, Thẻ, Đối soát, Log Thông báo |
| **Staff (Bếp / Quầy)** | `staff@smartcanteen.local` | Mật khẩu: `Staff@123456` (`role: staff`) | Quầy POS `/admin/pos`, điều phối đơn hàng, xem thẻ/đối soát/thông báo; bị chặn `403` khi sửa thực đơn/học sinh |
| **Phụ huynh #1 (Chính)** | SĐT `0901234567` | Nút đăng nhập nhanh **Phụ huynh Demo** trên Mini App | Sở hữu **2 con (2 thẻ)** — vừa trải nghiệm thẻ đã liên kết, vừa demo liên kết thẻ mới qua OTP |
| **Học sinh #1** | SĐT `0901234568` | Nút đăng nhập nhanh **Học sinh Demo** trên Mini App | Gắn trực tiếp với thẻ `F156BA` (`role: student`): xem thẻ/hạn mức/lịch sử, đặt món trong hạn mức, bị chặn `403` khi sửa hạn mức hoặc hủy ngân hàng |
| **Phụ huynh #2** | SĐT `0909888777` | Nhập tay tại modal SĐT Demo | Sở hữu thẻ số dư thấp (`15.000 ₫`) & thẻ đang khóa (`LOCKED`) |
| **Thẻ `F156BA`** | `HS-2026-0001` · *Titkul Student* (8A2) | `ACTIVE` · **Đã liên kết Vietcombank** (`****5678`, số dư Sandbox **`500.000 ₫`**) · Hạn mức **`100.000 ₫/ngày`** (`spent_today = 0`) | Sẵn sàng đặt Online, quẹt POS, đổi hạn mức, xem lịch sử đơn (`SC-DEMO-000001`, `POS-DEMO-000001`), lịch sử hạn mức và `2` thông báo chưa đọc |
| **Thẻ `B294CD`** | `HS-2026-0002` · *Nguyễn Minh Anh* (6A1) | `ACTIVE` · **Chưa liên kết ngân hàng** | Demo trực tiếp luồng **Liên kết Ngân hàng + SMS OTP** và **Hủy liên kết OTP** |
| **Thẻ `E741MN`** | `HS-2026-0003` · *Trần Gia Huy* (7B3) | `ACTIVE` · Liên kết **BIDV** (`****2222`, số dư Sandbox **`15.000 ₫`**) · Hạn mức `100.000 ₫/ngày` | Demo lỗi **Không đủ số dư ngân hàng (`INSUFFICIENT_BANK_BALANCE`)** ghi bền vững vào sổ cái `payment_transactions` |
| **Thẻ `L003KK`** | `HS-2026-0004` · *Trần Bảo Ngọc* (9A1) | `LOCKED` (Đã khóa thẻ) | Demo Admin mở/khóa thẻ & chặn thanh toán thẻ bị khóa tại quầy POS |
| **Thực đơn** | 6 danh mục · 30 món | `28` món còn hàng (`stock = 30`, riêng *Bánh flan* `stock = 4` sắp hết), `2` món hết hàng (`stock = 0`) | Demo lọc danh mục, tìm kiếm tiếng Việt, tồn kho sắp hết & trừ/hoàn kho nguyên tử |
| **Đối soát mẫu** | Ngày hôm qua (`sandbox_epay`) | `2 MATCHED` + `1 STATUS_MISMATCH` (`TXN-DEMO-TIMEOUT-001`, `45.000 ₫`) | Mở `/admin/canteen/reconciliation` thấy ngay dòng lệch chưa xử lý để demo nút **Xác nhận xử lý** |

---

## 2. Kịch bản Trình diễn Đầy đủ 12 Bước (10–12 phút)

### Giai đoạn A — Khám phá Công khai, Lazy Auth & Thực đơn Tồn kho (Mini App)

#### Bước 1 — Lazy Auth & Thực đơn có Tồn kho (`MenuPage`, `ProductDetailPage`)
1. Mở `http://localhost:3000/` (**chưa đăng nhập**): cho Giám đốc thấy khách vẫn duyệt mượt toàn bộ **6 danh mục · 30 món ăn**, tìm kiếm tiếng Việt không dấu/có dấu (`"sườn"`), phân trang, và xem badge tồn kho:
   - *Bánh flan trứng sữa caramel* hiển thị nhãn sắp hết (`Còn 4 phần`).
   - *Cơm cá kho tộ miền Tây* hiển thị **"Hết món"** và khóa nút thêm.
2. Thêm `1x Cơm sườn nướng mật ong` (`45.000 ₫`) và `1x Trà đào cam sả` (`25.000 ₫`) vào giỏ (hoạt động không cần đăng nhập).

#### Bước 2 — Xác thực SĐT Zalo (Lazy Auth Modal)
1. Bấm vào tab **Căn tin** (`/canteen`) hoặc nút **Đặt món** trong Giỏ hàng → hệ thống tự bật **Zalo Phone Login Modal**.
2. Chọn nút đăng nhập nhanh **Phụ huynh Demo (`0901234567`)** (ở production gọi `zmp-sdk getPhoneNumber()` gửi `phone_token` lên server đổi qua `graph.zalo.me/v2.0/me/info` — không lộ `ZALO_APP_SECRET`).
3. Chỉ vào thanh **BottomNav 5 tab**: tab **Thông báo** lập tức hiện badge đỏ **`2`** (2 thông báo chưa đọc được đồng bộ tự động).

---

### Giai đoạn B — Phân hệ Căn tin Số: Đa thẻ, Liên kết Ngân hàng + OTP & Hạn mức

#### Bước 3 — Xem Căn tin Hub đa thẻ & Hạn mức (`CanteenHubPage`)
1. Tại `/canteen`, chỉ cho thấy phụ huynh `0901234567` quản lý cùng lúc **2 thẻ**:
   - **Thẻ `F156BA` (Titkul Student — 8A2):** đã liên kết **Vietcombank (`****5678`)**, hạn mức `100.000 ₫/ngày`, còn lại `100.000 ₫`, kèm 2 đơn hàng lịch sử hôm qua (`SC-DEMO-000001`, `POS-DEMO-000001`).
   - Bấm **"Lịch sử hạn mức"** (`/canteen/limit-history`) để xem audit log các lần tăng hạn mức (`0 → 50k → 100k`).
2. Bấm **"Thay đổi hạn mức"** (`SpendingLimitSheet`), chọn nhanh `150.000 ₫` hoặc nhập tùy chỉnh → lưu thành công và kiểm tra lịch sử hạn mức cập nhật tức thì + badge thông báo tăng thêm `1`.

#### Bước 4 — Demo trực tiếp Liên kết Ngân hàng + SMS OTP trên Thẻ thứ 2 (`B294CD`)
1. Chọn thẻ **Nguyễn Minh Anh (`B294CD`)** đang ở trạng thái **Chưa liên kết**.
2. Bấm **"Liên kết ngân hàng"**: chọn `MB Bank`, nhập STK `09876543210`, tên chủ TK `NGUYEN VAN AN`, CCCD `079090001234`, SĐT (đã tự điền sẵn `0901234567`), tích đồng ý điều khoản trích nợ tự động → bấm **Tiếp tục**.
3. Tại màn hình **Nhập OTP 6 chữ số** (`/canteen/verify-otp`):
   - Chỉ ra tính năng bảo mật: đếm ngược `180s`, cooldown gửi lại `30s`, tối đa `5` lần nhập sai, OTP băm `bcrypt` trên DB, và `debug_otp` hiển thị ở môi trường local/dev (`SMS_DRIVER=log`).
   - Nhập đúng 6 số OTP → liên kết thành công, STK & CCCD lập tức được mã hóa AES-256 trong DB và chỉ hiển thị dạng che mờ (`*******3210`).
4. *(Tùy chọn)* Bấm **"Hủy liên kết"** trên chính thẻ `B294CD` → nhập OTP hủy (`BANK_UNLINK`) → chứng minh màn hình **Hủy liên kết thành công** mà vẫn bảo toàn dữ liệu lịch sử.

---

### Giai đoạn C — Đặt món Online, Trừ Kho/Hạn mức/Ngân hàng & Điều phối Bếp

#### Bước 5 — Thanh toán Đơn Online bằng Thẻ Học sinh (Idempotency + Server Pricing + Atomic Lock)
1. Quay lại **Giỏ hàng** (`/cart`) đang có `1x Cơm sườn` (`45k`) + `1x Trà đào` (`25k`) = **`70.000 ₫`**.
2. Chỉ 3 phương thức thanh toán trên `radiogroup` chuẩn WAI-ARIA: **Thẻ học sinh (`F156BA`)**, **Thanh toán VNPay (Sandbox)**, **Tiền mặt tại quầy**.
3. Chọn **Thẻ học sinh (`F156BA`)**, nhập ghi chú `"Ít cơm, lấy thêm canh nóng"`, bấm **Đặt món ngay**.
4. Giải thích ngắn điểm mạnh kỹ thuật cho 2 anh Developer:
   - Header `Idempotency-Key` chống đúp đơn.
   - Server khóa dòng theo **thứ tự khóa chuẩn chống deadlock**: `student_cards → spending_limits → sandbox_bank_accounts → products (id ASC) → orders → payment_transactions`.
   - Trừ đồng thời: `stock_quantity` của 2 món (`30 → 29`), `spent_today` (`0 → 70.000 ₫`), và số dư Vietcombank Sandbox (`500.000 → 430.000 ₫`), ghi sổ cái `payment_transactions` (`SUCCEEDED`) và phát thông báo sau `DB::afterCommit()`.

#### Bước 6 — Điều phối Đơn trên Admin Portal & Polling Thời gian thực
1. Cửa sổ phải: đăng nhập `http://127.0.0.1:8000/admin/login` bằng **`admin@smartcanteen.local` / `Admin@123456`**.
2. Tại **Dashboard (`/admin`)**: thấy **Tổng đơn hôm nay = 1**, đơn mới `SC-...` (`v1`, `PAID`, `STUDENT_CARD`).
3. Bấm chuyển **`PENDING (v1) → PREPARING (v2) → READY (v3)`**:
   - Quan sát cửa sổ trái (**Mini App** — không reload): sau `<= 7s`, trạng thái tự đổi sang **"Sẵn sàng nhận món" (`READY`)**, đồng thời tab **Thông báo** tăng badge chưa đọc.
4. Bấm **`COMPLETED (v4)`**: quay lại `/admin` thấy **Doanh thu hoàn thành hôm nay = `70.000 ₫`**.

#### Bước 7 — Chứng minh Hoàn tiền, Hoàn Hạn mức & Hoàn Kho khi Hủy đơn (`CANCELLED`)
1. Đặt nhanh 1 đơn thứ 2 bằng thẻ `F156BA` (ví dụ `1x Bánh flan` `15.000 ₫` — tồn kho giảm `4 → 3`, `spent_today` tăng `70k → 85k`).
2. Trên Admin Portal, chuyển đơn đó sang **`CANCELLED`** → mở lại chi tiết thẻ `F156BA` và kho món *Bánh flan*:
   - Tồn kho tự hoàn lại `3 → 4`, `spent_today` hoàn `85k → 70k`, số dư Sandbox hoàn `+15.000 ₫`, sổ cái ghi nhận 1 dòng `REFUND / SUCCEEDED` và phụ huynh nhận thông báo hoàn tiền.

---

### Giai đoạn D — POS Quầy Căn tin, Xử lý Lỗi Ngân hàng & VNPay Sandbox

#### Bước 8 — Quầy Thu ngân POS (`/admin/pos`) & In Biên lai
1. Trên Admin Portal, mở **POS Bán hàng (`/admin/pos`)** (có thể mở bằng tài khoản `staff@smartcanteen.local` để chứng minh quyền `staff`).
2. Tra cứu mã thẻ **`F156BA`** (hoặc `HS-2026-0001`): hiển thị tên học sinh, ngân hàng đã che mờ, hạn mức còn lại hôm nay.
3. Chọn `1x Sữa chua nếp cẩm` (`20.000 ₫`) → bấm **Thanh toán quẹt thẻ**:
   - Tạo đơn `#POS-...` (`COMPLETED`, `PAID`), tự động prepend vào danh sách **Đơn POS gần đây**, mở nút **In biên lai** (`/admin/pos/receipt/{id}` tối ưu máy in nhiệt `@media print`).
4. **Demo các tình huống lỗi bảo vệ tiền thật ngay tại quầy POS:**
   - Tra cứu thẻ **`L003KK`** (`LOCKED`): hệ thống hiển thị cảnh báo thẻ bị khóa, chặn thanh toán.
   - Tra cứu thẻ **`E741MN`** (số dư BIDV Sandbox chỉ còn **`15.000 ₫`**), chọn món `45.000 ₫` → bấm thanh toán: hệ thống trả lỗi **`422 INSUFFICIENT_BANK_BALANCE`** (*Tài khoản ngân hàng không đủ số dư*), **KHÔNG** trừ kho, **KHÔNG** trừ hạn mức, nhưng **VẪN ghi bền vững** dòng `FAILED` vào sổ cái `payment_transactions` để kế toán tra cứu!

#### Bước 9 — Thanh toán VNPay Sandbox (`HMAC-SHA512`)
1. Trên Mini App Giỏ hàng, chọn món bất kỳ và chọn phương thức **Thanh toán VNPay (Sandbox)** → tạo đơn `UNPAID`.
2. Tại Chi tiết đơn, bấm **"Thanh toán qua VNPay"**: hệ thống gọi `POST /api/v1/payments/vnpay/create-url` sinh URL có chữ ký `vnp_SecureHash` (`HMAC-SHA512`) và ghi sổ cái `PENDING`.

---

### Giai đoạn E — Quản trị Học sinh, Thẻ, Tồn kho, Đối soát & Trung tâm Thông báo

#### Bước 10 — Quản trị Học sinh, Tồn kho & Đối soát Ngân hàng (Admin Blade)
1. **Học sinh (`/admin/canteen/students`):** Xem 4 học sinh mẫu, tạo nhanh 1 học sinh mới hoặc cấp thẻ thay thế (tự động thu hồi thẻ cũ).
2. **Tồn kho (`/admin/products`):** Bấm nhập kho nhanh (`+10` phần cho món *Bánh flan*) và cho thấy cơ chế chống ghi đè tồn kho đồng thời (`original_stock_quantity`).
3. **Đối soát Ngân hàng (`/admin/canteen/reconciliation`):**
   - Mở đợt đối soát ngày hôm qua đã seed sẵn (`2 MATCHED`, `1 STATUS_MISMATCH` cho `TXN-DEMO-TIMEOUT-001` `45.000 ₫`).
   - Nhập ghi chú xử lý và bấm **"Xác nhận đã xử lý"** → đợt đối soát đổi sang `COMPLETED`.
   - Bấm **"Chạy đối soát hôm nay"** → đối chiếu tức thì các giao dịch vừa phát sinh trong buổi demo!

#### Bước 11 — Trung tâm Thông báo & Nhật ký Push Zalo OA/ZNS
1. **Mini App (`/notifications`):** Mở tab **Thông báo**, lọc *Chưa đọc*, bấm vào 1 thông báo để tự đánh dấu đã đọc & điều hướng thẳng tới đơn hàng/lịch sử hạn mức; bấm **"Đọc tất cả"** → badge đỏ trên `BottomNav` về `0`.
2. **Admin (`/admin/canteen/notifications`):** Mở nhật ký gửi thông báo toàn hệ thống, lọc theo `type` / `push_status`, chứng minh **không hề lộ STK/CCCD/SĐT nguyên văn** trong log.

#### Bước 12 — Chứng minh Phân quyền 4 Vai trò: Chế độ Học sinh (`student` — `0901234568`)
1. Trên Mini App (`/canteen`), bấm **Đăng xuất** → đăng nhập lại bằng nút **Học sinh Demo (`0901234568`)**.
2. Quan sát:
   - Màn hình Căn tin hiển thị thẻ `F156BA`, hạn mức còn lại hôm nay và lịch sử giao dịch ở **chế độ chỉ xem dành cho Học sinh** (ẩn toàn bộ nút *Liên kết / Hủy ngân hàng / Đổi hạn mức*).
   - Học sinh vẫn có thể đặt món bằng thẻ của mình trong phạm vi hạn mức còn lại.
   - Nếu gọi trực tiếp API tài chính của phụ huynh (`PUT /spending-limit` hoặc `POST /bank-links`), Backend trả ngay **`403 PARENT_ROLE_REQUIRED`**.
