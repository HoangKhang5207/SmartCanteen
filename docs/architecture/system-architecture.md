# Kiến trúc Hệ thống — Smart Canteen (`docs/architecture/system-architecture.md`)

## 1. Tổng quan Kiến trúc (High-Level Architecture)

Smart Canteen được thiết kế theo mô hình **Modular Monolith** đặt trong một **Monorepo** gồm hai ứng dụng độc lập giao tiếp qua giao thức RESTful JSON API (`/api/v1`):

1. **`miniapp/` (Customer Frontend):** Ứng dụng Zalo Mini App viết bằng **React 18 + TypeScript + ZaUI (`zmp-ui`) + Jotai + TanStack Query + Tailwind CSS**.
2. **`api/` (Backend API & Admin Portal):** Ứng dụng **Laravel 13 (PHP 8.5)** đóng hai vai trò:
   - Cung cấp REST API (`/api/v1/*`) xác thực bằng **Laravel Sanctum** (Personal Access Token).
   - Cung cấp Cổng Quản trị dành cho Nhân viên Bếp & Quản trị viên (`/admin/*`) render bằng **Laravel Blade** xác thực qua **Session Guard** + CSRF protection.
   - Cả hai tầng giao tiếp (API và Blade) đều gọi chung tầng dịch vụ nghiệp vụ (`OrderService`, `StatisticsService`, `ZaloIdentityService`) để đảm bảo tính nhất quán tuyệt đối.
3. **PostgreSQL 17.5 (`titkul_canteen_demo` / `titkul_canteen_test`):** Hệ quản trị cơ sở dữ liệu quan hệ đảm bảo tính toàn vẹn giao dịch ACID, khóa dòng (`SELECT ... FOR UPDATE`), và ràng buộc khóa ngoại nghiêm ngặt.

```mermaid
flowchart LR
    subgraph Client["Thiết bị Người dùng"]
        ZMA["Zalo Mini App (React 18 + TS)\nminiapp/"]
        Browser["Trình duyệt Nhân viên / Quản trị\n/admin/*"]
    end

    subgraph ZaloCloud["Hạ tầng Zalo"]
        ZaloGraph["Zalo Open API\ngraph.zalo.me/v2.0/me"]
    end

    subgraph Backend["Laravel 13 Monolith (api/)"]
        Cors["CORS & Sanctum / Session Middleware"]
        RBAC["EnsureUserHasRole (role:staff,admin / role:admin)"]
        ApiCtrl["API V1 Controllers\nAuth / Menu / Order / Admin"]
        WebCtrl["Admin Blade Controllers\nDashboard / Orders / Categories / Products"]
        Services["Domain Services\nOrderService · StatisticsService · ZaloIdentityService"]
    end

    subgraph DB["PostgreSQL 17.5"]
        PG[("titkul_canteen_demo\nusers · categories · products\norders · order_items · order_status_logs")]
    end

    ZMA -->|"Bearer Token + Idempotency-Key\nJSON /api/v1/*"| Cors
    Browser -->|"Session Cookie + CSRF\n/admin/*"| Cors
    Cors --> RBAC
    RBAC --> ApiCtrl
    RBAC --> WebCtrl
    ApiCtrl --> Services
    WebCtrl --> Services
    Services -->|"HMAC-SHA256 appsecret_proof"| ZaloGraph
    Services -->|"DB::transaction + lockForUpdate"| PG
```

---

## 2. Luồng Xác thực Khách hàng & Phân quyền (Authentication & RBAC Flow)

```mermaid
sequenceDiagram
    actor Customer as Khách hàng (Zalo Mini App)
    participant ZSDK as zmp-sdk (getAccessToken)
    participant API as Laravel AuthController
    participant ZService as ZaloIdentityService
    participant Zalo as graph.zalo.me/v2.0/me
    participant DB as PostgreSQL (users + tokens)

    Customer->>ZSDK: Mở ứng dụng Smart Canteen
    alt Chạy trong Zalo Mini App thật
        ZSDK-->>Customer: Trả về Zalo OAuth access_token
        Customer->>API: POST /api/v1/auth/zalo { access_token }
        API->>ZService: verifyAccessToken(accessToken)
        ZService->>Zalo: GET /v2.0/me (header: access_token, appsecret_proof)
        Zalo-->>ZService: { id, name, picture }
        ZService-->>API: Thông tin định danh Zalo đã xác thực
        API->>DB: updateOrCreate User (zalo_user_id, role=customer)
        API->>DB: createToken('miniapp')
        API-->>Customer: 200 OK { token, user }
    else Chạy trên trình duyệt Local (APP_ENV=local & DEMO_LOGIN_ENABLED=true)
        Customer->>API: POST /api/v1/auth/demo { demo_key }
        API->>DB: updateOrCreate Demo User (role=customer)
        API-->>Customer: 200 OK { token, user }
    end
```

