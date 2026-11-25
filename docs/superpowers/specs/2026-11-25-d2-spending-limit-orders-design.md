# Smart Canteen — Thiết kế Chi tiết Trục 2 · Slice D2: Hạn mức Chi tiêu Hằng ngày, Lịch sử Hạn mức, Thanh toán Trừ Hạn mức & Chi tiết Đơn Gắn Thẻ

- **Ngày:** 2026-11-25
- **Trạng thái:** **APPROVED 2026-11-25** bởi người dùng.
- **Nguồn yêu cầu:** [FEATURES.md](../../../FEATURES.md) (Mục 3.4 `LIMIT-01..13`, Mục 3.5 `ORDER-01..07`, Mục 3.6 `DETAIL-01..14`, Mục 7 Production-Ready Gaps: tự động làm mới hạn mức ngày, xử lý vượt hạn mức, hoàn tiền khi hủy đơn, quẹt thẻ POS mô phỏng) + Phương án đã chốt **`3A`**.
- **Môi trường mục tiêu:** Local / Development / Testing (mô phỏng trích nợ tự động qua `BankingGatewayInterface`, không kết nối ngân hàng thật).
- **Ký hiệu:** `[C]` CONFIRMED · `[I]` INFERRED · `[A]` ASSUMPTION (quyết định kỹ thuật của agent, có thể chỉnh sửa)

---

## 1. Mục tiêu Nghiệp vụ & Phạm vi Slice D2

### Mục tiêu (Business Objective)
`[C]` Nâng cấp phân hệ **Căn tin Số** sau khi thẻ học sinh đã liên kết ngân hàng (Slice D1), cho phép phụ huynh & hệ thống:
1. **Quản lý Hạn mức Chi tiêu Hằng ngày (`LIMIT-01..13`):** Xem hạn mức còn lại hôm nay & hạn mức hiện tại, thiết lập/thay đổi hạn mức tùy chỉnh hoặc chọn nhanh (`20k`, `30k`, `50k`, `100k`), tự động làm mới hạn mức mỗi ngày theo múi giờ `Asia/Ho_Chi_Minh`, và xem **Lịch sử thay đổi hạn mức**.
2. **Thanh toán Trừ Hạn mức An toàn (Đặt đơn Mini App + Mô phỏng Quẹt thẻ POS):**
   - Mở rộng luồng đặt đơn từ Giỏ hàng (`POST /api/v1/orders`) hỗ trợ chọn thanh toán bằng **Thẻ học sinh thông minh** (kiểm tra hạn mức còn lại, khóa dòng chống vượt hạn mức đồng thời, mô phỏng trích nợ tự động và ghi nhận hạn mức còn lại sau đơn).
   - Cung cấp endpoint mô phỏng **Quẹt thẻ học sinh tại quầy POS** (`POST /api/v1/canteen/cards/{cardId}/pos-purchases`) tạo đơn `#POS-YYMMDD-XXXXXX` thanh toán ngay để kiểm chứng trực tiếp luồng POS trong `FEATURES.md`.
   - Tự động **hoàn hạn mức chi tiêu trong ngày** khi Staff/Admin hủy một đơn thanh toán bằng thẻ học sinh (`status → CANCELLED`, `payment_status → REFUNDED`).
3. **Theo dõi Giao dịch & Chi tiết Đơn hàng Đầy đủ (`ORDER-01..07`, `DETAIL-01..14`):**
   - Hiển thị danh sách **Đơn hàng gần đây** gắn với thẻ ngay trên màn hình Căn tin Số + liên kết *Xem tất cả*.
   - Nâng cấp màn hình **Chi tiết đơn hàng** (`/orders/:id`) hiển thị đầy đủ 14 trường (`DETAIL-01..14`): mã đơn, trạng thái thanh toán, tổng thanh toán, học sinh & lớp, thời gian, cửa hàng/trường, phương thức thanh toán, danh sách món (đơn giá × số lượng), tạm tính, giảm giá, tổng cộng, và hạn mức còn lại sau giao dịch.

