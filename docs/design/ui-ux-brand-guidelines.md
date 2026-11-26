# Hướng dẫn Thiết kế Giao diện & Trải nghiệm Người dùng (`docs/design/ui-ux-brand-guidelines.md`)

## 1. Nhận diện Thương hiệu & Bảng màu Chuẩn WCAG 2.1 AA

Hệ thống giao diện đồng nhất trên cả **Zalo Mini App (`miniapp/`)**, **Admin Portal (`/admin`)** và **Quầy POS (`/admin/pos`)** xuất phát từ logo `canteen_logo.jpg` (`#5DBE46` xanh lá tươi + `#111827` đen than):

| Token Tailwind / CSS | Mã Hex | Vai trò sử dụng | Độ tương phản (Contrast Ratio) |
|---|---|---|---|
| `brand-50` | `#EEF8EB` | Nền banner chào mừng, thẻ trạng thái đang chọn, nền thông báo thành công | `18.1:1` với `#111827` |
| `brand-100` | `#D8F0D2` | Viền thẻ thương hiệu, nền huy hiệu `READY` / `PAID` | `8.40:1` với `brand-900` (`#14532D`) |
| `brand-500` | `#5DBE46` | Màu nhấn logo, viền thẻ học sinh, nút nổi trên thanh tối `#0F172A` | `8.25:1` với `#0F172A` |
| `brand-700` (`--zmp-primary-color`) | `#2E7D32` | **Màu Tương Tác Chính (Primary CTA)** — nút bấm, giá tiền, viền focus bàn phím | **`5.14:1`** với `#FFFFFF` (**Đạt WCAG 2.1 AA**) |
| `brand-800` | `#236B27` | Hover / Active nút chính | `6.72:1` với `#FFFFFF` |
| `surface-muted` | `#F6F8F5` | Nền trang tổng thể làm nổi bật các thẻ nội dung trắng `#FFFFFF` | — |

---

## 2. Kiến trúc Điều hướng & 11 Màn hình Zalo Mini App (`miniapp/`)

### 2.1. Thanh Điều hướng Đáy 5 Tab (`BottomNav`)
1. **Thực đơn (`/`)** — Public (Lazy Auth).
2. **Căn tin (`/canteen`)** — Yêu cầu đăng nhập SĐT Zalo.
3. **Giỏ (`/cart`)** — Public xem/chỉnh giỏ; yêu cầu đăng nhập khi bấm đặt đơn; hiện badge tổng số phần ăn.
4. **Đơn hàng (`/orders`)** — Lịch sử đơn hàng (Online + POS), polling 10s.
5. **Thông báo (`/notifications`)** — Badge pill đỏ (`1..99`, `99+`) cập nhật mỗi 20s kèm vùng live region `aria-live="polite" aria-atomic="true"` cho trình đọc màn hình (không đọc trùng với pill visual `aria-hidden="true"`).

