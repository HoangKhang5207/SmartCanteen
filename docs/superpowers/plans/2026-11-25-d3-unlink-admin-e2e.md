# Kế hoạch Triển khai — Trục 2 · Slice D3: Hủy Liên kết Ngân hàng qua OTP, Quản trị Blade Căn tin Số, Dọn 7 Deferred Minors & E2E 4 Luồng

- **Đặc tả đã phê duyệt:** `docs/superpowers/specs/2026-11-25-d3-unlink-admin-e2e-design.md`
- **Nhánh làm việc:** `feat/d3-unlink-admin-e2e` (độc lập trên `root`, `api/`, `miniapp/`)
- **Chiến lược kiểm chứng:** TDD / Feature + Unit Tests trên PostgreSQL 17 thật qua Docker `sc-d3`, Laravel Pint, Vitest, TypeScript, Vite Build, và Independent Code Review.

## Task 1 — Chuẩn bị nhánh & tài liệu kế hoạch
- Tạo nhánh `feat/d3-unlink-admin-e2e` từ `main` trên cả 3 repo (`root`, `api/`, `miniapp/`).
- Đánh dấu đặc tả Slice D3 `APPROVED (2026-11-25)`.

## Task 2 — Backend API: Hủy liên kết Ngân hàng qua OTP (`UNLINK-01..07`, `OTP-08`) + 3 Deferred Minors Backend
1. `api/app/Services/BankLinkingService.php`:
   - Bổ sung `initiateUnlink(User $user, int $bankLinkId): array`, `verifyUnlinkOtp(User $user, int $bankLinkId, string $otpCode): StudentCard`, `resendUnlinkOtp(User $user, int $bankLinkId): array` với khóa hàng `student_cards → bank_links → otp_verifications`, tách helper dùng chung cho `BANK_LINK` và `BANK_UNLINK`.
   - Khắc phục Minor #2: load thêm `'spendingLimit'` trong `verifyOtp()` và `verifyUnlinkOtp()`.
2. `api/app/Http/Controllers/Api/V1/CanteenBankLinkController.php` & `api/routes/api.php`:
   - Thêm 3 action `initiateUnlink`, `verifyUnlinkOtp`, `resendUnlinkOtp` với middleware `throttle:10,1`.
3. Khắc phục Minor #1 & #3:
   - `api/app/Http/Controllers/Api/V1/Admin/OrderController.php`: eager-load `'studentCard.student'`.
   - `api/app/Http/Requests/StorePosPurchaseRequest.php`: chỉ `trim` `store_name` khi `is_string`.
4. Test `api/tests/Feature/Canteen/BankUnlinkOtpTest.php`.

## Task 3 — Backend Admin Blade: Quản trị Học sinh / Thẻ / Hạn mức / Giao dịch Căn tin Số + E2E 4 Luồng
1. `api/app/Services/SpendingLimitService.php`: bổ sung `updateDailyLimitByAdmin(User $admin, StudentCard $card, int $dailyLimit, ?string $reason): StudentCard` dùng chung canonical lock order `student_cards → spending_limits` & audit log `changed_by = $admin->id`.
2. `api/app/Services/StudentCardService.php`: bổ sung `updateCardStatusByAdmin(StudentCard $card, StudentCardStatus $nextStatus): StudentCard` (chặn mở lại `REVOKED`; khi chuyển sang `REVOKED`, tự động `unlinkAutoDebit()` và chuyển `activeBankLink` sang `UNLINKED` trong cùng transaction).
3. `api/app/Http/Controllers/Admin/CanteenCardManagementController.php` & `api/routes/web.php`:
   - `index` & `show` (`role:staff,admin`), `updateStatus` & `updateSpendingLimit` (`role:admin`).
4. Nâng cấp `Admin\OrderManagementController`, `Admin\DashboardController`, `StatisticsService` và views Blade:
   - `resources/views/admin/layout.blade.php` (nav link + CSS badges)
   - `resources/views/admin/canteen/cards/{index,show}.blade.php`
   - `resources/views/admin/dashboard.blade.php`
   - `resources/views/admin/orders/{index,show}.blade.php`
5. Tests:
   - `api/tests/Feature/Admin/AdminCanteenManagementTest.php`
   - `api/tests/Feature/Canteen/DigitalCanteenEndToEndFlowsTest.php` (chạy đủ 4 luồng nghiệp vụ trọng tâm trong `FEATURES.md`).

## Task 4 — Frontend Mini App (`miniapp/`): Hủy liên kết OTP + 3 Deferred Minors Frontend
1. `miniapp/src/features/canteen/{types,api,store}.ts`:
   - Thêm kiểu `OtpPurpose`, `UnlinkSummary`, `initiateBankUnlink`, `verifyBankUnlinkOtp`, `resendBankUnlinkOtp`, `pendingUnlinkOtpSessionAtom`.
2. `miniapp/src/components/unlink-confirm-dialog.tsx`: hộp thoại xác nhận hủy liên kết accessible (`UNLINK-02..04`).
3. `miniapp/src/pages/bank-unlink-otp.tsx` & `miniapp/src/components/layout.tsx`: tuyến `/canteen/unlink-otp` với nhập OTP 6 số (`UNLINK-05..06`, `OTP-08`) và màn hình xác nhận Hủy liên kết thành công (`UNLINK-07`, Screen #15).
4. Nâng cấp `miniapp/src/pages/canteen.tsx` (nút Hủy liên kết, tích hợp `UnlinkConfirmDialog`, sửa Minor #5 & #6), `miniapp/src/components/spending-limit-sheet.tsx` (Minor #4), `miniapp/src/pages/{order-detail,limit-history}.tsx` (Minor #6).
5. Bổ sung unit test cho helper / logic mới trong `miniapp/src/features/canteen/`.

## Task 5 — Cập nhật tài liệu & Code Review độc lập
- Cập nhật `docs/api-contract.md`, `docs/database.md` (Minor #7), `docs/PROJECT_STATE.md`.
- Chạy đầy đủ Pint, PHPUnit + Migrations/Seeders (Docker `sc-d3`), Vitest, `tsc --noEmit`, `vite build`.
- Gọi subagent `Slice D3 Code Reviewer` kiểm tra độc lập toàn bộ diff của 3 repo.

## Task 6 — Khắc phục Review, Kiểm chứng cuối, Commit & Handoff
- Sửa các phát hiện Critical/Important (nếu có), chạy lại toàn bộ cổng chất lượng sau lần sửa cuối, dọn Docker `sc-d3`, commit sạch trên nhánh `feat/d3-unlink-admin-e2e` ở cả 3 repo và bàn giao Báo cáo Mục 17.
