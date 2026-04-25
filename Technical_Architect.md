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
                        │  [payment-queue]            [product-updated-queue]     │
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

| Service                   | Loại    | Database      | Mô tả                                                                                             |
| ---------------------------| ---------| ---------------| ---------------------------------------------------------------------------------------------------|
| Identity Service          | Web API | AWS Cognito   | Xác thực, phân quyền JWT                                                                          |
| User Service              | Web API | MySQL         | Quản lý profile, địa chỉ người dùng, store_staff (dùng Cognito `sub` làm id)                      |
| Product Service           | Web API | MySQL         | Quản lý sản phẩm, danh mục 3 cấp, variant + SKU, giá (product level + variant level), lịch sử giá |
| Store & Inventory Service | Web API | ScyllaDB      | Quản lý cửa hàng, kho tổng, tồn kho theo từng cửa hàng, nhập hàng từ kho tổng                     |
| Order Service             | Web API | MySQL + Redis | Tạo và quản lý đơn hàng (online + POS), giỏ hàng, cache product, auto-cancel                      |
| Payment Service           | Worker  | MySQL         | Consume queue, xử lý thanh toán (mock), lưu lịch sử giao dịch với payment gateway                |
| Notification Service      | Worker  | -             | Consume queue, gửi email qua AWS SES                                                              |
| Analytics Service         | Worker  | DynamoDB      | Consume queue, ghi nhận và tổng hợp dữ liệu bán hàng                                              |

---

## 3. Phân chia Database

| Database            | Service sử dụng                              | Lý do                                                                                                                                           |
| ---------------------| ----------------------------------------------| -------------------------------------------------------------------------------------------------------------------------------------------------|
| MySQL               | User Service, Product Service, Order Service | Dữ liệu có quan hệ, cần ACID transaction                                                                                                        |
| Redis (ElastiCache) | Order Service                                | Cache thông tin product (tên, giá product, giá variant, SKU, ảnh, variant attributes) — cache-aside + SQS invalidation, cache miss gọi HTTP sang Product Service |
| ScyllaDB            | Store & Inventory Service                    | Tồn kho cần read/write cực nhanh, scale tốt khi nhiều request đồng thời                                                                         |
| DynamoDB            | Analytics Service                            | Event log, schema linh hoạt, không cần join phức tạp                                                                                            |

---

## 4. SQS Queues

| Queue                 | Publisher       | Consumer                                | Mô tả                                                       |
| -----------------------| -----------------| -----------------------------------------| -------------------------------------------------------------|
| order-confirmed-queue | Order Service   | Payment Service                         | Kích hoạt xử lý thanh toán sau khi đơn được xác nhận        |
| payment-queue         | Payment Service | Order Service, Notification Service     | Kết quả thanh toán → cập nhật trạng thái đơn + gửi email    |
| order-status-queue    | Order Service   | Notification Service, Analytics Service | Mỗi lần trạng thái đơn thay đổi → gửi email + ghi analytics (bao gồm cả đơn POS) |
| product-updated-queue | Product Service | Order Service                           | Product thay đổi → Order Service invalidate Redis cache     |

---

## 4.1 SQS Message Schema

Tất cả message đều dùng **JSON format** với các field chuẩn:

### Standard Message Fields

```json
{
  "version": "1.0",
  "message_id": "uuid-v4",
  "correlation_id": "uuid-v4",
  "timestamp": "2025-01-15T10:30:00Z",
  "event_type": "event.name",
  "data": { ... }
}
```

**Field mô tả:**
- `version`: Message format version (để backward compatibility)
- `message_id`: Unique ID của message (UUID v4)
- `correlation_id`: ID để trace xuyên suốt flow (ví dụ: order_id)
- `timestamp`: ISO 8601 timestamp khi message được publish
- `event_type`: Tên event (ví dụ: "order.confirmed")
- `data`: Payload cụ thể cho từng event type

---

### 4.1.1 order-confirmed-queue

**Event Type:** `order.confirmed`

**Publisher:** Order Service

**Consumer:** Payment Service

