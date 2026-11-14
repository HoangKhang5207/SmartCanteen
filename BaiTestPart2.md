# **KẾ HOẠCH TRIỂN KHAI TEST PART 2 — ZALO MINI APP × REACT × PHP**

CÔNG TY CỔ PHẦN TITKUL · ĐỀ XUẤT KỸ THUẬT · 14/11/2026

## **1. Kết luận: Nên xây dựng ứng dụng nào?**

Tôi đề xuất bạn triển khai một Zalo Mini App có tên tạm thời:

**Smart Canteen — Đặt suất ăn & quản lý đơn hàng qua Zalo.**

Đây là ứng dụng mới, lấy cảm hứng từ nhóm nghiệp vụ căn tin, đặt món và quản lý suất ăn, nhưng không sao chép nguyên trạng hệ thống Căn Tin Số đang chạy production.

Ứng dụng sẽ có hai thành phần hoạt động thực tế:

- **Khách hàng:** Mở Zalo Mini App, xem thực đơn, chọn món, đặt hàng, theo dõi trạng thái đơn và xem lịch sử.
- **Nhân viên/quản trị:** Truy cập giao diện quản trị riêng để tiếp nhận đơn, thay đổi trạng thái chế biến, xác nhận hoàn thành và xem thống kê.

Điểm quan trọng là hai phía sử dụng **chung một backend PHP và một cơ sở dữ liệu thử nghiệm độc lập**, nên giám đốc có thể kiểm chứng dữ liệu thay đổi thực sự giữa các thiết bị.

