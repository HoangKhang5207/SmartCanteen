# Đặc tả Yêu cầu Nghiệp vụ & Kỹ thuật — Smart Canteen (`docs/requirements.md`)

- **Dự án:** Smart Canteen (Zalo Mini App × React 18 × Laravel 13 × PostgreSQL 17.5)
- **Loại dự án:** Greenfield (Độc lập 100%, không kết nối cơ sở dữ liệu hay dịch vụ của hệ thống Căn Tin Số production)
- **Trạng thái:** Đã phê duyệt & hiện thực hóa hoàn chỉnh (P0 + P1)

---

## 1. Mục tiêu Nghiệp vụ (Business Objective)

Xây dựng một hệ thống đặt món và điều phối đơn hàng căn tin thông minh hoạt động đầu-cuối (end-to-end):
1. **Khách hàng (Customer):** Mở ứng dụng Smart Canteen trên Zalo Mini App, xem thực đơn, lọc theo danh mục, tìm kiếm món ăn, thêm món vào giỏ hàng, đặt đơn hàng và theo dõi trạng thái chế biến theo thời gian thực.
2. **Nhân viên Bếp (Staff):** Đăng nhập vào Cổng Quản trị Laravel Blade (hoặc sử dụng Admin API), tiếp nhận đơn hàng mới, chuyển trạng thái đơn theo quy trình chuẩn (`PENDING → PREPARING → READY → COMPLETED` hoặc `CANCELLED`).
3. **Quản trị viên (Admin):** Bao gồm toàn bộ quyền của Staff, đồng thời theo dõi thống kê doanh thu trong ngày và quản lý thực đơn (thêm/sửa/ẩn Danh mục & Món ăn).

---

## 2. Phân quyền Người dùng (RBAC Roles)

| Vai trò (`UserRole`) | Định danh & Xác thực | Quyền hạn được phép | Giới hạn bảo mật |
|---|---|---|---|
| **`customer`** | Xác thực qua Zalo Open API (`POST /api/v1/auth/zalo` kèm `appsecret_proof` HMAC-SHA256) hoặc Demo Login local (`POST /api/v1/auth/demo`), nhận Sanctum Bearer Token | Xem thực đơn công khai; tạo đơn hàng mới; xem danh sách và chi tiết đơn hàng của chính mình | Không truy cập được `/admin/*` hoặc `/api/v1/admin/*` (`403 Forbidden`); không xem được đơn của khách khác (`404 Not Found`); không tự hủy đơn sau khi đặt |
| **`staff`** | Xác thực Session trên Web (`/admin/login`) hoặc Sanctum Token trên API | Xem Dashboard thống kê hôm nay; xem danh sách/chi tiết toàn bộ đơn hàng; chuyển trạng thái đơn hàng theo State Machine; tra cứu danh sách Món ăn & Danh mục | Không được thêm/sửa/xóa Danh mục hoặc Món ăn (`403 Forbidden` bởi middleware `role:admin`) |
| **`admin`** | Xác thực Session trên Web (`/admin/login`) hoặc Sanctum Token trên API | Toàn quyền của `staff` + Thêm/sửa/xóa Danh mục (`/admin/categories`) và Món ăn (`/admin/products`) | Không được xóa danh mục đang chứa món ăn hoặc xóa món ăn đã phát sinh trong lịch sử đơn hàng (`restrictOnDelete`) |

---

## 3. Phân nhóm Yêu cầu Chức năng (MoSCoW / P0–P2)

### 3.1. Nhóm P0 — Cốt lõi (Bắt buộc hoàn thiện & Đã nghiệm thu)
- **Thực đơn công khai:** Xem danh sách danh mục đang hoạt động (`is_active = true`), danh sách món ăn theo danh mục, tìm kiếm tiếng Việt không phân biệt hoa thường, xem chi tiết từng món ăn và trạng thái còn món / hết món (`is_available`).
- **Giỏ hàng trong bộ nhớ (In-Memory Cart):** Thêm món, tăng/giảm số lượng (`1..20` phần tử mỗi món), tự động xóa khi giảm về `0`, xóa toàn bộ giỏ, tính tổng số lượng và tổng tiền tạm tính.
- **Đặt đơn hàng (Transactional Checkout):** Tạo `orders` và `order_items` trong một `DB::transaction` kèm khóa dòng `lockForUpdate()` trên bảng `products`.
- **Theo dõi đơn hàng & Lịch sử đơn:** Khách hàng xem danh sách đơn của mình và màn hình chi tiết đơn tự động cập nhật trạng thái (polling mỗi 7 giây cho đơn chưa kết thúc).
- **Điều phối đơn hàng trên Admin:** Nhân viên xem đơn mới, lọc theo trạng thái / mã đơn, chuyển trạng thái đơn hợp lệ và đồng bộ về Mini App.

