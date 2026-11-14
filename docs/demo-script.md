# Kịch bản Trình diễn Trực tiếp (Demo Script 8 Bước) — Smart Canteen (`docs/demo-script.md`)

Tài liệu này hướng dẫn chi tiết từng bước trình diễn trực tiếp hệ thống **Smart Canteen** cho Giám đốc, chứng minh một luồng Full-stack hoạt động thực tế từ giao diện khách hàng (Zalo Mini App) qua REST API, PostgreSQL tới Cổng Quản trị Bếp (Laravel Blade) và đồng bộ ngược lại.

---

## 1. Chuẩn bị Trước Buổi Trình diễn (Pre-Demo Setup — 2 phút)

1. **Khởi động PostgreSQL & làm mới dữ liệu mẫu chuẩn:**
   ```powershell
   # Nếu dùng PostgreSQL cục bộ ở cổng 54329:
   D:\DB_Design\PostgreSQL\pgsql\bin\postgres.exe -D .local\pgdata -p 54329 -c listen_addresses=127.0.0.1

   # Trong thư mục api/:
   cd api
   php artisan migrate:fresh --seed
   php artisan serve --host=127.0.0.1 --port=8000
   ```
2. **Khởi động giao diện Zalo Mini App (`miniapp/`):**
   ```powershell
   cd miniapp
   npm run dev
   ```
3. **Bố trí màn hình trình chiếu (Split Screen):**
   - **Nửa màn hình bên trái (Khung di động 390×844):** Mở Zalo Mini App tại `http://localhost:3000/` (hoặc quét QR trên điện thoại nếu đã chạy S5).
   - **Nửa màn hình bên phải (Màn hình quản trị Bếp):** Mở Cổng Quản trị tại `http://127.0.0.1:8000/admin/login`.

---

## 2. Lời dẫn Mở đầu với Giám đốc (30 giây)

> *"Dạ thưa anh, đối với bài Test Part 2, em đã thiết kế và phát triển mới hoàn toàn ứng dụng **Smart Canteen** gồm **Zalo Mini App (React + TypeScript + ZaUI)** ở phía khách hàng, kết nối tới **Laravel 13 REST API + Cổng Quản trị Bếp** và cơ sở dữ liệu **PostgreSQL** riêng biệt. Toàn bộ dữ liệu đều độc lập 100%, không chạm vào hệ thống Căn Tin Số production.*
>
> *Sau đây em xin thao tác trực tiếp luồng 8 bước: khách đặt suất ăn bên trái → hệ thống tính tiền phía server và chống trùng đơn → bếp tiếp nhận và chuyển trạng thái bên phải → màn hình của khách tự động cập nhật theo thời gian thực."*

---

## 3. Kịch bản Thao tác 8 Bước Trọng tâm (5 phút)

### Bước 1 — Mở ứng dụng Smart Canteen & Khám phá Thực đơn (AC-01, AC-02)
- **Thao tác (Cửa sổ trái - Mini App):**
  - Chỉ vào header mang nhận diện thương hiệu **Smart Canteen** (Logo + tông màu xanh lá chuẩn WCAG AA).
  - Bấm lọc qua các chip danh mục: `Tất cả`, `Cơm`, `Bún & Phở`, `Đồ uống`, `Tráng miệng`.
  - Gõ từ khóa `"sườn"` vào ô tìm kiếm để cho thấy tìm kiếm tiếng Việt debounce hoạt động tức thì; sau đó bấm xóa từ khóa.
  - Chỉ vào món *"Bún bò Huế đặc biệt"* đang gắn nhãn **"Hết món"** (nút thêm bị khóa — chứng minh **AC-05**).

### Bước 2 — Xem Chi tiết món & Thêm vào Giỏ hàng (AC-02, AC-03)
- **Thao tác (Cửa sổ trái - Mini App):**
  - Bấm vào món **"Cơm sườn nướng mật ong"** (`45.000 ₫`) để mở màn hình Chi tiết món.
  - Tăng số lượng lên `2` phần (`90.000 ₫`) rồi bấm **"Thêm vào giỏ"**.
  - Quay lại thực đơn, bấm nút **"+ Thêm"** nhanh trên món **"Trà đào cam sả"** (`25.000 ₫`).
  - Chỉ vào thanh nổi màu tối ở đáy màn hình hiển thị: `3 phần · 115.000 ₫`.

### Bước 3 — Điều chỉnh Giỏ hàng & Đặt đơn hàng (AC-03, AC-04, AC-06)
- **Thao tác (Cửa sổ trái - Mini App):**
  - Bấm **"Xem giỏ hàng"** (`/cart`).
  - Thử tăng/giảm số lượng món và nhập ghi chú: `"Ít cơm, lấy thêm canh nóng"`.
  - Bấm nút **"Đặt món ngay · 115.000 ₫"**.