[Showcase App Ui Design Projects :: Photos, videos, logos, illustrations and branding :: Behance](https://images.openai.com/static-rsc-4/IOjDEfLWF-bDBe4jAnZ6m6XpIt57v9_Ex7_8TSdf-lfux7fGmMxuuWxGhnQMW7pW-piHnzjncA1wI7Fv9jTzAg2nKq_fwyiibyseItlw16-D7cRcxVz90JOcjrs6bnTLxPEc10YABOICkc2TJRkFOUJDSW206-RVuNkavN2BWkE?purpose=inline)

Minh họa định hướng giao diện Mini App cho khách hàng, không phải ảnh hệ thống TitKul.

[Next-gen Kitchen Display Systems | Grafterr India](https://images.openai.com/static-rsc-4/ENYm1zS11FqeFhQdq5SslipC3l8-sSUp4NLmCcSOXL1o7kG6aezKak2SJQlynahXgeYQu7kmNutM0OuiYNjqlKavSSbdLqgQUMjFcSXkrtu91nUA7wBLrB6ax7DT3iL0rzykd7XzVspWr5hHLZ2Y8umRnqUzYmrr_zIS__vxkCc?purpose=inline)

Minh họa giao diện tiếp nhận và xử lý đơn hàng của nhân viên.

### **Vì sao đây là phương án phù hợp?**

| Tiêu chí của giám đốc        | Cách đáp ứng                                                            |
| ---------------------------- | ----------------------------------------------------------------------- |
| Có nghiệp vụ thực tế         | Đặt món, xử lý đơn, quản lý thực đơn                                    |
| Có liên hệ với Căn Tin Số    | Cùng lĩnh vực căn tin, nhưng thiết kế lại luồng nghiệp vụ               |
| Frontend React               | React + TypeScript + Zalo Mini App                                      |
| Backend PHP                  | Laravel REST API                                                        |
| Trình diễn trên điện thoại   | Deploy Mini App lên môi trường Development/Testing                      |
| Giám đốc tự quét QR          | Sử dụng QR phiên bản được Zalo cấp, với quyền truy cập kiểm thử phù hợp |
| Thể hiện năng lực Full-stack | UI, API, database, xác thực, business logic, testing và deployment      |
| Không ảnh hưởng production   | Không kết nối database/API của Căn Tin Số                               |

**Mục tiêu của bài test không nên chỉ là làm ra giao diện trông giống một website.** Bạn cần chứng minh được một luồng nghiệp vụ end-to-end: khách đặt hàng trên điện thoại → Laravel tiếp nhận → dữ liệu được lưu → nhân viên xử lý → khách thấy trạng thái mới.

## **2. Nguyên tắc bảo mật đối với Căn Tin Số production**

Vì giám đốc đã yêu cầu bảo mật nghiêm ngặt, tôi sẽ coi hệ thống Căn Tin Số là **nguồn tham khảo nghiệp vụ có giới hạn**, không phải môi trường phát triển hoặc nguồn dữ liệu cho dự án.

Tôi chưa truy cập phần quản trị sau đăng nhập, vì vậy không khẳng định đã kiểm chứng các màn hình, tính năng hoặc database thực tế. Những nghiệp vụ đề xuất trong kế hoạch này là thiết kế mới, không phải kết quả trích xuất từ hệ thống production.

Các nguyên tắc bắt buộc:

1. Chỉ quan sát giao diện và quy trình trong phạm vi được cấp quyền; tránh thao tác tạo, sửa, xóa, import/export.
2. Không lấy API token, cookie phiên, file đính kèm, database dump hoặc dữ liệu khách hàng.
3. Không chạy crawler, quét endpoint, kiểm thử bảo mật hoặc thử API bằng các phương thức có thể làm thay đổi hệ thống.
4. Không đưa thông tin đăng nhập, ảnh chụp dữ liệu thật hay cấu hình nội bộ lên GitHub, AI coding tools hoặc môi trường demo.
5. Tất cả món ăn, khách hàng, giá, đơn hàng và tài khoản của ứng dụng mới đều là **dữ liệu giả lập tự tạo**.
6. Không kết nối Laravel mới trực tiếp tới production, kể cả chỉ để đọc, nếu chưa có phê duyệt tích hợp riêng.

**Lưu ý bảo mật khẩn:** Mật khẩu production đã được chia sẻ trong cuộc trò chuyện. Nên đề nghị người quản trị đổi mật khẩu tài khoản đó và cấp một tài khoản chỉ đọc, tối thiểu quyền, nếu công ty muốn bạn tiếp tục khảo sát. Không tái sử dụng mật khẩu này cho môi trường demo.

## **3. Kiến trúc kỹ thuật đề xuất**

Tôi khuyến nghị xây dựng theo hướng **API-first, modular monolith**, không cần microservices, Kafka, Redis hay hệ thống phức tạp ở giai đoạn bài test.

**Zalo Mini App — Customer**

React + TypeScript + ZMP SDK + ZaUI

HTTPS / REST / JSON

**Laravel 13 — Backend API**

Authentication · Orders · Menu · Authorization

**Database**

MySQL / PostgreSQL

**Admin**

Laravel Blade hoặc React

Database thử nghiệm riêng · Không kết nối production

### **Công nghệ nên sử dụng**

| Thành phần     | Công nghệ                                                         | Lý do                          |
| -------------- | ----------------------------------------------------------------- | ------------------------------ |
| Mini App       | React + TypeScript                                                | Đúng yêu cầu công ty           |
| Framework      | Zalo Mini App CLI                                                 | Build, debug, deploy           |
| UI             | ZaUI + CSS/Tailwind tương thích                                   | Đồng nhất với Zalo             |
| Routing        | Router tương thích ZMP template                                   | Chuyển trang                   |
| Data fetching  | TanStack Query hoặc fetch                                         | Quản lý API state              |
| Backend        | Laravel 13, PHP 8.5                                               | REST API, validation, Eloquent |
| Database       | MySQL 8 hoặc PostgreSQL                                           | Lưu trữ dữ liệu thật của demo  |
| Authentication | Laravel Sanctum cho token nội bộ; Zalo identity flow khi tích hợp | Bảo vệ API                     |
| Admin          | Laravel Blade                                                     | Giảm thời gian phát triển      |
| Testing        | PHPUnit/Pest + API tests                                          | Kiểm tra nghiệp vụ             |
| Deployment     | Zalo Testing + HTTPS staging API                                  | Demo từ điện thoại             |

Laravel 13 tương thích PHP 8.3–8.5 theo tài liệu chính thức, do đó phù hợp với PHP 8.5.12 đang có trong Laravel Herd.

[image](https://www.google.com/s2/favicons?domain=https://laravel.com\&sz=32)

Laravel+1



**Tối ưu thời gian:** không nên tự xây một React Admin Dashboard lớn ngay từ đầu. Laravel Blade đủ để giám đốc xem đơn hàng thay đổi trên giao diện quản trị. Nếu còn thời gian, nâng cấp admin thành React.

## **4. Phạm vi nghiệp vụ cần xây dựng**

Tôi chia thành ba mức ưu tiên để tránh tình trạng phát triển rất nhiều tính năng nhưng không có một luồng nào chạy hoàn chỉnh.

**P0 — Bắt buộc hoàn thành**

Danh sách món, chi tiết món, giỏ hàng, tạo đơn, lịch sử đơn, thay đổi trạng thái đơn qua Admin và đồng bộ trạng thái về Mini App.

**P1 — Nâng chất lượng bài test**

Đăng nhập/định danh, RBAC Customer–Staff–Admin, validation, thống kê đơn, tìm kiếm, xử lý đơn trùng và bảo vệ API.

**P2 — Chỉ làm khi P0/P1 đã ổn định**

QR tại bàn, mã nhận món, thông báo trạng thái, chương trình khuyến mãi, thanh toán trực tuyến, báo cáo nâng cao.

### **Luồng demo quan trọng nhất**

1. Giám đốc quét QR và mở Smart Canteen trên Zalo.
2. Chọn một món ăn và thêm vào giỏ hàng.
3. Điều chỉnh số lượng, tạo đơn hàng thử nghiệm.
4. Laravel xác thực, tính tổng tiền phía server và lưu đơn hàng vào database riêng.
5. Bạn mở Admin trên laptop và nhìn thấy đơn mới.
6. Chuyển trạng thái đơn từ `PENDING` sang `PREPARING`, sau đó `READY`.
7. Giám đốc mở lại màn hình theo dõi trên Zalo và nhìn thấy trạng thái đã cập nhật.
8. Giám đốc kiểm tra lịch sử đơn và mã đơn hàng.

Đây là phần trình diễn quan trọng nhất vì nó chứng minh được cả Frontend, Backend, API, database và business workflow.

## **5. Thiết kế dữ liệu và API**

### **5.1. Database tối thiểu**

| Bảng                | Trường chính                                             |
| ------------------- | -------------------------------------------------------- |
| `users`             | id, name, role, zalo_user_id (nullable)                  |
| `categories`        | id, name, is_active                                      |
| `products`          | id, category_id, name, price, image_url, is_available    |
| `orders`            | id, user_id, code, status, total_amount, version         |
| `order_items`       | id, order_id, product_id, quantity, unit_price, subtotal |
| `order_status_logs` | id, order_id, from_status, to_status, changed_by         |

Các bảng cần có `created_at`, `updated_at` khi phù hợp và quan hệ khóa ngoại cần thiết.

Tiền tệ nên lưu bằng kiểu số nguyên theo đơn vị VND, không dùng `float` để tính tiền. Giá và tổng tiền được chốt tại thời điểm tạo đơn; thay đổi giá món sau đó không được làm thay đổi đơn cũ.

### **5.2. REST API tối thiểu**

| Method | Endpoint                           | Mục đích                    |
| ------ | ---------------------------------- | --------------------------- |
| GET    | `/api/v1/categories`               | Danh mục                    |
| GET    | `/api/v1/products`                 | Danh sách món               |
| GET    | `/api/v1/products/{id}`            | Chi tiết món                |
| POST   | `/api/v1/orders`                   | Tạo đơn                     |
| GET    | `/api/v1/orders`                   | Đơn của người dùng          |
| GET    | `/api/v1/orders/{id}`              | Chi tiết đơn                |
| GET    | `/api/v1/admin/orders`             | Danh sách đơn cho nhân viên |
| PATCH  | `/api/v1/admin/orders/{id}/status` | Chuyển trạng thái           |
| GET    | `/api/v1/admin/statistics`         | Thống kê cơ bản             |

Đây là bản thiết kế API cho dự án mới, không phải endpoint của Căn Tin Số.

### **5.3. Business rules phải làm đúng**

- Backend tính tổng từ giá sản phẩm trong database, không tin `total_amount` frontend gửi lên.
- Không nhận đơn chứa sản phẩm ngừng bán hoặc số lượng không hợp lệ.
- Một người dùng chỉ xem được đơn của mình, trừ khi có quyền nhân viên/quản trị.
- Phải có kiểm tra chuyển trạng thái hợp lệ.
- Dùng database transaction khi tạo order và order items.
- Sử dụng idempotency key hoặc cơ chế chống tạo đơn trùng khi khách bấm nhiều lần.
- Trả về status code và thông báo lỗi nhất quán.

Ví dụ state machine đơn hàng

Với demo, cập nhật trạng thái bằng polling mỗi 5–10 giây là đủ; chưa cần WebSocket hay message broker.

## **6. Hướng dẫn thiết lập trên Windows 11**

Máy bạn đã có Git, Node.js, npm, Laravel Herd và VS Code, nên không cần cài lại các công cụ đó.

### **Bước 1 — Kiểm tra môi trường**

Mở PowerShell trong VS Code:

```
node -v
npm -v
git --version
php -v
composer --version
```

Nếu `composer` chưa khả dụng, cài Composer trước khi tạo Laravel project. Dùng PHP 8.5 từ Herd và kiểm tra các PHP extensions theo yêu cầu Laravel.

### **Bước 2 — Tạo workspace độc lập**

```
mkdir titkul-smart-canteen
cd titkul-smart-canteen

git init
```

Đề xuất tổ chức theo monorepo:

```
titkul-smart-canteen/
├── miniapp/              # React + Zalo Mini App
├── api/                  # Laravel REST API + Admin
├── docs/
│   ├── requirements.md
│   ├── api-contract.md
│   ├── database.md
│   ├── test-cases.md
│   └── demo-script.md
├── .gitignore
└── README.md
```

### **Bước 3 — Khởi tạo Zalo Mini App**

Tài liệu Zalo cập nhật ngày 09/11/2026 hỗ trợ trực tiếp lệnh sau:&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.zaloplatforms.com\&sz=32)

Zalo Platform Document Hub+1



```
npx create-zalo-mini-app miniapp
```

Chọn React, TypeScript và template phù hợp với nghiệp vụ khi CLI yêu cầu.

```
cd miniapp
npm install
npm start
```

Có thể sử dụng `zmp start` nếu phiên bản CLI/template được tạo cấu hình theo cách này.

Mở địa chỉ localhost do CLI hiển thị để kiểm tra màn hình đầu tiên.

### **Bước 4 — Tạo Laravel Backend**

Mở terminal mới tại thư mục root:

```
composer create-project laravel/laravel api "^13.0"

cd api
php artisan --version
```

Khởi tạo API routing theo cấu hình Laravel:

```
php artisan install:api
```

Lệnh này cài đặt các thành phần API cần thiết, bao gồm Sanctum trong cấu hình Laravel tương ứng. Hãy kiểm tra thay đổi được tạo ra trước khi tiếp tục.

### **Bước 5 — Tạo database riêng**

Đối với bài test ngắn, nếu chưa có MySQL/PostgreSQL, bạn có thể bắt đầu bằng SQLite cho local development. Khi triển khai staging, chọn database bền vững riêng biệt.

Ví dụ với MySQL, tạo database rỗng:

```
CREATE DATABASE titkul_canteen_demo
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Cấu hình `.env` của Laravel:

```
APP_ENV=local
APP_DEBUG=true

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=titkul_canteen_demo
DB_USERNAME=demo_user
DB_PASSWORD=your_local_secret
```

`demo_user` phải được tạo riêng và chỉ có quyền cần thiết trên database demo. Không dùng tài khoản database của bất kỳ hệ thống production nào.

Sau khi hoàn thiện migrations:

```
php artisan migrate
php artisan db:seed
php artisan serve
```

Chỉ seed dữ liệu giả lập.

### **Bước 6 — Tạo Models, Controllers, Requests**

Ví dụ:

```
php artisan make:model Category -m
php artisan make:model Product -m
php artisan make:model Order -m
php artisan make:model OrderItem -m
php artisan make:model OrderStatusLog -m

php artisan make:controller Api/V1/ProductController
php artisan make:controller Api/V1/OrderController
php artisan make:controller Api/V1/Admin/OrderController

php artisan make:request StoreOrderRequest
```

Tách business logic ra khỏi controller, ví dụ `OrderService`, để xử lý tạo đơn trong transaction và kiểm soát tính hợp lệ của trạng thái.

### **Bước 7 — Kết nối Mini App với Laravel**

Ở môi trường local, có thể cấu hình API URL qua biến môi trường tương thích Vite:

```
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

Ví dụ một HTTP client cơ bản:

```
const API_URL = import.meta.env.VITE_API_BASE_URL;

export async function getProducts() {
  const response = await fetch(`${API_URL}/products`, {
    headers: {
      Accept: "application/json",
    },
  });

  if (!response.ok) {
    throw new Error("Cannot fetch products");
  }

  return response.json();
}
```

Đây chỉ là ví dụ cho API công khai `GET /products`. Các API được bảo vệ phải truyền token xác thực và xử lý `401`, `403`, hết hạn token, lỗi mạng.

Khi deploy lên Zalo, **không sử dụng localhost**: chuyển `VITE_API_BASE_URL` sang endpoint HTTPS staging thực sự có thể truy cập từ điện thoại.

## **7. Làm sao để giám đốc tự quét QR và trải nghiệm?**

Đây là điểm phải ưu tiên từ đầu, không để đến cuối mới triển khai.

Theo tài liệu Zalo Mini App cập nhật ngày 09/11/2026, Extension hỗ trợ deploy bản **Development** và **Testing**, sau đó trả về QR Code và Deep Link.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.zaloplatforms.com\&sz=32)

Zalo Platform Document Hub+1



### **Phân biệt các môi trường**

| Môi trường  | Vai trò                       | Có phù hợp demo?                       |
| ----------- | ----------------------------- | -------------------------------------- |
| Local       | Code và debug trên máy        | Chỉ phù hợp trình diễn local           |
| Development | Kiểm tra bản build trên Zalo  | Có, nếu tài khoản được quyền truy cập  |
| Testing     | Lưu bản kiểm thử có phiên bản | Ưu tiên khi giao cho giám đốc kiểm thử |
| Published   | Phân phối rộng sau kiểm duyệt | Không cần cho bài test nội bộ          |

Testing không đồng nghĩa với phát hành công khai. Tuy nhiên, **có QR không bảo đảm mọi tài khoản Zalo đều mở được**. Bạn phải kiểm tra quyền truy cập phiên bản thử nghiệm và tài khoản giám đốc; đừng giả định QR là liên kết công khai.

### **Quy trình cụ thể**

1. **Tạo Mini App ID**

   Truy cập [Zalo Mini App Developer Center](https://mini.zalo.me/developers/?utm_source=chatgpt.com) và thiết lập ứng dụng theo quyền được TitKul cấp. Nên dùng tài nguyên do công ty quản lý thay vì phụ thuộc lâu dài vào tài khoản Zalo cá nhân.
2. **Liên kết project**

   Mở VS Code Extension Zalo Mini App, cấu hình Mini App ID và đăng nhập bằng tài khoản Admin/Developer hợp lệ.
3. **Deploy Development**

   Đưa bản chạy đầu tiên lên Zalo để kiểm chứng UI, tài nguyên và kết nối backend thực tế.
4. **Deploy Testing**

   Tạo phiên bản có mô tả rõ ràng, ví dụ `v0.1.0 - Order workflow demo`.
5. **Kiểm tra từ điện thoại khác**

   Sử dụng tài khoản Zalo khác thuộc phạm vi kiểm thử được phép. Kiểm tra liệu QR có mở được hay bị giới hạn quyền.
6. **Trình diễn cho giám đốc**

   Gửi QR/Deep Link được sinh từ bản deploy thành công, không tự tạo đường dẫn giả. Đồng thời chuẩn bị chia sẻ màn hình làm phương án dự phòng.

### **Backend staging phải được chuẩn bị riêng**

Bạn sẽ có hai deployment độc lập:

**Frontend**

Build React được deploy lên hạ tầng Zalo Mini App.

**Backend**

Laravel API phải được deploy trên server hoặc dịch vụ hosting HTTPS có thể truy cập từ điện thoại giám đốc.

**Database**

Lưu dữ liệu đơn hàng demo riêng, không phụ thuộc localhost của bạn.

Lưu ý theo tài liệu Zalo: Mini App thường cần API HTTPS, SSL hợp lệ, CORS chính xác (đặc biệt origin `https://h5.zdn.vn`), xử lý preflight `OPTIONS`, và không nên phụ thuộc cookie/localStorage như một web app thông thường.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.zaloplatforms.com\&sz=32)

Zalo Platform Document Hub+1



Không đưa Zalo App Secret hoặc private key vào mã React. Xử lý các API server-to-server và xác minh định danh ở Laravel. CORS cũng không thay thế cho authentication hoặc authorization.

Đối với bản demo, có thể dùng tài khoản demo được cấp token riêng để kiểm chứng workflow ban đầu. Nhưng trước khi gọi đó là tính năng **đăng nhập bằng Zalo**, bạn phải hoàn thiện và kiểm thử quy trình định danh theo SDK/Open API chính thức. Không tin một `zalo_user_id` do frontend tự gửi lên.

## **8. Kế hoạch triển khai trong 7 ngày**

Đây là lộ trình mục tiêu cho một bài test MVP, không phải cam kết thời gian. Nếu phát sinh vấn đề quyền truy cập Zalo hoặc hosting, ưu tiên hoàn thiện P0 và một bản demo trên thiết bị thật trước.

Ngày 1

**Khảo sát nghiệp vụ và chạy Zalo Mini App**

Viết yêu cầu, xác định use cases, tạo Mini App ID, chạy React trên môi trường phát triển.

Mini App có màn hình đầu tiên và chạy được trên Zalo.

Ngày 2

**Laravel và database**

Tạo project, migrations, seed dữ liệu giả, xây API danh mục và sản phẩm.

Frontend lấy được danh sách món từ PHP API.

Ngày 3

**Shopping Cart & Checkout**

Hoàn thiện chọn món, giỏ hàng, validation, transaction, API tạo đơn.

Đặt hàng và lưu được dữ liệu thực.

Ngày 4

**Admin & Order Workflow**

Xây màn hình tiếp nhận đơn, chuyển trạng thái, lịch sử và polling.

Hai giao diện đồng bộ trạng thái đơn.

Ngày 5

**Authentication & Security**

Thiết lập xác thực và phân quyền, chống đơn trùng, xử lý lỗi, viết test trọng yếu.

Các API quan trọng được bảo vệ và kiểm thử.

Ngày 6

**Deploy & UAT**

Triển khai API HTTPS, deploy Zalo Testing, kiểm tra trên thiết bị và tài khoản khác.

QR/Deep Link của bản kiểm thử chạy end-to-end.

Ngày 7

**Ổn định và chuẩn bị báo cáo**

Sửa bug, tối ưu UI, lập README, test report, hướng dẫn sử dụng và kịch bản demo.

Bản MVP có thể bàn giao để giám đốc đánh giá.

## **9. Checklist nghiệm thu trước khi trình diễn**

**Mức độ sẵn sàng demo**

0/13

Mini App mở được trên điện thoại bằng Zalo

Tài khoản kiểm thử của giám đốc được cấp quyền truy cập

Backend HTTPS hoạt động ngoài mạng nội bộ

Có ít nhất 10 sản phẩm giả và 3 danh mục

Thêm/xóa món và thay đổi số lượng trong giỏ

Tạo đơn thành công, tổng tiền tính ở backend

Admin nhìn thấy đơn vừa tạo

Admin chuyển trạng thái đơn đúng quy tắc

Mini App cập nhật trạng thái và lịch sử đơn

Người dùng không xem được đơn của người khác

Chặn gửi trùng đơn và xử lý lỗi API

Không có thông tin hoặc dữ liệu production trong demo

Có README, test cases và kịch bản trình diễn

Đặt lại Sao chép checklist

## **10. Bộ deliverables nên giao cho giám đốc**

| Deliverable            | Nội dung                                    |
| ---------------------- | ------------------------------------------- |
| Source code            | Repo có `miniapp/` và `api/`                |
| Working demo           | Zalo Mini App Testing + Laravel API staging |
| QR/Deep Link           | Link mở phiên bản kiểm thử hợp lệ           |
| README                 | Hướng dẫn cài đặt, vận hành và deploy       |
| API documentation      | Endpoint, payload, response, error codes    |
| Database documentation | ERD, migrations, seed data                  |
| Test report            | Test cases, kết quả, lỗi còn tồn tại        |
| Demo video             | Video ngắn mô tả một luồng đặt hàng đầy đủ  |

### **Kịch bản báo cáo cho giám đốc**

Dạ anh, đối với bài Test Part 2, em đề xuất triển khai một Zalo Mini App phục vụ nghiệp vụ đặt món và quản lý đơn hàng căn tin, lấy cảm hứng từ lĩnh vực nghiệp vụ của Căn Tin Số nhưng được thiết kế và phát triển độc lập.

Ứng dụng sử dụng React + Zalo Mini App ở Frontend, PHP Laravel ở Backend và một database thử nghiệm riêng. Em sẽ không chỉnh sửa, sao chép hay kết nối tới dữ liệu production của hệ thống hiện có.

Mục tiêu của em là hoàn thiện một luồng nghiệp vụ thực tế: khách hàng mở Mini App trên Zalo, đặt món, hệ thống lưu đơn, nhân viên xử lý trên trang quản trị và khách hàng theo dõi trạng thái đơn hàng.

Sau khi triển khai bản Development/Testing và kiểm tra quyền truy cập, em sẽ cung cấp QR/Deep Link để anh có thể tự trải nghiệm trên điện thoại, đồng thời chuẩn bị demo trực tiếp để anh đánh giá kết quả hoạt động thực tế.

## **11. Tài liệu chính thức cần sử dụng**

Đây là các nguồn tôi đã kiểm tra, ưu tiên tài liệu Zalo cập nhật ngày 09/11/2026 và tài liệu Laravel hiện hành.

- [Zalo — Tạo project bằng CLI](https://docs.zaloplatforms.com/docs/MA/intro/getting-started/dev-use-command-line?utm_source=chatgpt.com)
- [Zalo — Deploy bằng VS Code Extension](https://docs.zaloplatforms.com/docs/MA/devtools/ext/deploy?utm_source=chatgpt.com)
- [Zalo — Các lỗi kỹ thuật thường gặp](https://docs.zaloplatforms.com/docs/MA/intro/getting-started/frequently-solved-issues?utm_source=chatgpt.com)
- [Zalo — Các lưu ý tích hợp và xác thực API](https://docs.zaloplatforms.com/docs/MA/intro/getting-started/convert-web-app-to-mini-app?utm_source=chatgpt.com)
- [Laravel 13 — Documentation](https://laravel.com/docs/13.x?utm_source=chatgpt.com)

## **12. Ba quyết định cần chốt với TitKul**

Trước khi triển khai, có ba vấn đề ảnh hưởng trực tiếp tới việc giám đốc sử dụng được sản phẩm.

Mini App ID thuộc tài khoản nào?

1. Tài khoản Developer/OA do TitKul quản lý (ưu tiên)

2. Tài khoản Developer cá nhân cho bản thử nghiệm => Chọn. Vì tôi phát triển cá nhân với mục đích Demo.

3. Chưa xác định

Backend staging sẽ đặt ở đâu?

1. Hosting/VPS độc lập có HTTPS

2. Server staging riêng do công ty cấp

3. Tunnel tạm thời cho bản demo có kiểm soát

4. Chưa có server

=> Tôi cũng không rõ. Hãy chọn giải pháp miễn phí và hoạt động tốt nhất với Zalo Mini App nhé.


Phạm vi nghiệp vụ triển khai

1. Smart Canteen — Đặt món + Admin (đề xuất) => Chọn.

2. Đặt suất ăn trước + mã nhận suất

3. Ứng dụng đặt suất ăn cho nhân viên nội bộ


Chốt: Thiết kế chi tiết theo phương án đã chọn.

**Kết luận:** Tôi ưu tiên xây dựng một sản phẩm nhỏ nhưng hoàn chỉnh, với Laravel API thật, dữ liệu giả lập riêng, phân quyền tối thiểu, các bài test trọng yếu và QR kiểm thử trên thiết bị thật. Chỉ nên mở rộng thêm tính năng khi luồng đặt hàng từ Zalo tới Admin đã chạy ổn định.

Điều quan trọng nhất trong Part 2 là giám đốc có thể **tự thao tác, nhìn thấy kết quả thay đổi thực tế và xác minh bạn đã triển khai được một ứng dụng Full-stack hoạt động**, thay vì chỉ xem giao diện hoặc video mô phỏng.
