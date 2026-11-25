# Đặc tả Thiết kế Chi tiết — Trục 3 · Slice P3: POS Quầy Căn tin (Staff), Quản lý Tồn kho (`stock_quantity`), CRUD Học sinh & Thẻ trên Admin và RBAC 4 Vai trò

- **Ngày lập:** 2026-11-25
- **Mã tác vụ:** `CANTEEN-P3-POS-INVENTORY-STUDENTS-RBAC`
- **Trạng thái:** `APPROVED (2026-11-25)`
- **Nền tảng cơ sở (`main`):** Sau khi đã merge cục bộ Trục 3 · Slice P2 (`root @ 120e480`, `api @ 3ab9af5`, `miniapp @ 7a2d4e5`).
- **Nguồn đối chiếu:** [2026-11-25-truc3-production-readiness-roadmap.md](file:///d:/Coding/TitKul_ProjectTest/TestPart2/docs/superpowers/specs/2026-11-25-truc3-production-readiness-roadmap.md) (`APPROVED 2026-11-25`), `FEATURES.md` (Mục 7 #10, #11, #12, #13 & Mục 9), `AGENTS.md`.
- **Phạm vi bao gồm:** 4 trụ cột của Slice P3 + 2 điểm Deferred Minors từ Slice P2. **Ngoài phạm vi:** Notification Module In-App/Zalo OA (Slice P4), triển khai production/push remote.

---

## 1. Mục tiêu & Kiến trúc Tổng thể

Slice P3 đưa phân hệ vận hành tại căn tin và quản trị học sinh lên chuẩn **Production-Ready** với 4 năng lực cốt lõi + 2 cải tiến nhỏ từ Slice P2:

1. **Quản lý Tồn kho Sản phẩm (`stock_quantity`) nguyên tử chống bán vượt kho (Oversell Prevention):**
   - Hỗ trợ đồng thời món **không giới hạn tồn kho** (`stock_quantity = null`, giữ 100% tương thích ngược với dữ liệu/test cũ) và món **theo dõi tồn kho chặt** (`stock_quantity >= 0`).
   - Trừ tồn kho nguyên tử trong cùng `DB::transaction()` khi đặt đơn (Mini App Cart, Mini App POS, Staff POS Terminal); tự động rollback nếu thanh toán ngân hàng/hạn mức lỗi; tự động **hoàn lại tồn kho** khi đơn bị hủy (`CANCELLED`).
2. **Màn hình POS Quầy Căn tin chuyên dụng cho Nhân viên (`/admin/pos`):**
   - Giao diện 2 cột tốc độ cao dành cho `staff` và `admin`: quét/nhập mã thẻ học sinh (`F156BA`) hoặc mã học sinh → xem tức thì tên học sinh, trạng thái thẻ, hạn mức chi tiêu còn lại hôm nay (tuyệt đối không lộ STK/CCCD) → chọn nhiều món đang còn hàng → bấm thanh toán **Quẹt thẻ học sinh** (`STUDENT_CARD_POS`) hoặc **Tiền mặt tại quầy** (`COUNTER_CASH`) với chống bấm đúp (`Idempotency-Key`) → xuất đơn `#POS-YYMMDD-XXXXXX` và **Biên lai POS có thể in**.
3. **CRUD Học sinh & Cấp phát Thẻ trên Admin Blade (`/admin/canteen/students`):**
   - Cho phép `admin` thêm mới/chỉnh sửa hồ sơ học sinh, gắn hoặc chuyển nhiều học sinh cho cùng 1 tài khoản phụ huynh (`students.user_id`), gắn tài khoản học sinh (`students.student_user_id`), và **cấp phát thẻ mới / cấp lại thẻ** ngay trong giao diện quản trị (`staff` chỉ xem danh sách).
4. **Mở rộng RBAC 4 Vai trò (`admin`, `staff`, `parent`, `student`) tương thích ngược 100%:**
   - Thêm vai trò `parent` và `student` vào `UserRole` (giữ `customer` làm alias tương đương `parent` để không phá vỡ API contract & 114 test hiện có).
   - Bổ sung `students.student_user_id` cho phép tài khoản `student` đăng nhập Mini App để xem thẻ, hạn mức còn lại, lịch sử giao dịch và mua hàng trong hạn mức, nhưng bị chặn nghiêm ngặt `403 PARENT_ROLE_REQUIRED` ở mọi thao tác tài chính của phụ huynh (liên kết/hủy liên kết ngân hàng, thay đổi hạn mức).
5. **Xử lý 2 Deferred Minors từ Slice P2:**
   - Sinh sẵn `gateway_txn_ref` trước khi gọi `chargeAutoDebit()` và lưu vào cả bản ghi `SUCCEEDED` lẫn `FAILED`/`TIMEOUT` để đối soát chính xác giao dịch timeout với sao kê ngân hàng.
   - Hiển thị badge trạng thái thanh toán trên danh sách thẻ của `OrdersPage` và hỗ trợ điều hướng phím mũi tên (WAI-ARIA) trên `role="radiogroup"` của `CartPage`.

---

## 2. Phương án Thiết kế Đã Cân nhắc (Approaches)

| Quyết định | Phương án chọn | Phương án loại bỏ & Lý do |
|---|---|---|
| **Mô hình tồn kho (`stock_quantity`)** | **Nullable integer (`null` = không giới hạn, `>= 0` = trừ kho nghiêm ngặt)** + `low_stock_threshold` (mặc định `5`). | *Bắt buộc NOT NULL mặc định 0 hoặc 999:* Gây lỗi cho món chế biến theo yêu cầu (cooked-to-order) và dễ phá vỡ các test hiện có tạo `Product` không truyền `stock_quantity`. |
| **RBAC 4 vai trò vs tương thích `customer`** | Giữ `UserRole::CUSTOMER = 'customer'` làm legacy alias tương đương `UserRole::PARENT = 'parent'`; thêm `UserRole::STUDENT = 'student'`, thêm helper `isParentEquivalent()` & trả kèm `effective_role` / `can_manage_canteen` trong `UserResource`. | *Đổi toàn bộ `customer` → `parent` bằng data migration phá vỡ:* Làm hỏng hợp đồng API hiện hành và các test xác thực cũ đang kiểm chứng `'customer'`. |
| **Liên kết tài khoản Học sinh** | Thêm cột nullable `UNIQUE` `students.student_user_id` (`FK → users.id`, `nullOnDelete`), tách bạch rõ `students.user_id` (Phụ huynh, 1-N) và `students.student_user_id` (Học sinh, 1-1). | *Tạo bảng trung gian N-N:* Dư thừa vì 1 hồ sơ học sinh thuộc đúng 1 phụ huynh giám hộ chính và tối đa 1 tài khoản học sinh. |
| **POS Staff Terminal UI** | Server-rendered **Laravel Blade + Vanilla JS tối giản** tại `/admin/pos` dùng chung session & CSRF của Admin Blade + JSON endpoint `/admin/pos/lookup-card` & `POST /admin/pos/checkout`. | *Dựng SPA React riêng cho POS:* Tăng gấp đôi độ phức tạp build/deploy, không cần thiết khi Admin Blade đã ổn định, nhanh và sẵn RBAC `role:staff,admin`. |

---

## 3. Thiết kế Cơ sở Dữ liệu (100% Additive Migrations)

### 3.1. `2026_11_25_500001_add_inventory_columns_to_products_table.php`
- Bảng `products`:
  - `stock_quantity` (`unsignedInteger`, `nullable`, default `null`) — `null`: không giới hạn; `0`: hết hàng; `> 0`: số phần còn lại.
  - `low_stock_threshold` (`unsignedInteger`, default `5`) — ngưỡng cảnh báo sắp hết hàng trên Admin Blade.
  - Index: `products_available_stock_idx` trên `(is_available, stock_quantity)`.

### 3.2. `2026_11_25_500002_add_student_user_id_to_students_table.php`
- Bảng `students`:
  - `student_user_id` (`foreignId`, `nullable`, `unique`, `constrained('users')->nullOnDelete()`) — tài khoản Mini App riêng của học sinh (role `student`), 1-1 với hồ sơ `students`.

### 3.3. Thứ tự Khóa Chuẩn PostgreSQL (Canonical Lock Order — Bắt buộc tuân thủ mọi luồng)
```
student_cards → spending_limits → sandbox_bank_accounts → products (ORDER BY id ASC) → orders → payment_transactions
```
- **Tạo đơn (`createOrder`, `createPosPurchase`, `createStaffPosOrder`):** khóa `student_cards` → `spending_limits` → `sandbox_bank_accounts` → `products` (trừ `stock_quantity`) → tạo `orders` → ghi `payment_transactions`. Nếu lỗi bất kỳ bước nào (`INSUFFICIENT_STOCK`, `SPENDING_LIMIT_EXCEEDED`, `INSUFFICIENT_BANK_BALANCE`, `BANK_DECLINED`, `GATEWAY_TIMEOUT`), PostgreSQL rollback trọn vẹn → **tồn kho, hạn mức và số dư ngân hàng không bị lệch**.
- **Hủy đơn (`transitionStatus(..., CANCELLED)`):** đọc danh sách `product_id` (sắp xếp `ASC`) trước giao dịch → khóa `student_cards` → `spending_limits` → `sandbox_bank_accounts` → `products (ORDER BY id ASC)` → `orders` → hoàn tồn kho (`stock_quantity += quantity` nếu `stock_quantity !== null`), hoàn hạn mức & hoàn số dư sandbox + ghi `REFUND` ledger.

---

## 4. Hợp đồng API, Routes & Ma trận RBAC 4 Vai trò

### 4.1. Ma trận Phân quyền 4 Vai trò

| Vai trò | `/admin/*` (Dashboard, Orders, POS, Read Lists) | `/admin/*` (CRUD Menu/Stock/Students/Cards, Reconcile Run/Resolve, Sandbox Config) | Mini App Public (`/`, `/products`, `/cart`) | Mini App Căn tin: Xem thẻ, Hạn mức, Lịch sử, Mua hàng | Mini App Căn tin: Liên kết/Hủy liên kết NH OTP, Sửa Hạn mức |
|---|---|---|---|---|---|
| **`admin`** | Có | **Có (Toàn quyền)** | Có | Chỉ thẻ của mình (nếu có) | **403** (quản lý qua Admin Blade) |
| **`staff`** | **Có** (bao gồm `/admin/pos`) | **403 Forbidden** | Có | Chỉ thẻ của mình (nếu có) | **403** (chỉ vai trò Phụ huynh) |
| **`parent` / `customer`** | **403 Forbidden** | **403 Forbidden** | Có | **Có** (mọi học sinh `students.user_id = me`) | **Có** (toàn quyền tài chính) |
| **`student`** | **403 Forbidden** | **403 Forbidden** | Có | **Có** (chỉ hồ sơ `students.student_user_id = me`) | **403 `PARENT_ROLE_REQUIRED`** |

- **`UserResource` mở rộng (additive):**
  - Giữ nguyên `role` (`admin` | `staff` | `parent` | `student` | `customer`).
  - Thêm `effective_role` (`admin` | `staff` | `parent` | `student`, quy đổi `customer → parent`) và `can_manage_canteen` (`bool`: `true` với `parent`/`customer`, `false` với `student`).
- **`EnsureUserHasRole` middleware:**
  - Nếu danh sách role cho phép chứa `parent`, tự động chấp nhận cả `customer` (legacy alias).
  - Các endpoint tài chính phụ huynh (`POST /cards/{id}/bank-links`, `verify-otp`, `resend-otp`, `unlink/*`, `PUT /cards/{id}/spending-limit`) gắn middleware `role:parent` (hoặc kiểm tra chặt trong controller/service) trả JSON `403` với `error_code: "PARENT_ROLE_REQUIRED"` khi tài khoản `student` gọi tới.
- **IDOR & Scope cho `student`:**
  - `StudentCardService::listForUser()` / `findForUser()` và `SpendingLimitService::listLogs()` / `listCardOrders()` / `OrderService::createOrder()` / `createPosPurchase()` mở rộng điều kiện sở hữu:
    - Với `parent`/`customer`: `students.user_id = $user->id AND students.is_active = true`.
    - Với `student`: `students.student_user_id = $user->id AND students.is_active = true`.
  - Học sinh xem được lịch sử đơn gắn thẻ của mình (`GET /api/v1/orders` và `GET /api/v1/orders/{id}` cho phép xem đơn có `order.user_id = $user->id` HOẶC `order.student_card_id` thuộc thẻ của học sinh đó!).

### 4.2. Routes Admin Blade mới

| Method & URI | Middleware | Chức năng |
|---|---|---|
| `GET /admin/pos` | `auth`, `role:staff,admin` | Màn hình POS Quầy Căn tin (tra cứu thẻ + lưới món có tồn kho + giỏ POS + lịch sử đơn POS gần đây trong ngày). |
| `GET /admin/pos/lookup-card?q=...` | `auth`, `role:staff,admin` | JSON API tra cứu thẻ theo `card_code` hoặc `student_code` (trả về tên HS, lớp, trường, trạng thái thẻ, `is_bank_linked`, `daily_limit`, `spent_today`, `remaining_limit`; **không trả STK/CCCD**). |
| `POST /admin/pos/checkout` | `auth`, `role:staff,admin` | Tạo đơn POS (`STUDENT_CARD_POS` hoặc `COUNTER_CASH`), hỗ trợ `idempotency_key`, trả JSON kết quả + URL biên lai hoặc lỗi `422`/`502`/`504` chi tiết. |
| `GET /admin/pos/receipt/{order}` | `auth`, `role:staff,admin` | Biên lai POS thân thiện máy in (`@media print`). |
| `GET /admin/canteen/students` | `auth`, `role:staff,admin` | Danh sách học sinh (tìm kiếm, lọc trạng thái, số thẻ, tên & SĐT phụ huynh, tài khoản học sinh). |
| `GET /admin/canteen/students/create` & `POST /admin/canteen/students` | `auth`, `role:admin` | Tạo học sinh mới + gắn phụ huynh (`user_id`) + tùy chọn gắn tài khoản `student` + tùy chọn phát hành thẻ đầu tiên ngay. |
| `GET /admin/canteen/students/{student}/edit` & `PUT /admin/canteen/students/{student}` | `auth`, `role:admin` | Cập nhật hồ sơ học sinh, chuyển phụ huynh, gắn tài khoản học sinh, bật/tắt `is_active`. |
| `POST /admin/canteen/students/{student}/cards` | `auth`, `role:admin` | Cấp phát thẻ học sinh mới cho học sinh (`card_code` tùy chỉnh hoặc tự sinh 6 ký tự hex in hoa). |
| `DELETE /admin/canteen/students/{student}` | `auth`, `role:admin` | Xóa học sinh nếu chưa phát sinh thẻ nào; chặn kèm thông báo hướng dẫn ngừng kích hoạt nếu đã có thẻ. |
| `PATCH /admin/products/{product}/stock` | `auth`, `role:admin` | Nhập thêm / đặt lại tồn kho nhanh ngay trên danh sách món ăn. |

### 4.3. Mở rộng API Public/Customer (`api/routes/api.php`)
- `ProductResource` bổ sung `stock_quantity` (`int|null`), `low_stock_threshold` (`int`), `is_in_stock` (`bool`).
- `POST /api/v1/orders` và `POST /api/v1/canteen/cards/{cardId}/pos-purchases` trả `422` với `error_code: "INSUFFICIENT_STOCK"` + `meta` khi vượt tồn kho.
- `POST /api/v1/auth/demo` / `POST /api/v1/auth/demo-phone`: `StudentCardSeeder` bổ sung tài khoản demo học sinh (`phone: 0901234568`, `zalo_user_id: demo_student1`, `role: student`) gắn vào `Titkul Student` (`HS-2026-0001`, thẻ `F156BA`) để kiểm thử ngay 2 vai trò Phụ huynh (`0901234567`) và Học sinh (`0901234568`).

---

## 5. Thiết kế Frontend Mini App (`miniapp/`)

1. **Tồn kho trên Thực đơn, Chi tiết Món & Giỏ hàng:**
   - Cập nhật `Product` type với `stock_quantity?: number | null`, `low_stock_threshold?: number`, `is_in_stock?: boolean`.
   - `HomePage` / `ProductDetailPage`: hiển thị badge *"Còn X phần"* (màu trung tính/hổ phách khi `<= low_stock_threshold`) và *"Hết hàng"* (khóa nút thêm vào giỏ khi `stock_quantity === 0` hoặc `!is_available`).
   - `CartPage`: giới hạn số lượng tăng tối đa `Math.min(MAX_ITEM_QUANTITY, product.stock_quantity)` khi `stock_quantity !== null`; cập nhật `formatPaymentError()` xử lý `INSUFFICIENT_STOCK` rõ ràng.
2. **Nhận diện Vai trò `student` trên Mini App:**
   - Cập nhật `AuthUser` hỗ trợ `role: "customer" | "parent" | "student" | "staff" | "admin"`, `effective_role`, `can_manage_canteen`.
   - `CanteenHubPage`: khi `can_manage_canteen === false` (tài khoản `student`), hiển thị thẻ học sinh, hạn mức còn lại hôm nay, lịch sử hạn mức & đơn hàng gần đây, kèm banner **"Tài khoản Học sinh — Liên hệ phụ huynh để liên kết ngân hàng hoặc thay đổi hạn mức"**, đồng thời ẩn các nút thao tác tài chính của phụ huynh (backend vẫn enforce `403` độc lập!).
   - `ZaloPhoneLoginPanel` / `ZaloPhoneLoginModal`: bổ sung nút điền nhanh SĐT demo thứ hai (`0901234568 — Học sinh Demo`) bên cạnh (`0901234567 — Phụ huynh Demo`) khi `VITE_DEMO_PHONE_LOGIN=true` ở môi trường local/dev.
3. **2 Deferred Minors từ Slice P2:**
   - `OrdersPage`: hiển thị badge trạng thái thanh toán (`getPaymentBadgeClass` + `payment_status_label`) cạnh trạng thái xử lý đơn.
   - `CartPage`: thêm sự kiện `onKeyDown` hỗ trợ phím `ArrowUp`/`ArrowDown`/`ArrowLeft`/`ArrowRight` di chuyển giữa các radio phương thức thanh toán trong `role="radiogroup"`.

---

## 6. Tiêu chí Nghiệm thu (Acceptance Criteria — Slice P3)

1. **INV-01 (Tồn kho nguyên tử):** Đặt đơn (Cart/POS/Staff POS) giảm đúng `stock_quantity`; vượt tồn kho trả `422 INSUFFICIENT_STOCK` và không tạo đơn; lỗi thanh toán ngân hàng rollback không làm giảm tồn kho.
2. **INV-02 (Hoàn kho khi hủy):** Chuyển đơn sang `CANCELLED` hoàn trả đúng `stock_quantity` cho các món theo dõi tồn kho, đồng thời vẫn hoàn hạn mức và số dư ngân hàng theo đúng thứ tự khóa chuẩn không deadlock.
3. **INV-03 (Admin & Mini App Inventory):** Admin CRUD sửa được `stock_quantity`/`low_stock_threshold`, có lọc sắp hết hàng/hết hàng và nhập kho nhanh; Mini App hiển thị tồn kho và chặn thêm quá tồn kho.
4. **POS-01 (Staff POS Terminal):** `staff` và `admin` tra cứu thẻ học sinh tại `/admin/pos` thấy đúng hạn mức còn lại hôm nay (không lộ STK/CCCD), thanh toán nhiều món tạo đơn `#POS-...` (`COMPLETED`/`PAID`), trừ tồn kho + hạn mức + số dư sandbox + ghi sổ cái `payment_transactions`, và hiển thị biên lai in được.
5. **POS-02 (POS Idempotency & Error Handling):** Gửi trùng `idempotency_key` không tạo đơn thứ 2 hay trừ tiền lần 2; các lỗi thẻ khóa/chưa liên kết/hết kho/vượt hạn mức/thiếu số dư ngân hàng trả thông báo rõ ràng.
6. **STU-01 (Admin Student & Card CRUD):** `admin` tạo mới học sinh, gắn nhiều học sinh cho 1 phụ huynh, sửa hồ sơ, cấp phát thẻ mới, chặn xóa học sinh đã có thẻ; `staff` xem được danh sách nhưng nhận `403` cho mọi thao tác thêm/sửa/xóa/cấp thẻ.
7. **RBAC-01 (4 Vai trò):** Tài khoản `student` xem được thẻ, hạn mức, lịch sử hạn mức & đơn của chính mình và đặt mua trong hạn mức; bị chặn `403 PARENT_ROLE_REQUIRED` khi cố liên kết/hủy liên kết ngân hàng hoặc sửa hạn mức, bị chặn `403` trên `/admin/*`; tài khoản `parent` và `customer` giữ nguyên 100% quyền cũ.
8. **P2-MINOR-01 & P2-MINOR-02:** `gateway_txn_ref` được sinh trước khi trừ tiền ngân hàng và lưu bền vững cả khi `TIMEOUT` để đối soát; `OrdersPage` hiển thị badge thanh toán và `CartPage` radiogroup hỗ trợ điều hướng phím mũi tên.
9. **Quality Gates:** `vendor/bin/pint --test` PASS, `php artisan test --compact` PASS (toàn bộ 114 test cũ + bộ test mới của Slice P3), `migrate:fresh --force --seed` PASS, `npm test` + `npx tsc --noEmit` + `npm run build` PASS, Independent Code Review sạch Critical/Important.

---

## 7. Đề xuất Kiểm chứng & Câu hỏi Xác nhận

Đặc tả trên đã bao quát đầy đủ 4 trụ cột Slice P3 + 2 Deferred Minors của Slice P2 theo đúng lộ trình Trục 3 đã duyệt, đồng thời bảo toàn 100% tương thích ngược. Nếu bạn đồng ý với thiết kế chi tiết này, hãy phản hồi **Phê duyệt (Approve)** để tôi lập kế hoạch chi tiết (`superpowers:writing-plans`) và triển khai trọn vẹn Slice P3 (implementation → testing → independent code review → verification → handoff).
