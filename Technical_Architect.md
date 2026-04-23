# Kiến trúc hệ thống

## 1. Tổng quan kiến trúc

```
                        ┌─────────────────────────────────────────────────────────┐
                        │                      Clients                            │
                        │    Vue 3 (Customer Web)     Vue 3 (Staff/POS Web)       │
                        └──────────────────────┬──────────────────────────────────┘
                                               │ HTTPS
                                               ▼
                        ┌─────────────────────────────────────────────────────────┐
                        │                 Envoy API Gateway                       │
                        │            (Routing + JWT Validation)                   │
                        └──┬────────┬─────────┬──────────┬──────────┬─────────────┘
                           │        │         │          │          │
            ┌──────────────▼┐ ┌─────▼──────┐ ┌▼────────┐ ┌▼─────────────┐ ┌▼───────────────┐
            │ Identity      │ │ User       │ │ Product │ │ Order        │ │ Store &        │
            │ Service       │ │ Service    │ │ Service │ │ Service      │ │ Inventory      │
            │               │ │            │ │         │ │              │ │ Service        │
            │ AWS Cognito   │ │ MySQL      │ │ MySQL   │ │ MySQL+Redis │ │ ScyllaDB       │
            └───────────────┘ └────────────┘ └────┬────┘ └──────┬──────┘ └────────────────┘
                                                  │             │
                                                  │  Publish    │ Publish events
                                                  ▼             ▼
                        ┌─────────────────────────────────────────────────────────┐
                        │                       AWS SQS                           │
                        │                                                         │
                        │  [order-confirmed-queue]    [order-status-queue]        │
                        │  [payment-queue]            [analytics-queue]           │
                        │  [product-updated-queue]                                │
                        └───────────────────────┬─────────────────────────────────┘
                                                │ Consume
                                   ┌────────────┼────────────┐
                                   ▼            ▼            ▼
                         ┌──────────────┐ ┌────────────┐ ┌────────────────┐
                         │ Payment      │ │Notification│ │ Analytics      │
                         │ Service      │ │ Service    │ │ Service        │
                         │ (Worker)     │ │ (Worker)   │ │ (Worker)       │
                         │              │ │ AWS SES    │ │ DynamoDB       │
                         └──────────────┘ └────────────┘ └────────────────┘
```

---

## 2. Danh sách Microservices

| Service | Loại | Database | Mô tả |
|---|---|---|---|
| Identity Service | Web API | AWS Cognito | Xác thực, phân quyền JWT |
| User Service | Web API | MySQL | Quản lý profile, địa chỉ người dùng, store_staff (dùng Cognito `sub` làm id) |
| Product Service | Web API | MySQL | Quản lý sản phẩm, danh mục 3 cấp, variant + SKU, giá, lịch sử giá |
| Store & Inventory Service | Web API | ScyllaDB | Quản lý cửa hàng, kho tổng, tồn kho theo từng cửa hàng, nhập hàng từ kho tổng |
| Order Service | Web API | MySQL + Redis | Tạo và quản lý đơn hàng (online + POS), giỏ hàng, cache product, auto-cancel |
| Payment Service | Worker | - | Consume queue, xử lý thanh toán (mock) |
| Notification Service | Worker | - | Consume queue, gửi email qua AWS SES |
| Analytics Service | Worker | DynamoDB | Consume queue, ghi nhận và tổng hợp dữ liệu bán hàng |

---

## 3. Phân chia Database

| Database | Service sử dụng | Lý do |
|---|---|---|
| MySQL | User Service, Product Service, Order Service | Dữ liệu có quan hệ, cần ACID transaction |
| Redis (ElastiCache) | Order Service | Cache thông tin product (tên, giá, SKU, ảnh, variant) — write-through, cache miss gọi HTTP sang Product Service |
| ScyllaDB | Store & Inventory Service | Tồn kho cần read/write cực nhanh, scale tốt khi nhiều request đồng thời |
| DynamoDB | Analytics Service | Event log, schema linh hoạt, không cần join phức tạp |

