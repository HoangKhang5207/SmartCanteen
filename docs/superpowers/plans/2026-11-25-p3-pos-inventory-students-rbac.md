# Kế hoạch Triển khai Chi tiết — Trục 3 · Slice P3 (`CANTEEN-P3-POS-INVENTORY-STUDENTS-RBAC`)

- **Ngày lập:** 2026-11-25
- **Đặc tả nguồn:** `docs/superpowers/specs/2026-11-25-p3-pos-inventory-students-rbac-design.md` (`APPROVED 2026-11-25`)
- **Nhánh làm việc (3 repo):** `feat/p3-pos-inventory-students-rbac`

---

## Thứ tự Triển khai (6 Giai đoạn Gia tăng)

### Giai đoạn 1 — Schema, Enums, Exceptions & Tồn kho nguyên tử (`api/`)
1. Tạo 2 additive migrations:
   - `2026_11_25_500001_add_inventory_columns_to_products_table.php` (`stock_quantity` nullable unsigned int, `low_stock_threshold` default 5, composite index `(is_available, stock_quantity)`).
   - `2026_11_25_500002_add_student_user_id_to_students_table.php` (`student_user_id` nullable unique FK → `users.id` `nullOnDelete`).
2. Cập nhật `Product` model (`stock_quantity`, `low_stock_threshold`, `isInStock()`), `ProductResource` (`stock_quantity`, `low_stock_threshold`, `is_in_stock`), `MenuSeeder` (tồn kho mẫu thực tế cho 30 món).
3. Tạo `InsufficientStockException` (`422`, `error_code: INSUFFICIENT_STOCK`, `meta`).
4. Cập nhật `OrderService`:
   - `resolveAndLockProducts()`: trừ `stock_quantity` nguyên tử nếu `stock_quantity !== null`, ném `InsufficientStockException` nếu vượt tồn kho.
   - `transitionStatus(..., CANCELLED)`: khóa sản phẩm theo thứ tự chuẩn `student_cards → spending_limits → sandbox_bank_accounts → products (id ASC) → orders → payment_transactions` và hoàn `stock_quantity` khi hủy đơn.
   - **Slice P2 Minor #1:** sinh sẵn `gateway_txn_ref` (`CHG-SBX-...`) trước khi gọi `chargeAutoDebit()` và lưu vào cả bản ghi `SUCCEEDED` lẫn `FAILED`/`TIMEOUT` qua `PaymentLedgerService`.
   - Thêm `createStaffPosOrder(User $staff, ...)` dùng chung 100% logic tồn kho/hạn mức/ngân hàng/sổ cái cho Staff POS Terminal (`/admin/pos`).
5. Xử lý `InsufficientStockException` trả JSON `422` chuẩn ở `OrderController` và `CanteenPosPurchaseController`.

### Giai đoạn 2 — RBAC 4 Vai trò (`admin`, `staff`, `parent`, `student`) (`api/`)
1. Mở rộng `UserRole` (`PARENT = 'parent'`, `STUDENT = 'student'`, giữ `CUSTOMER = 'customer'` làm alias tương thích ngược), bổ sung `isParentEquivalent()`, `isStudent()`, `effectiveRoleValue()`.
2. Cập nhật `UserResource` trả thêm `effective_role` và `can_manage_canteen`.
3. Cập nhật `EnsureUserHasRole`: khớp `parent` với cả `parent` và `customer`, trả `403` kèm `error_code: PARENT_ROLE_REQUIRED` khi `student` truy cập route `role:parent`.
4. Cập nhật `Student` model (`studentUser()` relation + `student_user_id` fillable), `StudentCardService`, `SpendingLimitService`, `OrderService`, `OrderController` để `student` chỉ truy cập thẻ/đơn hàng của chính mình (`students.student_user_id = $user->id`) và gắn `role:parent` lên các route tài chính phụ huynh (`bank-links`, `verify-otp`, `resend-otp`, `unlink/*`, `spending-limit`).
5. Cập nhật `StudentCardSeeder`: khởi tạo thêm tài khoản học sinh demo (`phone: 0901234568`, `zalo_user_id: demo_student1`, `role: student`) gắn vào `Titkul Student` (`HS-2026-0001`, thẻ `F156BA`).