**Message Schema:**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440000",
  "correlation_id": "order-123",
  "timestamp": "2025-01-15T10:30:00Z",
  "event_type": "order.confirmed",
  "data": {
    "order_id": "order-123",
    "user_id": "user-456",
    "store_id": "store-789",
    "order_type": "ONLINE",
    "total_amount": 150000.00,
    "currency": "VND",
    "items": [
      {
        "product_variant_id": "variant-001",
        "product_name": "Pepsi",
        "sku": "PEPSI-330-COLA",
        "quantity": 2,
        "unit_price": 8000.00,
        "subtotal": 16000.00
      }
    ],
    "shipping_address": {
      "street": "123 Đường ABC",
      "district": "Quận 1",
      "city": "TP. Hồ Chí Minh",
      "latitude": 10.7769,
      "longitude": 106.7009
    },
    "created_at": "2025-01-15T10:30:00Z"
  }
}
```

---

### 4.1.2 payment-queue

**Event Type:** `payment.completed` / `payment.failed`

**Publisher:** Payment Service

**Consumer:** Order Service, Notification Service

**Message Schema (Payment Completed):**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440001",
  "correlation_id": "order-123",
  "timestamp": "2025-01-15T10:31:00Z",
  "event_type": "payment.completed",
  "data": {
    "payment_id": "payment-999",
    "order_id": "order-123",
    "user_id": "user-456",
    "payment_method": "ONLINE",
    "amount": 150000.00,
    "currency": "VND",
    "status": "SUCCESS",
    "gateway_transaction_id": "GTX-123456789",
    "gateway_response": {
      "code": "00",
      "message": "Transaction successful"
    },
    "paid_at": "2025-01-15T10:31:00Z"
  }
}
```

**Message Schema (Payment Failed):**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440002",
  "correlation_id": "order-123",
  "timestamp": "2025-01-15T10:31:00Z",
  "event_type": "payment.failed",
  "data": {
    "payment_id": "payment-999",
    "order_id": "order-123",
    "user_id": "user-456",
    "payment_method": "ONLINE",
    "amount": 150000.00,
    "currency": "VND",
    "status": "FAILED",
    "error_code": "INSUFFICIENT_BALANCE",
    "error_message": "Số dư không đủ",
    "failed_at": "2025-01-15T10:31:00Z"
  }
}
```

---

### 4.1.3 order-status-queue

**Event Type:** `order.status.changed`

**Publisher:** Order Service

**Consumer:** Notification Service, Analytics Service

**Message Schema:**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440003",
  "correlation_id": "order-123",
  "timestamp": "2025-01-15T11:00:00Z",
  "event_type": "order.status.changed",
  "data": {
    "order_id": "order-123",
    "user_id": "user-456",
    "store_id": "store-789",
    "order_type": "ONLINE",
    "old_status": "CONFIRMED",
    "new_status": "PREPARING",
    "total_amount": 150000.00,
    "currency": "VND",
    "items": [
      {
        "product_variant_id": "variant-001",
        "product_name": "Pepsi",
        "sku": "PEPSI-330-COLA",
        "quantity": 2,
        "unit_price": 8000.00,
        "subtotal": 16000.00,
        "category_id": "cat-123",
        "category_path": "/1/2/3"
      }
    ],
    "updated_at": "2025-01-15T11:00:00Z",
    "additional_info": {
      "cancelled_by": null,
      "cancel_reason": null,
      "return_reason": null
    }
  }
}
```

**Special Cases:**

**Auto-cancel:**
```json
{
  "event_type": "order.status.changed",
  "data": {
    "order_id": "order-123",
    "old_status": "PENDING",
    "new_status": "CANCELLED",
    "additional_info": {
      "cancelled_by": "SYSTEM",
      "cancel_reason": "Quá hạn xác nhận"
    }
  }
}
```

**Proactive cancellation:**
```json
{
  "event_type": "order.status.changed",
  "data": {
    "order_id": "order-123",
    "old_status": "CONFIRMED",
    "new_status": "CANCELLED",
    "additional_info": {
      "cancelled_by": "CUSTOMER",
      "cancel_reason": "Khách hàng yêu cầu hủy"
    }
  }
}
```