### Ngoài phạm vi Slice D2 (Dành cho Slice D3)
- Luồng Hủy liên kết ngân hàng qua xác nhận & OTP (`UNLINK-01..07`, `OTP-08`).
- Giao diện quản trị Học sinh / Thẻ / Liên kết / Hạn mức trên Laravel Blade Admin (`/admin/*`).
- Kiểm thử E2E xuyên suốt cả 4 luồng Căn tin Số.

### Anti-Goals
- **Không nhầm lẫn Hạn mức chi tiêu với Ví điện tử:** Không xây dựng chức năng nạp tiền ví; `daily_limit` là trần trích nợ tự động hằng ngày (`FEATURES.md` dòng 251).
- Không phá vỡ hợp đồng API cũ của `POST /api/v1/orders` (69 PHPUnit tests + 24 Vitest tests hiện có phải giữ nguyên 100% PASS).

---

## 2. Thiết kế Cơ sở Dữ liệu (PostgreSQL 17 — 3 Migrations Bổ sung)

> Tất cả migrations đều là **additive**, tương thích ngược 100% với các đơn hàng và bảng hiện có.

```mermaid
erDiagram
    STUDENT_CARDS ||--o| SPENDING_LIMITS : "cấu hình hạn mức (student_card_id)"
    STUDENT_CARDS ||--o{ SPENDING_LIMIT_LOGS : "lịch sử đổi hạn mức (student_card_id)"
    STUDENT_CARDS ||--o{ ORDERS : "thanh toán bằng thẻ (student_card_id)"
    USERS ||--o{ SPENDING_LIMIT_LOGS : "thay đổi bởi (changed_by)"

    SPENDING_LIMITS {
        bigint id PK
        bigint student_card_id FK UK "cascadeOnDelete"
        integer daily_limit "VND (>= 0, <= 5_000_000, default 0)"
        integer spent_today "VND (>= 0, default 0)"
        date spent_date "YYYY-MM-DD theo Asia/Ho_Chi_Minh"
        timestamp created_at
        timestamp updated_at
    }

    SPENDING_LIMIT_LOGS {
        bigint id PK
        bigint student_card_id FK "cascadeOnDelete"
        bigint changed_by FK "restrictOnDelete -> users.id"
        integer previous_limit "VND"
        integer new_limit "VND"
        varchar reason "nullable (<= 255)"
        timestamp created_at
    }

    ORDERS {
        bigint id PK
        bigint student_card_id FK "nullable, restrictOnDelete"
        varchar payment_method "COUNTER_CASH | STUDENT_CARD | STUDENT_CARD_POS"
        varchar payment_status "UNPAID | PAID | REFUNDED"
        varchar store_name "nullable (e.g. THCS Trần Phú)"
        integer discount_amount "VND (default 0)"
        integer remaining_limit_snapshot "nullable VND sau giao dịch"
        varchar payment_reference "nullable (EPay/Bank charge ref)"
    }
```

### Quy tắc Toàn vẹn & Reset Hạn mức Hằng ngày
1. **Tiền tệ nguyên dương `VND`:** Toàn bộ `daily_limit`, `spent_today`, `previous_limit`, `new_limit`, `discount_amount`, `remaining_limit_snapshot` dùng kiểu `unsignedInteger` (không dùng `float`).
2. **Tự động làm mới hạn mức mỗi ngày (`Asia/Ho_Chi_Minh`) không cần Cron:**
   - Ngày nghiệp vụ hiện tại: `$today = now('Asia/Ho_Chi_Minh')->toDateString()`.
   - Khi đọc (`GET`): `effective_spent_today = ($limit->spent_date === $today) ? $limit->spent_today : 0`; `remaining_limit = max(0, $limit->daily_limit - $effective_spent_today)`.
   - Khi trừ hoặc hoàn hạn mức trong `DB::transaction()` + `lockForUpdate()` trên `spending_limits`: nếu `$limit->spent_date !== $today`, đặt lại `spent_today = 0, spent_date = $today` trước khi cộng `total_amount`.
   - Khi phụ huynh giảm `daily_limit` xuống thấp hơn `spent_today`: vẫn cho phép cập nhật, `remaining_limit` hiển thị `0 đ` (không âm) và chặn các giao dịch mới trong ngày hôm đó.
