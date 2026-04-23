# Review Issues — Hệ thống E-Commerce Microservice

> Tài liệu tổng hợp các vấn đề cần xem lại và cải thiện, dựa trên review BA_Document.md, Technical_Architect.md và Database_Schema.md.

---

## Mục lục

1. [Vấn đề nghiêm trọng (Cần sửa trước khi implement)](#1-vấn-đề-nghiêm-trọng)
2. [Vấn đề thiết kế (Nên cải thiện)](#2-vấn-đề-thiết-kế)
3. [Thiếu sót trong tài liệu (Cần bổ sung)](#3-thiếu-sót-trong-tài-liệu)
4. [Gợi ý cải thiện (Nice to have)](#4-gợi-ý-cải-thiện)

---

## 1. Vấn đề nghiêm trọng

### 1.1 Cache strategy mâu thuẫn giữa BA và Technical Architect

**Tài liệu liên quan:** BA_Document.md (mục 9), Technical_Architect.md (mục 5.5)

**Mô tả:**
- BA Document mô tả **write-through**: Product Service ghi MySQL + ghi Redis cùng lúc.
- Đồng thời lại mô tả: Product Service publish SQS → Order Service **invalidate** cache.
- Đây là 2 strategy khác nhau đang bị trộn lẫn.

**Tại sao là vấn đề:**
- Write-through nghĩa là Product Service ghi trực tiếp vào Redis → data trong Redis luôn mới nhất → không cần invalidate.
- Nếu cần invalidate qua SQS, đó là **cache-aside** pattern, không phải write-through.
- Nếu triển khai cả 2 cùng lúc sẽ gây nhầm lẫn cho developer và có thể dẫn đến bug cache inconsistency.

**Gợi ý:**
Chọn 1 trong 2 approach:

| Approach | Ưu điểm | Nhược điểm |
|---|---|---|
| **Cache-aside + SQS invalidation** (khuyến nghị) | Product Service không cần biết Redis của Order Service, đúng nguyên tắc service independence | Có khoảng thời gian cache stale (eventual consistency) |
| **Write-through** | Data luôn mới nhất trong cache | Product Service phải biết và ghi vào Redis của Order Service → coupling giữa 2 service |

---

### 1.2 ScyllaDB không hỗ trợ cross-partition transaction

**Tài liệu liên quan:** BA_Document.md (mục 3), Database_Schema.md (Store & Inventory Service)

**Mô tả:**
Luồng nhập hàng từ kho tổng vào cửa hàng yêu cầu **atomic**, nhưng thực tế cần cập nhật:
1. `inventory_by_store` cho kho tổng (quantity -= N)
2. `inventory_by_variant` cho kho tổng (quantity -= N)
3. `inventory_by_store` cho cửa hàng (quantity += N)
4. `inventory_by_variant` cho cửa hàng (quantity += N)
5. `inventory_transactions` cho kho tổng (TRANSFER_OUT)
6. `inventory_transactions` cho cửa hàng (TRANSFER_IN)

ScyllaDB **không có multi-partition transaction**. Nếu bước 3 fail sau khi bước 1-2 đã thành công, kho tổng bị trừ nhưng cửa hàng không được cộng → mất hàng.

**Gợi ý:**

**Option A — Saga pattern với idempotent operations:**
```
1. Ghi inventory_transactions (TRANSFER_OUT + TRANSFER_IN) làm source of truth
2. Cập nhật inventory_by_store + inventory_by_variant cho kho tổng
3. Cập nhật inventory_by_store + inventory_by_variant cho cửa hàng
4. Nếu bước nào fail → retry dựa trên transaction log
```

**Option B — Lightweight Transaction (LWT) cho từng bước:**
```cql
UPDATE inventory_by_store 
SET quantity = quantity - N 
WHERE store_id = ? AND product_variant_id = ?
IF quantity >= N;
```
Kết hợp với compensation logic khi bước sau fail.

---

### 1.3 Race condition trong luồng đặt hàng online

**Tài liệu liên quan:** Technical_Architect.md (mục 5.1)

**Mô tả:**
Luồng hiện tại:
```
[Store & Inventory Service] Trừ tồn kho tạm thời
        ↓
[Order Service] Tạo đơn hàng
```

Nếu bước tạo đơn hàng fail (network error, Order Service down, DB timeout...) sau khi inventory đã bị trừ → inventory bị trừ mà không có đơn hàng tương ứng.

**Gợi ý:**

**Option A — Saga pattern:**
```
1. Order Service tạo đơn hàng status = CREATED (chưa trừ kho)
2. Gọi Store & Inventory Service trừ kho
3. Nếu trừ kho thành công → update status = PENDING
4. Nếu trừ kho fail → update status = FAILED
5. Nếu bước 3 fail (network) → background job reconcile
```

**Option B — Reservation pattern:**
```
1. Store & Inventory tạo reservation (reserved_quantity, expires in 5 phút)
2. Order Service tạo đơn hàng với reservation_id
3. Confirm reservation → trừ kho thật
4. Reservation hết hạn → tự động hoàn trả
```

---

### 1.4 Concurrent inventory deduction gây overselling

**Tài liệu liên quan:** Database_Schema.md (Store & Inventory Service)

**Mô tả:**
Khi nhiều khách cùng mua 1 variant tại 1 cửa hàng đồng thời, nếu không có cơ chế kiểm soát concurrency:
- Tồn kho còn 1, 2 khách cùng đọc quantity = 1 → cả 2 đều đặt hàng thành công → overselling.

Tài liệu chưa đề cập cách xử lý concurrent inventory deduction.

**Gợi ý:**
Dùng ScyllaDB **Lightweight Transaction (LWT)** hoặc **Compare-and-Set**:
```cql
UPDATE inventory_by_store 
SET quantity = ? 
WHERE store_id = ? AND product_variant_id = ?
IF quantity = ?;  -- optimistic locking
```

Hoặc dùng conditional update:
```cql
UPDATE inventory_by_store 
SET quantity = quantity - ? 
WHERE store_id = ? AND product_variant_id = ?
IF quantity >= ?;
```

**Lưu ý:** LWT trong ScyllaDB có performance overhead (~4x slower). Nếu throughput cao, cân nhắc dùng Redis distributed lock phía trước ScyllaDB.

---

## 2. Vấn đề thiết kế

### 2.1 Bảng `payments` vi phạm database-per-service

**Tài liệu liên quan:** Database_Schema.md (Order Service), Technical_Architect.md (mục 2)

**Mô tả:**
- Bảng `payments` nằm trong database của **Order Service**.
- Payment Service được liệt kê là service riêng (Worker) nhưng **không có database**.
- Khi tích hợp payment gateway thật (VNPay, MoMo, Stripe), Payment Service sẽ cần lưu: transaction ID từ gateway, retry history, refund records, webhook logs...

**Gợi ý:**
- Di chuyển bảng `payments` sang Payment Service với database riêng (MySQL hoặc PostgreSQL).
- Order Service chỉ lưu `payment_status` và `payment_id` (reference).
- Payment Service publish event qua SQS khi payment status thay đổi → Order Service consume và cập nhật.

---

### 2.2 Auto-cancel thiếu idempotency và race condition protection

**Tài liệu liên quan:** Technical_Architect.md (mục 5.2)

**Mô tả:**
Background job auto-cancel đơn PENDING quá hạn có các rủi ro:
1. Đơn hàng được confirm **đúng lúc** background job đang chạy → cancel đơn đã confirmed.
2. Job chạy 2 lần (retry, duplicate) → hoàn trả inventory 2 lần.
3. Hoàn trả inventory fail → đơn bị cancel nhưng inventory không được hoàn.

**Gợi ý:**
```sql
-- Optimistic locking: chỉ cancel nếu status vẫn là PENDING
UPDATE orders 
SET status = 'CANCELLED', updated_at = NOW() 
WHERE id = ? AND status = 'PENDING' AND expires_at < NOW();
-- Kiểm tra affected rows > 0 trước khi hoàn trả inventory
```

Kết hợp:
- Lưu `cancellation_id` (UUID) vào order để đảm bảo idempotent inventory restoration.
- Dùng **outbox pattern**: ghi cancel event vào outbox table trong cùng transaction với update order → background job publish event → inventory service consume và hoàn trả.

---

### 2.3 `order-confirmed-queue` và `analytics-queue` có thể gộp

**Tài liệu liên quan:** Technical_Architect.md (mục 4)

**Mô tả:**
- `order-status-queue` đã publish mỗi khi trạng thái đơn thay đổi → Analytics Service consume.
- `analytics-queue` riêng cho đơn POS.
- Có thể gộp bằng cách cho đơn POS cũng publish qua `order-status-queue` với status = DELIVERED (vì POS hoàn thành ngay).

**Lợi ích:** Giảm số queue cần quản lý, Analytics Service chỉ cần consume 1 queue.

---

### 2.4 Thiếu order status transition validation

**Tài liệu liên quan:** BA_Document.md (mục 4), Database_Schema.md (orders table)

**Mô tả:**
Order status có 10 giá trị nhưng tài liệu chưa define rõ **state machine** — trạng thái nào được phép chuyển sang trạng thái nào.

Ví dụ các transition không hợp lệ cần ngăn chặn:
- `DELIVERED` → `PENDING` (quay ngược)
- `CANCELLED` → `PREPARING` (đơn đã hủy không thể xử lý tiếp)
- `SHIPPING` → `RETURN_COMPLETED` (bỏ qua các bước hoàn hàng)

**Gợi ý:**
Bổ sung state machine diagram:
```
PENDING → CONFIRMED → PREPARING → SHIPPING → DELIVERED
PENDING → CANCELLED
DELIVERED → RETURN_REQUESTED → RETURN_PICKING → RETURN_SHIPPING → RETURN_COMPLETED
```

Implement validation trong Order Service domain layer:
```csharp
public static readonly Dictionary<OrderStatus, OrderStatus[]> AllowedTransitions = new()
{
    { OrderStatus.PENDING, new[] { OrderStatus.CONFIRMED, OrderStatus.CANCELLED } },
    { OrderStatus.CONFIRMED, new[] { OrderStatus.PREPARING, OrderStatus.CANCELLED } },
    { OrderStatus.PREPARING, new[] { OrderStatus.SHIPPING } },
    { OrderStatus.SHIPPING, new[] { OrderStatus.DELIVERED } },
    { OrderStatus.DELIVERED, new[] { OrderStatus.RETURN_REQUESTED } },
    // ... return flow
};
```

---

### 2.5 `price_history` thiếu soft delete fields nhưng các bảng khác đều có

**Tài liệu liên quan:** Database_Schema.md (Product Service)

**Mô tả:**
Bảng `price_history` không có `is_deleted`, `deleted_at`, `updated_at` trong khi BA Document quy định "mọi entity đều có `created_at`, `updated_at`, `is_deleted`, `deleted_at`".

**Gợi ý:**
- Nếu `price_history` là append-only log (không bao giờ xóa/sửa) → document rõ đây là exception, không cần soft delete.
- Nếu muốn đồng nhất → thêm các fields theo quy tắc chung.

---

## 3. Thiếu sót trong tài liệu

### 3.1 Thiếu document synchronous service-to-service calls

**Mô tả:**
Tài liệu chỉ document async communication qua SQS. Các HTTP call đồng bộ giữa services chưa được liệt kê rõ.

**Các sync call cần document:**

| Caller | Callee | Mục đích | Khi nào |
|---|---|---|---|
| Order Service | Store & Inventory Service | Kiểm tra + trừ tồn kho | Khi tạo đơn hàng |
| Order Service | Product Service | Lấy thông tin product (cache miss) | Khi Redis cache miss |
| Order Service | User Service | Lấy thông tin user/address | Khi tạo đơn hàng online |
| Order Service | Store & Inventory Service | Hoàn trả tồn kho | Khi cancel/return đơn |

**Tại sao quan trọng:**
- Sync call tạo runtime dependency → nếu callee down, caller cũng fail.
- Cần plan cho **circuit breaker**, **timeout**, **retry** cho từng call.
- Ảnh hưởng đến deployment order và health check.

---

### 3.2 Thiếu error handling và retry strategy

**Mô tả:**
Tài liệu chưa đề cập:
- **Dead Letter Queue (DLQ)** cho SQS: message xử lý thất bại sau N lần retry sẽ đi đâu?
- **Retry policy**: exponential backoff? max retries?
- **Poison message handling**: message format sai, data không hợp lệ.
- **Circuit breaker**: khi downstream service không phản hồi.

**Gợi ý bổ sung:**
```
Mỗi SQS queue cần có:
├── Main queue (visibility timeout: 30s)
├── DLQ (maxReceiveCount: 3)
└── Alarm khi DLQ có message
```

---

### 3.3 Thiếu observability strategy

**Mô tả:**
Hệ thống microservice cần:
- **Distributed tracing**: theo dõi 1 request đi qua nhiều service (OpenTelemetry + Jaeger/X-Ray).
- **Centralized logging**: aggregate logs từ tất cả services (ELK Stack hoặc CloudWatch).
- **Metrics & alerting**: request rate, error rate, latency per service (Prometheus + Grafana hoặc CloudWatch).
- **Health check endpoints**: `/health` cho mỗi service, k8s liveness/readiness probes.

---

### 3.4 Thiếu API versioning strategy

**Mô tả:**
Khi hệ thống phát triển, API sẽ thay đổi. Cần quy ước:
- URL versioning (`/api/v1/products`) hay header versioning?
- Backward compatibility policy: hỗ trợ bao nhiêu version cũ?
- Deprecation process.

---

### 3.5 Thiếu security concerns ngoài JWT

**Mô tả:**
Ngoài JWT validation ở API Gateway, cần bổ sung:
- **Rate limiting** per user/IP tại Envoy Gateway.
- **Input validation** cho tất cả API endpoints.
- **SQL injection protection** (EF Core parameterized queries đã handle, nhưng cần document).
- **CORS policy** cho Vue 3 frontend.
- **Secrets management**: connection strings, API keys lưu ở đâu? (AWS Secrets Manager / k8s Secrets).
- **Network policy** trong k8s: service nào được gọi service nào.

---

### 3.6 Thiếu data migration và seeding strategy

**Mô tả:**
- Làm sao migrate schema khi deploy version mới? (EF Core Migrations cho MySQL, CQL scripts cho ScyllaDB)
- Seed data ban đầu: categories, stores, warehouse, admin account.
- Rollback strategy khi migration fail.

---

## 4. Gợi ý cải thiện

### 4.1 Thêm monthly aggregation cho Analytics

**Vấn đề:** Query 30 ngày rồi sum ở application layer sẽ chậm khi data lớn.

**Gợi ý:** Thêm bảng `monthly_product_stats` và `monthly_category_stats` trong DynamoDB, được aggregate bởi scheduled job chạy hàng đêm hoặc cuối tháng.

---

### 4.2 Cân nhắc dùng Redis cho giỏ hàng thay vì MySQL

**Vấn đề:** Giỏ hàng là data tạm thời, thay đổi thường xuyên, không cần ACID transaction phức tạp.

**Gợi ý:**
- Lưu cart trong Redis (hash hoặc JSON) → read/write nhanh hơn MySQL.
- Set TTL cho cart (ví dụ 30 ngày không hoạt động → tự xóa).
- Giảm load cho MySQL Order Service database.

**Trade-off:** Mất cart nếu Redis restart (có thể chấp nhận được, hoặc dùng Redis persistence).

---

### 4.3 Bổ sung Notification Service database

**Vấn đề:** Notification Service không có database → không lưu lịch sử gửi email, không biết email nào đã gửi thành công/thất bại.

**Gợi ý:** Thêm database nhẹ (DynamoDB hoặc MySQL) để lưu:
- Notification history (email sent, status, retry count)
- User notification preferences (opt-out)
- Template management

---

### 4.4 Cân nhắc tách Identity Service và User Service

**Vấn đề:** Identity Service chỉ wrap AWS Cognito. Nếu chỉ validate JWT ở Envoy Gateway thì Identity Service có thể không cần là 1 service riêng.

**Gợi ý:**
- Nếu Identity Service chỉ forward request đến Cognito → gộp logic vào Envoy Gateway (JWT validation) + User Service (profile management).
- Nếu Identity Service có business logic riêng (custom auth flow, token refresh, session management) → giữ nguyên.

---

### 4.5 Document rõ POS flow khi mất kết nối

**Vấn đề:** POS chạy trên web, nếu mất internet thì nhân viên không thể bán hàng.

**Gợi ý:** Cân nhắc:
- Offline mode cho POS (Service Worker + IndexedDB)?
- Hay chấp nhận POS phải có internet (đơn giản hơn)?
- Document rõ quyết định này.

---

*Tài liệu này được tạo dựa trên review ngày 23/04/2026. Cần được cập nhật khi các vấn đề được giải quyết.*