### Giai đoạn 3 — Admin POS Terminal, Student CRUD & Product Stock Management (`api/`)
1. `ProductManagementController` + Blade `admin/products/{index,form}.blade.php`: hiển thị badge tồn kho, lọc `low`/`out`, chỉnh `stock_quantity`/`low_stock_threshold`, và nhập kho nhanh `PATCH /admin/products/{product}/stock` (`role:admin`).
2. `PosTerminalController` (`GET /admin/pos`, `GET /admin/pos/lookup-card`, `POST /admin/pos/checkout`, `GET /admin/pos/receipt/{order}`, `role:staff,admin`) + Blade `admin/pos/index.blade.php` & `admin/pos/receipt.blade.php` (in biên lai `@media print`, tra cứu thẻ không lộ STK/CCCD, giỏ POS + chống bấm đúp `idempotency_key`).
3. `StudentManagementController` (`GET /admin/canteen/students` cho `staff,admin`; `create/store/edit/update/destroy/issueCard` cho `admin`) + Blade `admin/canteen/students/{index,form}.blade.php`.
4. Cập nhật `admin/layout.blade.php` thêm mục điều hướng **POS Quầy Căn tin** và **Học sinh** (đúng quyền `staff`/`admin`).

### Giai đoạn 4 — Frontend Mini App (`miniapp/`)
1. Cập nhật types & helpers: `Product` (`stock_quantity`, `low_stock_threshold`, `is_in_stock`), `AuthUser` (`role`, `effective_role`, `can_manage_canteen`), `formatPaymentError()` hỗ trợ `INSUFFICIENT_STOCK` & `PARENT_ROLE_REQUIRED`, cập nhật `cart-logic.ts` / unit tests.
2. Cập nhật `HomePage`, `ProductDetailPage`, `CartPage`: hiển thị tồn kho, chặn thêm vượt tồn kho, + **Slice P2 Minor #2** (phím mũi tên WAI-ARIA cho `role="radiogroup"`).
3. Cập nhật `OrdersPage`: **Slice P2 Minor #2** hiển thị badge trạng thái thanh toán cạnh trạng thái xử lý đơn.
4. Cập nhật `CanteenHubPage` & `ZaloPhoneLoginPanel`/`Modal`: chế độ chỉ-xem cho tài khoản `student` (ẩn thao tác ngân hàng/hạn mức phụ huynh), nút điền nhanh SĐT Học sinh Demo (`0901234568`) cạnh Phụ huynh Demo (`0901234567`).

### Giai đoạn 5 — Automated Testing, Docs & Independent Code Review
1. Viết 4 bộ Feature Tests Backend mới:
   - `ProductInventoryTest.php` (trừ kho, vượt kho `422`, rollback khi lỗi bank/hạn mức, hoàn kho khi hủy đơn, Admin nhập kho nhanh & RBAC).
   - `StaffPosTerminalTest.php` (lookup thẻ không lộ PII, checkout thẻ học sinh & tiền mặt, idempotency, lỗi hạn mức/số dư/hết kho, biên lai, RBAC `staff` vs `parent`/`student`).
   - `AdminStudentCrudTest.php` (tạo học sinh gắn nhiều con cho 1 phụ huynh, sửa/chuyển phụ huynh, cấp phát thẻ mới, chặn xóa học sinh đã có thẻ, RBAC `staff` 403).
   - `FourRoleRbacTest.php` (kiểm chứng đầy đủ `admin`, `staff`, `parent`/`customer`, `student` trên cả Mini App API & Admin Blade, kèm `gateway_txn_ref` khi `TIMEOUT`).
2. Bổ sung Unit Tests Frontend (`payment.test.ts`, `cart-logic.test.ts`).
3. Cập nhật `docs/api-contract.md` (§10), `docs/database.md` (§6), `docs/PROJECT_STATE.md`.
4. Chạy **Independent Code Review** (`research` subagent) kiểm tra toàn bộ diff trên cả 3 repo.

### Giai đoạn 6 — Khắc phục Review, Kiểm chứng Tươi, Dọn Docker & Commit
1. Xử lý 100% Critical & Important (và Minor nhanh) từ Code Review.
2. Chạy tươi toàn bộ Quality Gates sau lần sửa cuối:
   - Backend (Docker): `vendor/bin/pint --test`, `php artisan test --compact`, `php artisan migrate:fresh --force --seed`.
   - Frontend: `npm test`, `npx tsc --noEmit`, `npm run build`.
3. Dừng & dọn sạch Docker stack (`down -v`), commit sạch trên cả 3 repo (`feat/p3-pos-inventory-students-rbac`), bàn giao báo cáo kèm bằng chứng.