3. **Thứ tự khóa dòng chuẩn (Chống AB-BA Deadlock):**
   - Mọi giao dịch liên quan đến thẻ + hạn mức + sản phẩm tuân thủ thứ tự khóa cố định:
     **`student_cards` → `spending_limits` → `products` → `orders`**.
4. **Hoàn hạn mức khi hủy đơn:**
   - Trong `OrderService::transitionStatus()`: khi đơn hàng có `student_card_id !== null` và `payment_status === PAID` chuyển sang `CANCELLED`:
     - Đổi `payment_status = REFUNDED`.
     - Nếu đơn được tạo trong ngày hôm nay (`$order->created_at->timezone('Asia/Ho_Chi_Minh')->toDateString() === $today`), khóa `spending_limits` của thẻ đó và giảm `spent_today = max(0, $limit->spent_today - $order->total_amount)`.

---

## 3. Ranh giới Tích hợp Ngân hàng & Dịch vụ Backend

### 3.1. Mở rộng `BankingGatewayInterface` & `MockEpayBankingGateway`
Thêm 2 phương thức gọn nhẹ (additive):
- `chargeAutoDebit(string $providerReference, int $amountVnd, string $orderCode): array` → trả `['charged' => true, 'charge_reference' => 'EPAY-CHG-...']`.
- `refundAutoDebit(string $chargeReference, int $amountVnd, string $orderCode): array` → trả `['refunded' => true, 'refund_reference' => 'EPAY-REF-...']`.

### 3.2. Các Thành phần mới trong `api/`
- **Enums:** `App\Enums\PaymentMethod` (`COUNTER_CASH`, `STUDENT_CARD`, `STUDENT_CARD_POS` kèm hàm `label()` tiếng Việt), `App\Enums\PaymentStatus` (`UNPAID`, `PAID`, `REFUNDED` kèm `label()`).
- **Models:** `SpendingLimit`, `SpendingLimitLog`; mở rộng `StudentCard` (quan hệ `spendingLimit()`, `spendingLimitLogs()`, `orders()`) và `Order` (quan hệ `studentCard()`, các cột thanh toán mới).
- **Exceptions:** `SpendingLimitExceededException` (`422` + `error_code = SPENDING_LIMIT_EXCEEDED` + `meta: { daily_limit, spent_today, remaining_limit, required_amount }`), `CardNotLinkedException` (`422` + `error_code = CARD_NOT_BANK_LINKED`).
- **Services:**
  - `App\Services\SpendingLimitService`: `getSummary()`, `updateDailyLimit()`, `listLogs()`, `consumeLimit()`, `refundLimitForOrder()`.
  - Mở rộng `App\Services\OrderService`:
    - `createOrder(..., ?int $studentCardId = null, ?PaymentMethod $paymentMethod = null)` — giữ tham số mặc định `null` để 100% tương thích ngược.
    - `createPosPurchase(User $user, int $cardId, array $items, ?string $storeName, string $idempotencyKey)` — tạo đơn `#POS-YYMMDD-XXXXXX` trạng thái `COMPLETED`, `payment_method = STUDENT_CARD_POS`, `payment_status = PAID`, kiểm tra & trừ hạn mức atomic + hỗ trợ `Idempotency-Key`.
    - `transitionStatus(...)` — tự động hoàn hạn mức + đổi `payment_status = REFUNDED` khi hủy đơn đã thanh toán bằng thẻ.

