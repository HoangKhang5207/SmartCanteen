# Trục 2 · Slice D2 — Hạn mức Chi tiêu Hằng ngày, Lịch sử Hạn mức, Thanh toán Trừ Hạn mức & Chi tiết Đơn Gắn Thẻ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans`, `superpowers:test-driven-development`, `frontend-ui-engineering`, `vercel-react-best-practices`, and `superpowers:verification-before-completion`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Hiện thực hóa trọn vẹn Slice D2 (`LIMIT-01..13`, `ORDER-01..07`, `DETAIL-01..14`, thanh toán trừ hạn mức khi đặt đơn & mô phỏng quẹt thẻ POS, tự động làm mới hạn mức ngày, hoàn hạn mức khi hủy đơn) đồng thời xử lý các Deferred Minors từ Slice D1.

**Architecture:** Mở rộng additive PostgreSQL 17 với 2 bảng mới (`spending_limits`, `spending_limit_logs`) và 7 cột thanh toán/thẻ tùy chọn trên `orders`. Áp dụng cơ chế lazy daily reset theo múi giờ `Asia/Ho_Chi_Minh` kèm thứ tự khóa dòng chuẩn `student_cards → spending_limits → products → orders` chống deadlock và chống vượt hạn mức đồng thời. Frontend Mini App nâng cấp `CanteenHubPage`, `CartPage`, `OrderDetailPage` và bổ sung `SpendingLimitSheet` + `LimitHistoryPage` tuân thủ `DESIGN.md` & WCAG 2.1 AA.

**Tech Stack:** Laravel 13 (PHP 8.4, Sanctum, PostgreSQL 17), React 18, TypeScript, ZaUI, Jotai, TanStack Query, Tailwind CSS, PHPUnit 12, Vitest 2.

**Spec:** [2026-11-25-d2-spending-limit-orders-design.md](../specs/2026-11-25-d2-spending-limit-orders-design.md) (APPROVED 2026-11-25)

## Global Constraints

- 100% additive migrations và tương thích ngược hoàn toàn với 69 PHPUnit tests + 24 Vitest tests hiện có.
- Tiền tệ lưu số nguyên dương `VND` (`unsignedInteger`), không dùng `float`.
- Hạn mức chi tiêu hằng ngày (`daily_limit`) là trần trích nợ tự động (`0..5_000_000 ₫`, bước `1.000 ₫`), KHÔNG phải ví nạp tiền.
- Thứ tự khóa dòng chuẩn: `student_cards → spending_limits → products → orders`.
- Không dùng `localStorage`/`sessionStorage`/`cookie` trên Zalo Mini App; không `git push` hoặc merge `main`.

## Review Focus

1. **Daily reset timezone boundary (`Asia/Ho_Chi_Minh`):** Kiểm chứng `spent_today` tự reset về `0` khi sang ngày mới theo múi giờ Việt Nam cả khi đọc (`GET /cards`) lẫn khi tiêu dùng (`POST /orders`, `POST /pos-purchases`).
2. **Concurrent/atomic spending limit deduction:** Khóa dòng `spending_limits` bằng `lockForUpdate()` trước khi trừ để không bao giờ vượt `daily_limit`; nếu phụ huynh hạ `daily_limit` thấp hơn `spent_today`, `remaining_limit` bằng `0` (không âm) và chặn giao dịch mới.
3. **Refund on order cancellation:** Khi Staff/Admin hủy đơn thanh toán bằng thẻ học sinh (`PAID → REFUNDED`) trong ngày, `spent_today` được hoàn đúng số tiền `total_amount` (kẹp dưới `0`); hủy đơn từ ngày trước chỉ đổi `payment_status = REFUNDED` mà không làm âm `spent_today` hôm nay.
4. **Backward compatibility of `POST /api/v1/orders` & `Idempotency-Key`:** Đơn không truyền `student_card_id` giữ nguyên `COUNTER_CASH` / `UNPAID`; `payload_hash` phải bao gồm `student_card_id` và `payment_method` để tái sử dụng `Idempotency-Key` với phương thức khác trả `409 Conflict`.
5. **Slice D1 Deferred Minors & UI a11y:** Lock `User` khi auto-provision thẻ demo, throttle `bank-links`, fail-closed default `expose_debug_otp=false` + `compose.prod.yaml` override + test `APP_ENV=production`, WCAG 2.1 AA contrast trên `StudentCardView`/`BankLinkFormPage`, `whitespace-nowrap` trên `BottomNav`.