**Return completed:**
```json
{
  "event_type": "order.status.changed",
  "data": {
    "order_id": "order-123",
    "old_status": "RETURN_SHIPPING",
    "new_status": "RETURN_COMPLETED",
    "additional_info": {
      "return_reason": "Sản phẩm bị lỗi"
    }
  }
}
```

---

### 4.1.4 product-updated-queue

**Event Type:** `product.updated` / `product.deleted`

**Publisher:** Product Service

**Consumer:** Order Service

**Message Schema (Product Updated):**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440004",
  "correlation_id": "product-123",
  "timestamp": "2025-01-15T12:00:00Z",
  "event_type": "product.updated",
  "data": {
    "product_id": "product-123",
    "product_name": "Pepsi",
    "base_price": 8000.00,
    "status": "ACTIVE",
    "category_id": "cat-123",
    "category_path": "/1/2/3",
    "variants": [
      {
        "variant_id": "variant-001",
        "sku": "PEPSI-330-COLA",
        "price": 8000.00,
        "attributes": {
          "Dung tích": "330ml",
          "Hương vị": "Cola"
        }
      }
    ],
    "updated_at": "2025-01-15T12:00:00Z"
  }
}
```

**Message Schema (Product Deleted - Soft Delete):**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440005",
  "correlation_id": "product-123",
  "timestamp": "2025-01-15T12:00:00Z",
  "event_type": "product.deleted",
  "data": {
    "product_id": "product-123",
    "deleted_at": "2025-01-15T12:00:00Z"
  }
}
```

**Message Schema (Variant Updated):**

```json
{
  "version": "1.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440006",
  "correlation_id": "variant-001",
  "timestamp": "2025-01-15T12:00:00Z",
  "event_type": "product.variant.updated",
  "data": {
    "product_id": "product-123",
    "variant_id": "variant-001",
    "sku": "PEPSI-330-COLA",
    "price": 9000.00,
    "attributes": {
      "Dung tích": "330ml",
      "Hương vị": "Cola"
    },
    "updated_at": "2025-01-15T12:00:00Z"
  }
}
```

---

### 4.1.5 Message Processing Guidelines

**Idempotency:**
- Consumer phải xử lý message idempotently (cùng message_id chỉ xử lý 1 lần)
- Dùng `message_id` để dedup (lưu vào Redis hoặc database)

**Error Handling:**
- Nếu consumer fail → message sẽ được retry (max 3 lần)
- Sau 3 lần fail → chuyển vào DLQ
- DLQ cần có monitoring và manual intervention

**Versioning:**
- Khi thay đổi message schema → tăng `version`
- Consumer phải hỗ trợ backward compatibility (cả version cũ và mới)
- Sau khi tất cả consumer đã migrate → có thể bỏ support version cũ

**Correlation ID:**
- `correlation_id` dùng để trace xuyên suốt flow
- Ví dụ: `order-123` → tất cả message liên quan đến order này đều dùng correlation_id này
- Dùng cho distributed tracing và debugging

**Timestamp:**
- `timestamp` là thời điểm message được publish (không phải thời điểm event xảy ra)
- Event time có thể nằm trong `data` field (ví dụ: `created_at`, `updated_at`)

---

## 5. Luồng xử lý chi tiết

### 5.1 Luồng đặt hàng Online

```
Khách chọn sản phẩm → thêm giỏ hàng
        ↓
Khách chọn cửa hàng + địa chỉ giao hàng → xác nhận đặt hàng
        ↓
[Order Service] Tạo đơn hàng (status: CREATED, snapshot product info)
        ↓
[Store & Inventory Service] Kiểm tra tồn kho, trừ tạm thời (conditional update: IF quantity >= N)
        ↓
Thành công → [Order Service] Cập nhật status: PENDING, set expires_at (+ 1 ngày)
Thất bại  → [Order Service] Cập nhật status: FAILED
        ↓ (nếu PENDING)
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

### 5.2.1 Luồng hủy đơn chủ động (Proactive Cancellation)

```
Khách hàng / Nhân viên / Cửa hàng trưởng / Admin yêu cầu hủy đơn
        ↓