---

## 4. Hợp đồng REST API (`/api/v1/*`)

### 4.1. Mở rộng `StudentCardResource` (`GET /api/v1/canteen/cards`, `GET /api/v1/canteen/cards/{cardId}`)
Bổ sung additive object `spending_limit`:
```json
{
  "spending_limit": {
    "daily_limit": 150000,
    "spent_today": 40000,
    "remaining_limit": 110000,
    "effective_date": "2026-11-25"
  }
}
```
*(Nếu thẻ chưa từng thiết lập hạn mức: `daily_limit: 0, spent_today: 0, remaining_limit: 0`.)*

### 4.2. `PUT /api/v1/canteen/cards/{cardId}/spending-limit` (`auth:sanctum`)
Thiết lập / thay đổi hạn mức chi tiêu hằng ngày (`LIMIT-03..12`):
- **Request Body:** `{ "daily_limit": 100000, "reason": "Tăng hạn mức bữa trưa" }`
  - `daily_limit`: bắt buộc, số nguyên `0..5000000` (bội số `1000` khi `> 0`, cho phép `0` để tạm chặn chi tiêu).
  - `reason`: tùy chọn, chuỗi tối đa `255` ký tự.
- **Quy tắc:** `404` nếu thẻ thuộc người khác; `422` nếu thẻ không `ACTIVE` hoặc chưa liên kết ngân hàng `ACTIVE` (`CARD_NOT_BANK_LINKED`).
- **Response `200 OK`:** `{ "data": StudentCardResource }` (kèm `spending_limit` mới). Ghi 1 bản ghi vào `spending_limit_logs` nếu giá trị thay đổi.

### 4.3. `GET /api/v1/canteen/cards/{cardId}/limit-logs` (`auth:sanctum`)
Lịch sử thay đổi hạn mức (`LIMIT-13`), phân trang `per_page` (`1..50`, mặc định `10`), sắp xếp mới nhất trước (`id DESC`):
```json
{
  "data": [
    {
      "id": 3,
      "previous_limit": 50000,
      "new_limit": 100000,
      "reason": null,
      "changed_by_name": "Khách hàng Demo",
      "created_at": "2026-11-25T10:15:00+00:00"
    }
  ],
  "meta": { "current_page": 1, "last_page": 1, "per_page": 10, "total": 1 }
}
```

### 4.4. `GET /api/v1/canteen/cards/{cardId}/orders` (`auth:sanctum`)
Danh sách đơn hàng gắn với thẻ học sinh (`ORDER-01..07`), phân trang `per_page` (`1..50`, mặc định `4` cho khối *Đơn hàng gần đây* trên màn hình Căn tin Số).

### 4.5. `POST /api/v1/canteen/cards/{cardId}/pos-purchases` (`auth:sanctum`)
Mô phỏng học sinh quẹt thẻ mua hàng tại quầy POS (`ORDER-01..07`, `DETAIL-01..14`):
- **Headers:** Bắt buộc `Idempotency-Key` (`1..100` ký tự).
- **Request Body:** `{ "items": [{ "product_id": 1, "quantity": 1 }], "store_name": "THCS Trần Phú" }`
- **Response `201 Created` (hoặc `200 OK` khi idempotent replay):** `{ "data": OrderResource }` với mã `#POS-YYMMDD-XXXXXX`, `status: "COMPLETED"`, `payment_method: "STUDENT_CARD_POS"`, `payment_status: "PAID"`, `remaining_limit_snapshot`.
- **Lỗi:** `404` (thẻ người khác), `409` (`Idempotency-Key` khác payload), `422` (`CARD_NOT_BANK_LINKED`, `SPENDING_LIMIT_EXCEEDED`, hoặc món ngừng bán).