- **Điểm nhấn kỹ thuật giải thích cho Giám đốc:**
  - Khi bấm đặt đơn, Mini App đính kèm header `Idempotency-Key` gắn chặt với tổ hợp giỏ hàng hiện tại (nếu mạng chập chờn khách bấm đúp 2 lần, server chỉ tạo duy nhất 1 đơn).
  - Toàn bộ đơn giá `unit_price`, `subtotal` và `total_amount = 115.000 ₫` đều do Laravel tính toán trong `DB::transaction` kèm khóa dòng `lockForUpdate()`, bỏ qua hoàn toàn mọi giá tiền nếu client cố tình sửa payload.

### Bước 4 — Kiểm tra Màn hình Theo dõi Đơn vừa tạo (AC-04)
- **Quan sát (Cửa sổ trái - Mini App):**
  - Ứng dụng tự động chuyển sang màn hình **Chi tiết đơn hàng** (`/orders/:id`) với mã đơn định dạng `SC-261114-000001`, trạng thái **"Chờ xác nhận" (`PENDING`)** và dòng thông báo *"Tự động cập nhật trạng thái mỗi 7 giây"*.

### Bước 5 — Nhân viên Bếp mở Cổng Quản trị & Tiếp nhận đơn mới (AC-07)
- **Thao tác (Cửa sổ phải - Admin Portal):**
  - Đăng nhập tại `http://127.0.0.1:8000/admin/login` bằng tài khoản `admin@smartcanteen.local` / `Admin@123456`.
  - Ngay tại **Bảng điều hành hôm nay (`/admin`)**, chỉ cho Giám đốc thấy:
    - Thẻ **Tổng đơn hôm nay** vừa nhảy lên `1`, **Đơn đang xử lý (Bếp)** là `1`.
    - Đơn hàng `SC-261114-000001` (`v1`) hiển thị đầy đủ tên khách, `2x Cơm sườn nướng mật ong`, `1x Trà đào cam sả` và ghi chú `"Ít cơm, lấy thêm canh nóng"`.

### Bước 6 — Chuyển trạng thái đơn từ `PENDING` → `PREPARING` → `READY` (AC-08)
- **Thao tác (Cửa sổ phải - Admin Portal):**
  1. Bấm nút **`→ PREPARING`** trên dòng đơn `SC-261114-000001`:
     - Trạng thái đơn đổi sang `PREPARING` (`v2`), ghi bản ghi đầu tiên vào bảng `order_status_logs`.
  2. Bấm vào mã đơn `SC-261114-000001` để mở trang Chi tiết đơn quản trị (`/admin/orders/1`):
     - Chỉ vào bảng **Nhật ký chuyển trạng thái (Audit Log)** đã ghi rõ người thực hiện và thời gian.
     - Bấm nút **`Chuyển sang READY`** (`v2 → v3`).

### Bước 7 — Giám đốc quan sát Mini App tự động cập nhật trạng thái (AC-09)
- **Quan sát (Cửa sổ trái - Mini App — không cần tải lại trang):**
  - Trong vòng `<= 7 giây`, màn hình Chi tiết đơn trên Mini App tự động chuyển huy hiệu sang màu xanh lá **"Sẵn sàng nhận món" (`READY`)** cùng thông điệp: *"Món ăn đã sẵn sàng tại quầy nhận món! Vui lòng đọc mã đơn để nhận."*

### Bước 8 — Kiểm tra Lịch sử đơn, Hoàn tất đơn & Chứng minh Bảo toàn Giá Lịch sử (AC-09, AC-11)
- **Thao tác:**
  1. Trên **Mini App (trái):** Bấm tab **"Đơn của tôi"** (`/orders`) ở thanh điều hướng đáy để kiểm tra lịch sử đơn và mã đơn `SC-261114-000001`.
  2. Trên **Admin Portal (phải):** Bấm **`Chuyển sang COMPLETED`** (`v3 → v4`). Quay về **Tổng quan hôm nay (`/admin`)**, chỉ vào **Doanh thu hoàn thành hôm nay** đã cộng chính xác **`115.000 ₫`**.
  3. **Chứng minh bổ sung AC-11 (Bảo toàn giá lịch sử):** Vào mục **Món ăn (`/admin/products`)**, sửa giá *"Cơm sườn nướng mật ong"* từ `45.000 ₫` lên `55.000 ₫`, sau đó mở lại đơn `SC-261114-000001` trên cả Admin và Mini App → đơn giá lịch sử của đơn đã đặt vẫn giữ nguyên `45.000 ₫` và tổng đơn vẫn là `115.000 ₫`.