---

### Task 1: RED Tests + Migrations, Enums & Models (`api/`)

- [ ] **Step 1:** Viết trước `SpendingLimitAndOrderPaymentTest` (backend) và `spending-limit.test.ts` (frontend), chạy xác nhận RED.
- [ ] **Step 2:** Tạo 3 migrations additive (`spending_limits`, `spending_limit_logs`, mở rộng `orders`), 2 Enums (`PaymentMethod`, `PaymentStatus`), 2 Models (`SpendingLimit`, `SpendingLimitLog`), cập nhật `StudentCard` & `Order`.

### Task 2: Domain Services, Banking Adapter Extension & Slice D1 Backend Minors (`api/`)

- [ ] **Step 1:** Mở rộng `BankingGatewayInterface` & `MockEpayBankingGateway` (`chargeAutoDebit`, `refundAutoDebit`).
- [ ] **Step 2:** Tạo `SpendingLimitExceededException`, `CardNotLinkedException`, `SpendingLimitService`.
- [ ] **Step 3:** Mở rộng `OrderService` (`createOrder` hỗ trợ `student_card_id` + `payment_method`, `createPosPurchase` sinh `#POS-YYMMDD-XXXXXX`, `transitionStatus` hoàn hạn mức khi `CANCELLED`).
- [ ] **Step 4:** Xử lý Slice D1 Backend Minors (`StudentCardService` lock `User`, `config/services.php` default `expose_debug_otp=false`, `compose.prod.yaml` explicit `BANKING_EXPOSE_DEBUG_OTP: "false"`, `throttle:20,1` trên `bank-links`, test `APP_ENV=production` trong `DigitalCanteenBankLinkTest`).

### Task 3: Slice D2 REST API Controllers, Requests, Resources & Routes (`api/`)

- [ ] **Step 1:** Tạo `UpdateSpendingLimitRequest`, `StorePosPurchaseRequest`, mở rộng `StoreOrderRequest`.
- [ ] **Step 2:** Tạo `SpendingLimitResource`, `SpendingLimitLogResource`, mở rộng `StudentCardResource` & `OrderResource` (đầy đủ 14 trường `DETAIL-01..14`).
- [ ] **Step 3:** Tạo `CanteenSpendingLimitController`, `CanteenPosPurchaseController`, cập nhật `OrderController` & `routes/api.php`.
- [ ] **Step 4:** Chạy `vendor/bin/pint --test` và toàn bộ `php artisan test --compact` trong Docker xác nhận 100% GREEN.

### Task 4: Frontend Mini App — Spending Limit, POS Demo, Cart Payment, Enhanced Order Detail & D1 A11y Polish (`miniapp/`)

- [ ] **Step 1:** Triển khai `spending-limit.ts` (làm `spending-limit.test.ts` chuyển RED → GREEN), mở rộng `canteen/types.ts`, `canteen/api.ts`, `orders/types.ts`, `orders/api.ts`, `lib/api-client.ts` (`apiPut`).
- [ ] **Step 2:** Tạo `SpendingLimitSheet`, `PosPurchaseSheet`, `LimitHistoryPage`, nâng cấp `CanteenHubPage`, `CartPage`, `OrderDetailPage`, đăng ký `/canteen/limit-history` trong `layout.tsx`.
- [ ] **Step 3:** Áp dụng Slice D1 Frontend A11y Minors (`student-card-view.tsx`, `bank-link-form.tsx`, `bottom-nav.tsx`).
- [ ] **Step 4:** Chạy `npm test`, `npx tsc --noEmit`, `npm run build` xác nhận 100% GREEN.

### Task 5: Docs, Independent Code Review, Verification & Handoff

- [ ] **Step 1:** Cập nhật `docs/api-contract.md`, `docs/database.md` (sửa thành 3 Seeders + 2 bảng mới + cột mở rộng `orders`), `docs/PROJECT_STATE.md`.
- [ ] **Step 2:** Dispatch Fresh-context Code Reviewer rà soát toàn bộ diff trên cả 3 repo theo **Review Focus**.
- [ ] **Step 3:** Khắc phục mọi vấn đề Critical/Important (nếu có), chạy lại toàn bộ Quality Gates, dọn stack Docker tạm `sc-d2` và trả báo cáo hoàn tất Mục 17.