### 4.6. Mở rộng `POST /api/v1/orders` & `OrderResource` (100% Backward Compatible)
- `POST /api/v1/orders`: chấp nhận thêm `student_card_id` (`nullable|integer`) và `payment_method` (`nullable|in:COUNTER_CASH,STUDENT_CARD`). Nếu không truyền → mặc định `COUNTER_CASH` / `UNPAID` như MVP cũ. Nếu truyền `student_card_id` → kiểm tra thẻ & trừ hạn mức ngày + chốt `payment_status = PAID` & `remaining_limit_snapshot`.
- `OrderResource`: bổ sung additive:
  - `student_card_id`, `card_code`, `student_name`, `student_class`
  - `store_name`, `payment_method`, `payment_method_label`, `payment_status`, `payment_status_label`
  - `subtotal_amount`, `discount_amount`, `remaining_limit_snapshot`

---

## 5. Thiết kế Giao diện Mini App (`DESIGN.md` & WCAG 2.1 AA)

1. **`CanteenHubPage` (`/canteen`) — Nâng cấp trạng thái `Linked`:**
   - **Khối Hạn mức chi tiêu hằng ngày (`LIMIT-01..02`):**
     - Hiển thị nổi bật *Hạn mức còn lại hôm nay* (`110.000 ₫`), *Hạn mức ngày* (`150.000 ₫/ngày`), *Đã chi hôm nay* (`40.000 ₫`), thanh tiến độ màu `brand-700` (chuyển sang `amber-500`/`red-600` khi còn `< 20%`).
     - 2 nút thao tác: **"Thiết lập hạn mức"** (mở `SpendingLimitSheet`) và **"Lịch sử hạn mức"** (`/canteen/limit-history`).
   - **Khối Đơn hàng gần đây (`ORDER-01..07`):**
     - Tiêu đề *"Đơn hàng gần đây"* kèm nút *"Xem tất cả"* (`/orders`).
     - Tối đa 4 đơn gắn thẻ gần nhất (tên món đầu tiên / giao dịch, địa điểm căn tin, thời gian, số tiền `-25.000 ₫`), chạm vào mở `/orders/:id`.
     - Nút nhỏ **"Mô phỏng quẹt thẻ POS (Demo)"** mở hộp chọn nhanh 1 món đang bán để demo tức thời luồng quẹt thẻ `#POS-...` & trừ hạn mức ngay trên giao diện!
2. **`SpendingLimitSheet` (`LIMIT-03..12`):**
   - Dialog/Bottom Sheet accessible (`role="dialog"`, `aria-modal="true"`, phím `Escape` đóng, focus vào ô nhập khi mở).
   - Hiển thị *Hạn mức hiện tại* (`X ₫/ngày`), ô nhập *Hạn mức mới (VNĐ)* (`inputMode="numeric"`), 4 nút chọn nhanh **`20k` · `30k` · `50k` · `100k`**, validation inline (`0..5.000.000 đ`, bội số `1.000 đ`), nút **Hủy** & **Lưu hạn mức** (guard `isPending`).
3. **`LimitHistoryPage` (`/canteen/limit-history` — `LIMIT-13`):**
   - Danh sách phân trang (`PaginationBar`) hiển thị từng lần thay đổi hạn mức (từ `X ₫` → `Y ₫`, người thay đổi, thời gian) với đủ 4 trạng thái `Loading`/`Error`/`Empty`/`Success`.
4. **`CartPage` (`/cart`):**
   - Nếu có thẻ học sinh đã liên kết ngân hàng: hiển thị lựa chọn phương thức thanh toán (**Thẻ học sinh thông minh — Trích nợ tự động** kèm hạn mức còn lại hôm nay vs **Thanh toán tại quầy**). Nếu chọn thẻ học sinh mà tổng giỏ vượt hạn mức còn lại, hiển thị cảnh báo rõ ràng + nút chuyển nhanh sang `/canteen` để nâng hạn mức.