### 2.2. Danh sách 11 Trang & Component Chuyên biệt
| Tuyến đường | Màn hình | Thành phần UI nổi bật |
|---|---|---|
| `/` | `MenuPage` | Chip danh mục, tìm kiếm debounce tiếng Việt, badge tồn kho (`Còn X phần` / `Hết món`), snackbar chạm trần tồn kho, phân trang |
| `/products/:id` | `ProductDetailPage` | `<ProductImage variant="detail" />`, giới hạn số lượng theo `stock_quantity` |
| `/cart` | `CartPage` | Chọn phương thức thanh toán `role="radiogroup"` hỗ trợ **roving `tabIndex` + phím mũi tên**, chọn thẻ học sinh, ẩn nút `"Đổi hạn mức"` với vai trò `student` |
| `/orders` | `OrdersPage` | Danh sách đơn có phân trang kèm cả badge trạng thái đơn & badge trạng thái thanh toán (`UNPAID`/`PAID`/`REFUNDED`/`FAILED`/`PENDING_RECONCILE`) |
| `/orders/:id` | `OrderDetailPage` | Đủ 14 trường `DETAIL-01..14`, nút mở cổng **VNPay Sandbox** an toàn (`isSafeHttpUrl`), tự động polling 7s |
| `/canteen` | `CanteenHubPage` | Bộ chuyển đổi đa thẻ học sinh, `<StudentCardView />`, thanh tiến độ hạn mức ngày, chế độ chỉ-xem cho `student`, nút mô phỏng POS & nút Đăng xuất |
| `/canteen/link-bank` | `BankLinkFormPage` | Chọn ngân hàng hỗ trợ, kiểm tra độ dài STK theo từng ngân hàng, tự động điền SĐT phụ huynh, checkbox điều khoản trích nợ tự động |
| `/canteen/verify-otp` | `BankLinkOtpPage` | `<OtpCodeInput />` 6 ô hỗ trợ dán/sửa chữ số giữa/backspace, đồng hồ đếm ngược wall-clock `180s`, cooldown gửi lại `30s`, chặn double-submit |
| `/canteen/unlink-otp` | `BankUnlinkOtpPage` | Xác thực OTP hủy liên kết + màn hình hoàn tất hủy liên kết (Screen #15) |
| `/canteen/limit-history` | `LimitHistoryPage` | Lịch sử kiểm toán thay đổi hạn mức ngày (`SpendingLimitLog`) phân trang |
| `/notifications` | `NotificationsPage` | Lọc *Tất cả / Chưa đọc*, nút *Đọc tất cả*, biểu tượng ngữ nghĩa theo `severity`, điều hướng an toàn `isSafeInternalRoute()` |

---

## 3. Hệ thống Huy hiệu Ngữ nghĩa (Semantic Badges)

### 3.1. Trạng thái Đơn hàng (`OrderStatus`)
| Mã | Nhãn | Nền / Chữ |
|---|---|---|
| `PENDING` | **Chờ xác nhận** | `amber-100` / `amber-800` |
| `PREPARING` | **Đang chế biến** | `blue-100` / `blue-800` |
| `READY` | **Sẵn sàng nhận món** | `brand-100` / `brand-900` |
| `COMPLETED` | **Đã hoàn thành** | `gray-100` / `gray-700` |
| `CANCELLED` | **Đã hủy** | `red-100` / `red-800` |

### 3.2. Trạng thái Thanh toán (`PaymentStatus`) & Tồn kho
- `PAID` (**Đã thanh toán** — xanh lá), `UNPAID` (**Chưa thanh toán** — hổ phách), `REFUNDED` (**Đã hoàn tiền** — xanh dương), `FAILED` (**Thanh toán lỗi** — đỏ), `PENDING_RECONCILE` (**Chờ đối soát** — tím).
- Tồn kho: `stock_quantity === 0` (**Hết hàng** — đỏ, khóa nút mua); `<= low_stock_threshold` (**Sắp hết: còn X** — hổ phách); `null` (**Không giới hạn**).

---

## 4. Quy chuẩn Tiếp cận (A11y), Hình ảnh & Biên lai POS
1. **4 Trạng thái Màn hình:** Mọi trang đều triển khai `LoadingState` (`role="status"`), `ErrorState` (`role="alert"` + nút Thử lại), `EmptyState`, và `Success`.
2. **Focus & Keyboard:** `useFocusTrap` cho tất cả Modal/Dialog (`ZaloPhoneLoginModal`, `SpendingLimitSheet`, `UnlinkConfirmDialog`), khôi phục focus khi đóng, và roving `tabIndex` trên `radiogroup`.
3. **Hình ảnh Món ăn (`<ProductImage />`):** Khung vuông cố định `80×80 px` (`card`), `56×56 px` (`cart`), `aspect-[4/3]` (`detail`) với `object-cover object-center` và fallback tự động về `canteen-logo.jpg` khi lỗi mạng.
4. **Biên lai Quầy POS (`/admin/pos/receipt/{order}`):** Bố cục khổ giấy in nhiệt gọn nhẹ, `@media print` tự động ẩn thanh điều hướng và nút thao tác.
