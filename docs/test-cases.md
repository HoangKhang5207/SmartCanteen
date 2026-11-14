# Danh mục Kịch bản Kiểm thử (Test Cases) — Smart Canteen (`docs/test-cases.md`)

Tài liệu này liệt kê đầy đủ các kịch bản kiểm thử chức năng, bảo mật, toàn vẹn dữ liệu và giao diện của dự án **Smart Canteen**, ánh xạ trực tiếp tới các bài kiểm thử tự động trong `api/tests/` và `miniapp/src/`.

---

## 1. Nhóm Kiểm thử Thực đơn & Khởi tạo Dữ liệu (Menu & Seeder — S1)

| Mã TC | Tên kịch bản kiểm thử | Tiền điều kiện | Các bước thực hiện | Kết quả mong đợi | Bài test tự động tương ứng | Kết quả |
|---|---|---|---|---|---|---|
| **TC-01** | Seed dữ liệu thực đơn giả lập idempotent (**AC-01**) | DB trống | Chạy `MenuSeeder` 2 lần liên tiếp | Tạo đúng 4 danh mục (`>= 3`) và 12 sản phẩm (`>= 10`), không nhân đôi bản ghi khi chạy lại | `MenuSeederTest` | **PASS** |
| **TC-02** | Lưu giá sản phẩm bằng số nguyên VND & chặn xóa danh mục đang có món | Đã migrate DB | Tạo danh mục + món ăn giá `45000`; thử xóa danh mục | Giá trả về kiểu `int`; xóa danh mục bị chặn bởi khóa ngoại `restrictOnDelete` | `SchemaTest` | **PASS** |
| **TC-03** | Lấy danh sách danh mục đang hoạt động (**AC-02**) | Có danh mục `is_active = true` và `false` | Gọi `GET /api/v1/categories` | Chỉ trả về các danh mục có `is_active = true` | `MenuApiTest::test_lists_only_active_categories` | **PASS** |
| **TC-04** | Lọc món ăn theo danh mục & tìm kiếm tiếng Việt không phân biệt hoa thường (**AC-02**) | Đã seed thực đơn | Gọi `GET /api/v1/products?category_id=1` và `?search=sườn nướng` | Trả về đúng các món thuộc danh mục đang hoạt động và khớp từ khóa tiếng Việt; truyền `category_id` không tồn tại trả `422` | `MenuApiTest` | **PASS** |
| **TC-05** | Xem chi tiết món ăn & ẩn món thuộc danh mục ngừng hoạt động | Có món thuộc danh mục ẩn | Gọi `GET /api/v1/products/{id}` | Trả `200 OK` cho món hợp lệ; trả `404 Not Found` nếu món không tồn tại hoặc thuộc danh mục `is_active = false` | `MenuApiTest` | **PASS** |
| **TC-06** | Kiểm tra CORS cho origin Zalo Mini App (`https://h5.zdn.vn`) (**AC-16**) | Cấu hình `CORS_ALLOWED_ORIGINS` | Gửi preflight `OPTIONS /api/v1/products` từ `https://h5.zdn.vn` và từ `https://evil.example.com` | Cho phép `https://h5.zdn.vn`; từ chối origin lạ | `CorsTest` | **PASS** |

---

## 2. Nhóm Kiểm thử Xác thực, Giỏ hàng & Đặt đơn (Auth, Cart & Checkout — S2)