---

## 4. SQS Queues

| Queue | Publisher | Consumer | Mô tả |
|---|---|---|---|
| order-confirmed-queue | Order Service | Payment Service | Kích hoạt xử lý thanh toán sau khi đơn được xác nhận |
| payment-queue | Payment Service | Order Service, Notification Service | Kết quả thanh toán → cập nhật trạng thái đơn + gửi email |
| order-status-queue | Order Service | Notification Service, Analytics Service | Mỗi lần trạng thái đơn thay đổi → gửi email + ghi analytics |
| analytics-queue | Order Service | Analytics Service | Ghi nhận đơn POS và đơn online vào DynamoDB |
| product-updated-queue | Product Service | Order Service | Product thay đổi → Order Service invalidate Redis cache |

---

## 5. Luồng xử lý chi tiết

### 5.1 Luồng đặt hàng Online

```
Khách chọn sản phẩm → thêm giỏ hàng
        ↓
Khách chọn cửa hàng + địa chỉ giao hàng → xác nhận đặt hàng
        ↓
[Store & Inventory Service] Kiểm tra tồn kho, trừ tạm thời
        ↓
[Order Service] Tạo đơn hàng (status: PENDING, snapshot product info), set TTL 1 ngày
        ↓
Publish → order-confirmed-queue
        ↓
[Payment Service] Xử lý thanh toán mock
        ↓
Publish → payment-queue
        ↓
[Order Service] Cập nhật status: CONFIRMED
[Notification Service] Gửi email xác nhận đặt hàng
        ↓
Nhân viên cập nhật trạng thái (PREPARING → SHIPPING → DELIVERED)
        ↓ (mỗi lần thay đổi)
Publish → order-status-queue
        ↓
[Notification Service] Gửi email theo trạng thái
[Analytics Service] Ghi nhận vào DynamoDB
```

### 5.2 Luồng tự động hủy đơn (Auto-cancel)

```
Đơn hàng PENDING quá 1 ngày chưa được xác nhận
        ↓
[Order Service] Background job cập nhật status: CANCELLED
        ↓
[Store & Inventory Service] Hoàn trả tồn kho đã trừ tạm thời
        ↓
Publish → order-status-queue
        ↓
[Notification Service] Gửi email thông báo hủy đơn
```

### 5.3 Luồng mua tại quầy (POS)

```
Nhân viên chọn sản phẩm + số lượng trên web
        ↓
[Order Service] Tạo đơn POS (type: POS, customer: WALK_IN)
        ↓
[Store & Inventory Service] Trừ tồn kho cửa hàng
        ↓
Publish → analytics-queue
        ↓
[Analytics Service] Ghi nhận đơn POS vào DynamoDB
        ↓
[Order Service] Trả về dữ liệu hóa đơn → Frontend in hóa đơn
```

### 5.4 Luồng hoàn hàng

```
Khách yêu cầu hoàn hàng
        ↓
[Order Service] Cập nhật status: RETURN_REQUESTED
        ↓
Publish → order-status-queue
        ↓
[Notification Service] Gửi email xác nhận yêu cầu hoàn hàng
        ↓
Nhân viên cập nhật: RETURN_PICKING → RETURN_SHIPPING → RETURN_COMPLETED
        ↓ (mỗi lần thay đổi)
Publish → order-status-queue
        ↓
[Store & Inventory Service] Hoàn trả tồn kho khi RETURN_COMPLETED
[Notification Service] Gửi email hoàn hàng hoàn tất
```

### 5.5 Luồng cache Product (Write-through + Invalidation)

```
Khi Product được tạo/update:
[Product Service] Ghi MySQL → ghi Redis (write-through)
        ↓
Publish → product-updated-queue
        ↓
[Order Service] Consume → invalidate Redis cache tương ứng

Khi Order Service cần thông tin product:
Đọc Redis → cache hit → trả về
           → cache miss → HTTP call sang Product Service → ghi lại Redis
```