5. **`OrderDetailPage` (`/orders/:id` — `DETAIL-01..14`):**
   - Bổ sung trọn bộ thông tin hóa đơn: trạng thái thanh toán (`Đã thanh toán` / `Đã hoàn tiền` / `Chờ thanh toán`), thông tin học sinh & lớp, thời gian định dạng đầy đủ `HH:mm, dd/MM/yyyy`, cửa hàng/trường (`THCS Trần Phú`), phương thức thanh toán (`Quẹt thẻ học sinh` / `Thẻ học sinh (Mini App)` / `Thanh toán tại quầy`), tạm tính, giảm giá (`0 ₫`), tổng cộng, và thẻ thông tin **Hạn mức còn lại sau đơn hàng** (`remaining_limit_snapshot`).

---

## 6. Chiến lược Kiểm thử (TDD)

- **Backend (PHPUnit trên PostgreSQL 17):**
  - `SpendingLimitAndOrderPaymentTest`:
    1. Chặn thiết lập hạn mức khi thẻ chưa liên kết ngân hàng (`422 CARD_NOT_BANK_LINKED`) hoặc thẻ thuộc user khác (`404`).
    2. Thiết lập & cập nhật hạn mức (`20k`/`100k`/tùy chỉnh/`0`), kiểm chứng validate sai (`< 0`, `> 5_000_000`, không chia hết cho `1000`), kiểm chứng ghi `spending_limit_logs` và phân trang `GET /limit-logs`.
    3. Tự động làm mới `spent_today = 0` khi qua ngày mới theo `Asia/Ho_Chi_Minh` (`travel(1)->day()`).
    4. Đặt đơn `POST /api/v1/orders` với `student_card_id`: trừ đúng `spent_today`, lưu `payment_status = PAID`, `remaining_limit_snapshot`, `student_name`, `student_class`, `store_name`; chặn `422 SPENDING_LIMIT_EXCEEDED` khi vượt hạn mức còn lại.
    5. Mô phỏng quẹt thẻ POS `POST /api/v1/canteen/cards/{cardId}/pos-purchases`: mã đơn `#POS-...`, `status = COMPLETED`, `payment_method = STUDENT_CARD_POS`, trừ hạn mức, hỗ trợ `Idempotency-Key` (`200` replay / `409` khác payload).
    6. Hủy đơn đã thanh toán bằng thẻ (`PATCH /api/v1/admin/orders/{id}/status` → `CANCELLED`): tự động hoàn `spent_today` và cập nhật `payment_status = REFUNDED`.
    7. Kiểm chứng toàn bộ 69 bài test cũ vẫn PASS 100% (không hồi quy).
- **Frontend (Vitest + TypeScript + Vite Build):**
  - Unit tests `spending-limit.test.ts` (validate input hạn mức, chọn nhanh presets `20k/30k/50k/100k`, kiểm tra đủ/vượt hạn mức giỏ hàng).

---

## 7. Những Quyết định Nhỏ đã Đề xuất (Calls Made by Agent)

1. `[A]` Dùng cơ chế lazy daily reset theo cột `spent_date` (múi giờ `Asia/Ho_Chi_Minh`) thay vì cronjob nền — đảm bảo chính xác tức thời ngay tại ranh giới `00:00` và không cần thêm worker process.
2. `[A]` Cho phép `daily_limit = 0` (chặn chi tiêu tạm thời) và giới hạn trần `5.000.000 ₫/ngày`, bước `1.000 ₫` khi `> 0`.
3. `[A]` Giữ `student_card_id` là tùy chọn trên `POST /api/v1/orders` (mặc định `COUNTER_CASH`) để 100% tương thích ngược với luồng đặt món MVP cũ.
4. `[A]` Cung cấp nút *"Mô phỏng quẹt thẻ POS"* ngay trên màn hình Căn tin Số để trình diễn đầy đủ luồng hóa đơn `#POS-...` trong `FEATURES.md` mà không cần phần cứng máy POS thật.