### 3.2. Nhóm P1 — Bảo mật, Toàn vẹn dữ liệu & Vận hành (Bắt buộc & Đã nghiệm thu)
- **Định danh Zalo thật & Demo Login an toàn:** Xác minh `access_token` với `https://graph.zalo.me/v2.0/me` kèm `appsecret_proof`; chế độ Demo Login tự động khóa (`403`) khi `APP_ENV=production` hoặc `DEMO_LOGIN_ENABLED=false`.
- **Tính tiền phía Server (Server-Side Pricing):** Lưu tiền tệ bằng số nguyên `VND` (`unsignedInteger`). Server tự tính `unit_price`, `subtotal`, `total_amount` từ bảng `products` và bỏ qua mọi giá/tổng tiền do client gửi lên.
- **Bảo toàn giá lịch sử (Historical Price Immutability):** Khi Admin thay đổi giá bán của món ăn, các đơn hàng đã tạo trước đó vẫn giữ nguyên `unit_price`, `subtotal` và `total_amount`.
- **Chống tạo đơn trùng (Idempotency):** Bắt buộc header `Idempotency-Key` (`1..100` ký tự) khi tạo đơn. Cùng `(user_id, idempotency_key)` + cùng `payload_hash` trả về đơn cũ (`200 OK`); cùng key nhưng khác payload trả `409 Conflict`.
- **Kiểm soát đồng thời (Optimistic Locking):** Bảng `orders` có cột `version`. Mọi yêu cầu chuyển trạng thái phải gửi kèm `version` hiện tại; nếu lệch trả `409 Conflict`.
- **Nhật ký kiểm toán (Audit Log):** Mỗi lần chuyển trạng thái đơn thành công đều ghi vào bảng `order_status_logs` (`from_status`, `to_status`, `changed_by`, `created_at`).
- **Thống kê vận hành hôm nay:** Tổng số đơn hôm nay, số đơn đang xử lý, doanh thu các đơn `COMPLETED` hôm nay, phân bổ số lượng theo từng trạng thái.
- **Quản lý Thực đơn (CRUD):** Admin thêm/sửa/xóa Danh mục và Món ăn ngay trên giao diện Blade.

### 3.3. Nhóm P2 — Mở rộng tương lai (Ngoài phạm vi MVP hiện tại)
- Quét QR chọn bàn tại căn tin, thông báo đẩy Zalo ZNS, mã giảm giá / khuyến mãi, cổng thanh toán trực tuyến (ZaloPay/VietQR), báo cáo kế toán nâng cao theo tháng/quý.

---

## 4. Yêu cầu Phi chức năng (Non-Functional Requirements)

1. **Bảo mật & Tách biệt môi trường:**
   - Không lưu `ZALO_APP_SECRET` hay bất kỳ khóa bí mật nào trong mã nguồn Frontend (`miniapp/`).
   - Không phụ thuộc vào `localStorage`, `sessionStorage` hoặc `cookie` trên Zalo Mini App.
   - Cấu hình CORS chặt chẽ qua biến môi trường `CORS_ALLOWED_ORIGINS` (hỗ trợ `https://h5.zdn.vn` và preflight `OPTIONS`).
2. **Khả năng truy cập & Giao diện (UI/UX & WCAG 2.1 AA):**
   - Áp dụng nhận diện thương hiệu Smart Canteen (`canteen_logo.jpg`) với màu chủ đạo tương tác `#2E7D32` (`brand-700`, độ tương phản `5.14:1` trên nền trắng) kết hợp màu xanh lá logo `#5DBE46` (`brand-500`).
   - Đầy đủ 4 trạng thái giao diện (`Loading`, `Error` kèm nút thử lại, `Empty`, `Success`) trên mọi màn hình.
3. **Kiểm thử tự động:**
   - 100% quy tắc nghiệp vụ cốt lõi được bảo vệ bởi bộ kiểm thử tự động PHPUnit chạy trực tiếp trên PostgreSQL 17.5 và Vitest trên Frontend.