[Order Service] Validate quyền hủy + trạng thái đơn:
  - Khách hàng: PENDING hoặc CONFIRMED
  - Nhân viên/Cửa hàng trưởng: PENDING, CONFIRMED, hoặc PREPARING
  - Admin: Bất kỳ trạng thái nào (trừ DELIVERED quá 2 ngày, RETURN_COMPLETED)
        ↓
[Order Service] Cập nhật status: CANCELLED
  - Lưu cancelled_by (CUSTOMER/STAFF/MANAGER/ADMIN/SYSTEM)
  - Lưu cancel_reason (lý do hủy)
  - Lưu cancelled_at (thời điểm hủy)
        ↓
[Store & Inventory Service] Hoàn trả tồn kho về cửa hàng
  - inventory_by_store: quantity += N
  - inventory_by_variant: quantity += N
  - inventory_transactions: INSERT (type: CANCEL, quantity_change: +N)
        ↓
Publish → order-status-queue
        ↓
[Notification Service] Gửi email thông báo hủy đơn cho khách hàng
        ↓
Nếu đã thanh toán → Nhân viên hoàn tiền thủ công
  (Giai đoạn sau → Payment Service hoàn tiền tự động)
```

**Lưu ý:**
- Đơn POS không hỗ trợ hủy (đã giao hàng ngay lập tức)
- Khi hủy đơn → luồng hoàn trả inventory giống auto-cancel
- Cần lưu `cancelled_by` và `cancel_reason` để audit và truy vết

### 5.3 Luồng mua tại quầy (POS)

```
Nhân viên chọn sản phẩm + số lượng trên web
        ↓
[Order Service] Tạo đơn POS (type: POS, customer: WALK_IN, status: DELIVERED)
        ↓
[Store & Inventory Service] Trừ tồn kho cửa hàng (conditional update)
        ↓
Publish → order-status-queue (status: DELIVERED)
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

### 5.5 Luồng cache Product (Cache-aside + SQS Invalidation)

```
Khi Product được tạo/update:
[Product Service] Ghi MySQL
        ↓
Publish → product-updated-queue
        ↓
[Order Service] Consume → invalidate Redis cache tương ứng

Khi Order Service cần thông tin product:
Đọc Redis → cache hit → trả về
           → cache miss → HTTP call sang Product Service → ghi vào Redis → trả về
```

> Product Service **không** ghi trực tiếp vào Redis của Order Service (đảm bảo service independence).

### 5.6 Luồng nhập hàng từ kho tổng vào cửa hàng

```
Admin/Cửa hàng trưởng tạo yêu cầu nhập hàng
        ↓
[Store & Inventory Service] Kiểm tra tồn kho kho tổng
        ↓
Ghi inventory_transactions (TRANSFER_OUT + TRANSFER_IN) làm source of truth
        ↓
Trừ số lượng kho tổng (conditional update: IF quantity >= N)
        ↓
Cộng số lượng cửa hàng
        ↓
Nếu bước nào fail → retry dựa trên transaction log (Saga pattern)
```

---

## 6. Phân quyền theo Role

| Role               | Quyền truy cập                                                          |
| --------------------| -------------------------------------------------------------------------|
| Khách hàng         | Xem sản phẩm, đặt hàng, xem lịch sử đơn hàng của mình                   |
| Nhân viên cửa hàng | Xử lý đơn online, tạo đơn POS, cập nhật trạng thái đơn                  |
| Cửa hàng trưởng    | Toàn quyền cửa hàng + xem báo cáo cửa hàng mình + nhập hàng từ kho tổng |
| Admin              | Toàn quyền hệ thống + xem báo cáo toàn chuỗi + quản lý kho tổng         |

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

### 8.5 Base Entity & Soft Delete

Tất cả entity kế thừa từ BaseEntity để đồng nhất toàn hệ thống:

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}
```

- Khi xóa: update `IsDeleted = true, DeletedAt = now()` (không xóa thật)
- Sử dụng **EF Core Global Query Filter** để tự động lọc `WHERE is_deleted = false` cho mọi query
- Áp dụng cho tất cả tables trong cả 3 MySQL services (User, Product, Order)