### 5.6 Luồng nhập hàng từ kho tổng vào cửa hàng

```
Admin/Cửa hàng trưởng tạo yêu cầu nhập hàng
        ↓
[Store & Inventory Service] Kiểm tra tồn kho kho tổng
        ↓
Trừ số lượng kho tổng + Cộng số lượng cửa hàng (atomic)
```

---

## 6. Phân quyền theo Role

| Role | Quyền truy cập |
|---|---|
| Khách hàng | Xem sản phẩm, đặt hàng, xem lịch sử đơn hàng của mình |
| Nhân viên cửa hàng | Xử lý đơn online, tạo đơn POS, cập nhật trạng thái đơn |
| Cửa hàng trưởng | Toàn quyền cửa hàng + xem báo cáo cửa hàng mình + nhập hàng từ kho tổng |
| Admin | Toàn quyền hệ thống + xem báo cáo toàn chuỗi + quản lý kho tổng |

---

## 7. Infrastructure (Kubernetes)

```
k8s Cluster
├── Namespace: api
│   ├── identity-service (Deployment)
│   ├── user-service (Deployment)
│   ├── product-service (Deployment)
│   ├── store-inventory-service (Deployment)
│   └── order-service (Deployment)
├── Namespace: workers
│   ├── payment-worker (Deployment)
│   ├── notification-worker (Deployment)
│   └── analytics-worker (Deployment)
├── Namespace: gateway
│   └── envoy-gateway (Deployment)
└── Namespace: data
    ├── mysql (StatefulSet)
    └── scylladb (StatefulSet)
```

- **Redis**: AWS ElastiCache for Redis (managed)
- **DynamoDB**: AWS managed
- **AWS SQS**: AWS managed
- **AWS Cognito**: AWS managed
- **AWS SES**: AWS managed
- **AWS S3**: lưu ảnh sản phẩm, truy cập qua Pre-Signed URL (Lambda + API Gateway)

---

## 8. Backend Architecture (Clean Architecture + CQRS)

### 8.1 Cấu trúc chung cho Web API Services

```
ServiceName/
├── ServiceName.Domain/
│   ├── Entities/
│   ├── Enums/
│   └── Events/            # Domain events
│
├── ServiceName.Application/
│   ├── Commands/          # Write operations
│   ├── Queries/           # Read operations
│   ├── Handlers/          # MediatR handlers
│   ├── DTOs/
│   └── Interfaces/        # IRepository, ISqsPublisher, ...
│
├── ServiceName.Infrastructure/
│   ├── Persistence/       # DbContext, Repository implementations
│   ├── Messaging/         # SQS publisher implementation
│   └── ExternalServices/  # HTTP clients gọi service khác
│
└── ServiceName.API/
    ├── Controllers/
    └── Middleware/        # Auth, error handling, logging
```

### 8.2 Cấu trúc cho Worker Services

Worker không có HTTP layer nên chỉ cần 3 layer:

```
WorkerName/
├── WorkerName.Application/
│   ├── Handlers/          # Xử lý message từ SQS
│   ├── DTOs/
│   └── Interfaces/
│
├── WorkerName.Infrastructure/
│   ├── Messaging/         # SQS consumer implementation
│   └── ExternalServices/  # AWS SES, HTTP clients, ...
│
└── WorkerName.Worker/
    ├── Workers/           # BackgroundService implementations
    └── Program.cs         # DI setup, host configuration
```

### 8.3 CQRS Flow

```
HTTP Request
    ↓
Controller → Send(Command / Query) via MediatR
                ↓
          Handler (Application layer)
                ↓
          IRepository / IExternalService (interfaces)
                ↓
          Infrastructure implements
                ↓
          Database / AWS SQS / External API
```

### 8.4 Dependency Rule

```
API → Application → Domain
Infrastructure → Application → Domain

Domain không phụ thuộc vào bất kỳ layer nào
```