| Mã TC | Tên kịch bản kiểm thử | Tiền điều kiện | Các bước thực hiện | Kết quả mong đợi | Bài test tự động tương ứng | Kết quả |
|---|---|---|---|---|---|---|
| **TC-07** | Xác thực Zalo Open API kèm `appsecret_proof` HMAC-SHA256 | Cấu hình `ZALO_APP_SECRET` | Gọi `POST /api/v1/auth/zalo` với `access_token` hợp lệ và không hợp lệ | Khi hợp lệ: gửi đúng `appsecret_proof` tới `graph.zalo.me`, upsert user `role=customer` và trả Sanctum token (`200`). Khi Zalo từ chối: trả `401` | `AuthApiTest` | **PASS** |
| **TC-08** | Đăng nhập Demo ở `local`/`testing` và tự động chặn ở `production` | `DEMO_LOGIN_ENABLED` | Gọi `POST /api/v1/auth/demo` khi bật và khi tắt | Trả `200` kèm token khi bật ở `local`/`testing`; trả `403 Forbidden` khi tắt | `AuthApiTest` | **PASS** |
| **TC-09** | Logic giỏ hàng trong bộ nhớ (**AC-03**) | Khởi tạo giỏ rỗng | Thêm món, cộng dồn số lượng, thử vượt ngưỡng `20`, thêm món `is_available = false`, giảm số lượng về `0` | Giới hạn tối đa `20`/món, từ chối món hết hàng, tự xóa món khi số lượng về `0`, tính đúng tổng tạm tính | `cart-logic.test.ts` | **PASS** |
| **TC-10** | Tạo đơn hàng và tính tổng tiền phía server, bỏ qua giá client gửi (**AC-04**) | Khách đã đăng nhập | Gửi `POST /api/v1/orders` kèm `Idempotency-Key` và cố tình truyền `total_amount: 1, unit_price: 1` | Server bỏ qua giá client, tự tính `unit_price`, `subtotal`, `total_amount` từ bảng `products` và trả `201 Created` | `OrderApiTest::test_creates_order_and_computes_total_on_server_ignoring_client_total` | **PASS** |
| **TC-11** | Từ chối đơn có món hết hàng, thuộc danh mục ẩn, hoặc số lượng ngoài `1..20` (**AC-05**) | Khách đã đăng nhập | Gửi `POST /api/v1/orders` với món `is_available = false` hoặc `quantity = 25` | Trả `422 Unprocessable Entity`, không tạo bản ghi trong `orders` | `OrderApiTest::test_rejects_unavailable_or_inactive_category_product_or_invalid_quantity` | **PASS** |
| **TC-12** | Chống tạo đơn trùng bằng `Idempotency-Key` (**AC-06**) | Khách đã đăng nhập | Gửi `POST /api/v1/orders` 2 lần với cùng `Idempotency-Key` + cùng payload; lần 3 dùng cùng key nhưng đổi số lượng | Lần 2 trả `200 OK` với đúng đơn đã tạo (chỉ có 1 đơn trong DB); lần 3 trả `409 Conflict` | `OrderApiTest::test_idempotency_key_returns_same_order_on_retry_and_409_on_different_payload` | **PASS** |
| **TC-13** | Bảo toàn giá lịch sử khi giá món ăn thay đổi sau đó (**AC-11**) | Đã có đơn hàng đặt ở giá `45.000 ₫` | Admin đổi giá món lên `60.000 ₫`; kiểm tra lại đơn hàng cũ | Đơn hàng cũ vẫn giữ nguyên `unit_price = 45000` và `total_amount` ban đầu | `OrderApiTest::test_changing_product_price_later_does_not_change_existing_order` | **PASS** |
| **TC-14** | Cô lập dữ liệu đơn hàng giữa các khách hàng (**AC-10**) | Khách A và Khách B đều có đơn | Khách A gọi `GET /api/v1/orders` và `GET /api/v1/orders/{id_cua_B}` | Danh sách chỉ chứa đơn của A; truy cập đơn của B nhận `404 Not Found` | `OrderApiTest::test_customer_lists_only_own_orders_and_gets_404_for_other_users_order` | **PASS** |

---

## 3. Nhóm Kiểm thử Quản trị Admin/Staff, State Machine, Optimistic Locking & Thống kê (S3 + S4)

