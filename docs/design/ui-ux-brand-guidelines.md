# Hướng dẫn Thiết kế Giao diện & Trải nghiệm Người dùng (`docs/design/ui-ux-brand-guidelines.md`)

## 1. Nguồn cảm hứng & Nhận diện Thương hiệu (Brand Identity)

Hệ thống giao diện của **Smart Canteen** (trên cả Zalo Mini App `miniapp/` và Cổng Quản trị `api/resources/views/admin/`) được xây dựng đồng nhất từ tài sản thương hiệu gốc `canteen_logo.jpg`:
- **Biểu tượng:** Bộ muỗng, nĩa và dao màu xanh lá tươi (`#5DBE46`) đặt trên nền trắng tinh khiết (`#FFFFFF`).
- **Chữ thương hiệu:** Dòng chữ `"Canteen"` màu đen than đậm (`#111827`).

---

## 2. Hệ thống Màu sắc Chuẩn Tiếp cận WCAG 2.1 AA (Color Palette)

Vì màu xanh lá tươi nguyên bản trên logo (`#5DBE46`) có độ tương phản `2.32:1` khi đi với chữ trắng (chưa đạt ngưỡng `4.5:1` của chuẩn WCAG 2.1 AA cho văn bản nhỏ), hệ thống thiết kế phân tách rõ **màu nhấn thương hiệu (Accent)** và **màu hành động chính (Primary Interactive)**:

| Token Tailwind / CSS | Mã Hex | Vai trò sử dụng | Độ tương phản (Contrast Ratio) |
|---|---|---|---|
| `brand-50` | `#EEF8EB` | Nền banner chào mừng, nền tab đang chọn, nền thông báo thành công | `18.1:1` với chữ `#111827` |
| `brand-100` | `#D8F0D2` | Viền thẻ thương hiệu, nền huy hiệu trạng thái `READY` | `8.40:1` với chữ `brand-900` (`#14532D`) |
| `brand-500` | `#5DBE46` | **Màu Lá Cây Gốc của Logo** — dùng cho điểm nhấn trang trí, viền nổi bật, và nút hành động trên thanh nổi nền tối (`#0F172A`) | `8.25:1` với chữ tối `#0F172A` |
| `brand-600` | `#3FA32B` | Màu chuyển tiếp trung gian cho biểu tượng và viền phụ | — |
| `brand-700` (`--zmp-primary-color`) | `#2E7D32` | **Màu Tương Tác Chính (Primary CTA)** — dùng cho nút bấm chính, chip danh mục đang chọn, giá tiền món ăn, đường viền focus bàn phím | **`5.14:1`** với chữ trắng `#FFFFFF` (**Đạt chuẩn WCAG 2.1 AA**) |
| `brand-800` | `#236B27` | Trạng thái hover / active khi nhấn nút chính | `6.72:1` với chữ trắng `#FFFFFF` |
| `brand-900` | `#14532D` | Chữ tiêu đề nhấn mạnh và chữ trên huy hiệu xanh lá | `9.65:1` với nền trắng |
| `surface-muted` | `#F6F8F5` | Màu nền tổng thể của trang (ấm, dịu mắt, làm nổi bật thẻ món ăn trắng `#FFFFFF`) | — |

---

## 3. Màu sắc Ngữ nghĩa cho Trạng thái Đơn hàng (Semantic Status Badges)

Mọi trạng thái đơn hàng luôn đi kèm **nhãn văn bản tiếng Việt rõ ràng** (không bao giờ chỉ dùng màu sắc đơn thuần):

| Mã trạng thái | Nhãn hiển thị (Mini App & Admin) | Nền (Background) | Chữ (Text) | Ý nghĩa nghiệp vụ |
|---|---|---|---|---|
| `PENDING` | **Chờ xác nhận** | `#FEF3C7` (`amber-100`) | `#92400E` (`amber-800`) | Đơn mới đặt, đang chờ bếp tiếp nhận |
| `PREPARING` | **Đang chế biến** | `#DBEAFE` (`blue-100`) | `#1E40AF` (`blue-800`) | Nhân viên bếp đang chuẩn bị suất ăn |
| `READY` | **Sẵn sàng nhận món** | `#D8F0D2` (`brand-100`) | `#14532D` (`brand-900`) | Món đã xong, mời khách tới quầy nhận |
| `COMPLETED` | **Đã hoàn thành** | `#F3F4F6` (`gray-100`) | `#374151` (`gray-700`) | Khách đã nhận món hoàn tất |
| `CANCELLED` | **Đã hủy** | `#FEE2E2` (`red-100`) | `#991B1B` (`red-800`) | Đơn bị hủy bởi nhân viên/quản trị |

---

## 4. Quy chuẩn 4 Trạng thái Màn hình & Tương tác (Screen States & Ergonomics)

Mỗi màn hình trên Zalo Mini App (`MenuPage`, `ProductDetailPage`, `CartPage`, `OrdersPage`, `OrderDetailPage`) đều tuân thủ:
1. **Loading State (`LoadingState`):** Hiển thị `Spinner` kèm thông báo ngữ cảnh (`role="status"`, `aria-live="polite"`).
2. **Error State (`ErrorState`):** Hiển thị thông báo lỗi rõ ràng (`role="alert"`) và nút **"Thử lại"** có trạng thái loading khi đang tải lại (`isRetrying`).
3. **Empty State (`EmptyState`):** Hướng dẫn thân thiện khi tìm kiếm không có món ăn, giỏ hàng trống hoặc chưa có đơn hàng nào, kèm nút điều hướng về Thực đơn.
4. **Success State:** Bố cục thẻ một cột ưu tiên thiết bị di động (`320px–480px`), kích thước nút bấm cảm ứng tối thiểu `36px–44px`, thanh điều hướng đáy cố định (`BottomNav`) hiển thị badge tổng số lượng món trong giỏ theo thời gian thực.