- **Lưu trữ Token trong bộ nhớ:** Token Sanctum và thông tin `AuthUser` được giữ trong bộ nhớ RAM của ứng dụng Mini App (`miniapp/src/lib/api-client.ts` và `miniapp/src/features/auth/api.ts`), không lưu vào `localStorage` hay `cookie`.

---

## 3. Luồng Đặt đơn hàng Giao dịch, Chống trùng & Tính tiền phía Server

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant Cart as CartPage (Jotai)
    participant OrderCtrl as Api\V1\OrderController
    participant OrderSvc as OrderService
    participant DB as PostgreSQL

    Customer->>Cart: Bấm "Đặt món ngay"
    Cart->>Cart: Sinh Idempotency-Key cố định theo fingerprint (items + note)
    Cart->>OrderCtrl: POST /api/v1/orders (Header: Idempotency-Key)
    OrderCtrl->>OrderSvc: createOrder(user, items, note, idempotencyKey)
    OrderSvc->>OrderSvc: Tính payload_hash = SHA-256(normalized items + note)
    OrderSvc->>DB: Tìm Order theo (user_id, idempotency_key)
    alt Đã tồn tại đơn hàng với cùng Idempotency-Key
        alt Cùng payload_hash
            OrderSvc-->>OrderCtrl: Trả về đơn hàng cũ (created = false)
            OrderCtrl-->>Cart: 200 OK (Không tạo đơn trùng)
        else Khác payload_hash
            OrderSvc-->>OrderCtrl: Throw IdempotencyConflictException
            OrderCtrl-->>Cart: 409 Conflict
        end
    else Chưa tồn tại
        OrderSvc->>DB: BEGIN TRANSACTION
        OrderSvc->>DB: SELECT products WHERE is_available=true AND category.is_active=true FOR UPDATE
        alt Có món không tồn tại / hết hàng / danh mục ẩn
            OrderSvc->>DB: ROLLBACK
            OrderSvc-->>OrderCtrl: Throw ValidationException
            OrderCtrl-->>Cart: 422 Unprocessable Entity
        else Tất cả món hợp lệ
            OrderSvc->>OrderSvc: Tính unit_price, subtotal, total_amount từ bảng products
            OrderSvc->>DB: INSERT orders (status=PENDING, version=1, code=SC-YYMMDD-XXXXXX)
            OrderSvc->>DB: INSERT order_items (unit_price, subtotal chốt cứng)
            OrderSvc->>DB: COMMIT
            OrderSvc-->>OrderCtrl: Order mới (created = true)
            OrderCtrl-->>Cart: 201 Created
        end
    end
```

---

## 4. Máy trạng thái Đơn hàng (Order State Machine) & Optimistic Locking

```mermaid
stateDiagram-v2
    [*] --> PENDING: Khách tạo đơn (version = 1)
    PENDING --> PREPARING: Staff/Admin nhận chế biến (version + 1)
    PENDING --> CANCELLED: Staff/Admin hủy đơn (version + 1)
    PREPARING --> READY: Bếp nấu xong, sẵn sàng giao (version + 1)
    PREPARING --> CANCELLED: Staff/Admin hủy đơn (version + 1)
    READY --> COMPLETED: Giao món hoàn tất (version + 1)
    COMPLETED --> [*]
    CANCELLED --> [*]
```

- **Quy tắc chuyển trạng thái (`App\Enums\OrderStatus`):**
  - Mọi yêu cầu chuyển trạng thái không nằm trong mũi tên hợp lệ của sơ đồ trên đều bị từ chối với mã `422 Unprocessable Entity`.
- **Kiểm soát xung đột đồng thời (`version`):**
  - Khi hai nhân viên cùng mở một đơn hàng ở `version = 1`: người thứ nhất bấm chuyển sang `PREPARING` thành công sẽ nâng `version` lên `2`. Khi người thứ hai gửi yêu cầu với `version = 1`, `OrderService::transitionStatus()` phát hiện `$lockedOrder->version !== $expectedVersion` trong `lockForUpdate()` và trả về ngay `409 Conflict`, ngăn chặn ghi đè ngoài ý muốn.