| Mã TC | Tên kịch bản kiểm thử | Tiền điều kiện | Các bước thực hiện | Kết quả mong đợi | Bài test tự động tương ứng | Kết quả |
|---|---|---|---|---|---|---|
| **TC-15** | Nhân viên xem danh sách đơn, lọc theo trạng thái & mã đơn, kiểm tra giới hạn `limit` (**AC-07**) | Có nhiều đơn hàng | Gọi `GET /api/v1/admin/orders?status=pending`, `?search=...`, `?limit=2`, và `?status=INVALID` | Chuẩn hóa `pending` → `PENDING`; trả đúng số lượng theo `limit`; trả `422` khi truyền `status` hoặc `limit` không hợp lệ | `AdminOrderApiTest` | **PASS** |
| **TC-16** | Chuyển trạng thái hợp lệ theo State Machine, tăng `version` và ghi `order_status_logs` (**AC-08**) | Đơn ở `PENDING` (`v1`) | Nhân viên chuyển `PENDING (v1) → PREPARING (v2) → READY (v3) → COMPLETED (v4)` | Cập nhật trạng thái thành công, tăng `version` mỗi bước, ghi đủ 3 bản ghi trong `order_status_logs` | `AdminOrderApiTest::test_staff_transitions_order_through_valid_states_increments_version_and_writes_status_logs` | **PASS** |
| **TC-17** | Từ chối chuyển trạng thái sai quy tắc State Machine (**AC-08**) | Đơn ở `PENDING` (`v1`) | Nhân viên cố chuyển thẳng từ `PENDING` sang `READY` | Trả `422 Unprocessable Entity`, giữ nguyên `PENDING (v1)`, không ghi log | `AdminOrderApiTest::test_invalid_status_transition_returns_422_and_writes_no_log` | **PASS** |
| **TC-18** | Chống ghi đè đồng thời bằng Optimistic Locking `version` | Đơn ở `PENDING` (`v1`) | Nhân viên 1 chuyển sang `PREPARING` (`v1 → v2`); Nhân viên 2 gửi yêu cầu hủy với `version = 1` cũ | Yêu cầu của Nhân viên 2 bị từ chối với `409 Conflict`, đơn giữ nguyên `PREPARING (v2)` | `AdminOrderApiTest::test_stale_version_returns_409_conflict_and_prevents_concurrent_overwrite` | **PASS** |
| **TC-19** | Thống kê hôm nay chỉ cộng doanh thu đơn `COMPLETED` trong ngày | Có đơn hôm nay và hôm qua | Gọi `GET /api/v1/admin/statistics` | `today_completed_revenue` chỉ cộng các đơn `COMPLETED` hôm nay; không tính đơn hôm qua hoặc đơn chưa hoàn thành | `AdminStatisticsApiTest` | **PASS** |
| **TC-20** | Phân quyền RBAC trên Cổng Quản trị Blade (`customer` vs `staff` vs `admin`) (**AC-10**) | Có đủ 3 vai trò | `customer` thử đăng nhập `/admin/login`; `staff` xem thực đơn và thử sửa/xóa món; `admin` thêm/sửa/xóa món | `customer` bị chặn `403` (hoặc tự logout nếu truy cập `GET /admin/login`); `staff` xem được nhưng bị chặn `403` khi sửa/xóa thực đơn; `admin` toàn quyền CRUD | `AdminWebPortalTest` | **PASS** |
| **TC-21** | Luồng tích hợp 8 bước trình diễn đầu-cuối (End-to-End Demo Workflow) (**AC-01..AC-11**) | Seed đầy đủ DB | Chạy xuyên suốt 8 bước từ khách xem món → đặt đơn → bếp nhận đơn → đổi trạng thái `PREPARING` → `READY` → khách polling thấy `READY` → hoàn tất `COMPLETED` | Toàn bộ 46 assertions trong luồng 8 bước đều đạt 100% | `EndToEndDemoWorkflowTest` | **PASS** |
