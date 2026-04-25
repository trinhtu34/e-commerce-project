# Review Issues — Hệ thống E-Commerce Microservice

> Tài liệu tổng hợp các vấn đề cần xem lại và cải thiện.
>
> Review lần 3: 23/04/2026 — Deep review toàn bộ BA_Document.md, Technical_Architect.md, Database_Schema.md.
> Review lần 4: 25/04/2026 — Giải quyết issue 3.1: Tách payments ra Payment Service DB.
> Review lần 5: 25/04/2026 — Giải quyết issue 2.2: Thiếu validation khi thêm sản phẩm vào giỏ hàng.
> Review lần 6: 25/04/2026 — Giải quyết issue 2.7: Thiếu xử lý khi khách hủy đơn chủ động.
> Review lần 7: 25/04/2026 — Giải quyết issue 2.8: SQS message schema chưa được define.

---

## Mục lục

1. [Mâu thuẫn giữa các tài liệu (Cần đồng bộ ngay)](#1-mâu-thuẫn-giữa-các-tài-liệu)
2. [Vấn đề thiết kế mới phát hiện](#2-vấn-đề-thiết-kế-mới-phát-hiện)
3. [Vấn đề thiết kế cũ còn tồn đọng](#3-vấn-đề-thiết-kế-cũ-còn-tồn-đọng)
4. [Thiếu sót trong tài liệu](#4-thiếu-sót-trong-tài-liệu)
5. [Gợi ý cải thiện (Nice to have)](#5-gợi-ý-cải-thiện)
6. [Các vấn đề đã được sửa](#6-các-vấn-đề-đã-được-sửa)

---

## 1. Mâu thuẫn giữa các tài liệu

> BA_Document.md đã được cập nhật nhiều giải pháp mới, nhưng Technical_Architect.md chưa đồng bộ. Developer đọc 2 file sẽ thấy thông tin khác nhau.

### 1.1 ~~Cache strategy: BA ghi cache-aside, Technical Architect ghi write-through~~ ✅ ĐÃ SỬA

> Technical_Architect.md mục 3 + mục 5.5 đã đồng bộ sang cache-aside + SQS invalidation.

---

### 1.2 ~~Luồng đặt hàng online: BA có Saga, Technical Architect vẫn luồng cũ~~ ✅ ĐÃ SỬA

> Technical_Architect.md mục 5.1 đã đồng bộ sang Saga pattern: CREATED → trừ kho → PENDING/FAILED.

---

### 1.3 ~~Luồng nhập hàng: BA có Saga, Technical Architect vẫn ghi "atomic"~~ ✅ ĐÃ SỬA

> Technical_Architect.md mục 5.6 đã đồng bộ sang Saga pattern với inventory_transactions làm source of truth.

---

### 1.4 ~~SQS Queues: BA bỏ analytics-queue, Technical Architect vẫn liệt kê~~ ✅ ĐÃ SỬA

> Technical_Architect.md mục 1 (diagram), mục 4 (bảng), mục 5.3 (luồng POS) đã xóa analytics-queue. POS publish qua order-status-queue.

---

### 1.5 ~~BA mục 4 luồng đặt hàng chưa cập nhật status CREATED~~ ✅ ĐÃ SỬA

> BA_Document.md mục 4 đã cập nhật luồng đặt hàng: CREATED → trừ kho → PENDING/FAILED → CONFIRMED → PREPARING → SHIPPING → DELIVERED.

---

### 1.6 ~~BA mục 3 vẫn ghi nhập hàng "atomic"~~ ✅ ĐÃ SỬA

> BA_Document.md mục 3 đã sửa: bỏ "(atomic)", thêm reference đến Saga pattern ở mục 10.

---

## 2. Vấn đề thiết kế mới phát hiện

### 2.1 ~~Giỏ hàng checkout nhiều variant nhưng chỉ chọn 1 cửa hàng~~ ✅ ĐÃ SỬA

> BA_Document.md mục 3 đã bổ sung business rule **all-or-nothing**: chỉ cửa hàng có đủ tất cả variant trong giỏ mới được chọn. Không hỗ trợ partial order.

---

### 2.2 ~~Thiếu validation khi thêm sản phẩm vào giỏ hàng~~ ✅ ĐÃ SỬA

> BA_Document.md mục 3 đã bổ sung đầy đủ validation rules:
> - **Product Status Validation**: ACTIVE (cho phép), SUSPENDED (warning), OUT_OF_STOCK (chặn)
> - **Inventory Check Strategy**: Không check khi thêm, check khi hiển thị giỏ và checkout
> - **Xử lý khi variant bị thay đổi/xóa**: Soft delete (disable), Hard delete (xóa), Price change (warning)
> - **Quantity Limits**: Max per variant (10), Max total items (50), Max per product (20), Min quantity (1)

---

### 2.3 ~~Hoàn hàng thiếu business rules quan trọng~~ ✅ ĐÃ SỬA

> BA_Document.md mục 4 đã bổ sung đầy đủ:
> - Thời hạn hoàn: 2 ngày sau DELIVERED
> - Chỉ hoàn toàn bộ, không partial return
> - Chỉ đơn ONLINE, không áp dụng POS
> - Hoàn tiền thủ công bởi nhân viên
> - Yêu cầu hoàn không tự động hủy
> - Khách cần cung cấp lý do hoàn hàng
>
> Database_Schema.md đã thêm `return_reason`, `return_requested_at` vào orders table + return validation rules.

---

### 2.4 Thiếu `level` hoặc `depth` column trong bảng categories

**File:** Database_Schema.md (Product Service — categories)

Bảng `categories` có `path` (materialized path) nhưng thiếu `level`/`depth`. Khi cần:
- Query tất cả danh mục cấp 1 (root) → phải dùng `WHERE parent_id IS NULL` (OK)
- Query tất cả danh mục cấp 2 → phải parse path hoặc join, không có cách query trực tiếp
- Validate sản phẩm chỉ gắn vào danh mục cấp lá → cần biết danh mục nào là lá

**Gợi ý:** Thêm column `level INT NOT NULL` (0 = root, 1 = cấp 2, ...) hoặc `is_leaf BOOLEAN`.

**Hành động:**
- [ ] Cân nhắc thêm `level` hoặc `is_leaf` vào bảng categories

---

### 2.5 Materialized path dùng UUID sẽ rất dài

**File:** Database_Schema.md (Product Service — categories)

Path ví dụ: `/1/2/3` nhưng thực tế ID là UUID (36 ký tự). Với 5 cấp:
```
/550e8400-e29b-41d4-a716-446655440000/6ba7b810-9dad-11d1-80b4-00c04fd430c8/...
```
→ Dễ vượt `VARCHAR(500)`, và `LIKE` query trên chuỗi dài sẽ chậm.

**Gợi ý:**
- Dùng integer auto-increment ID riêng cho categories (không dùng UUID cho path)
- Hoặc dùng short code thay vì UUID trong path

**Hành động:**
- [ ] Xem lại chiến lược materialized path với UUID

---

### 2.6 ScyllaDB stores table dùng full scan để lấy tất cả cửa hàng

**File:** Database_Schema.md (Store & Inventory Service — stores)

Ghi chú: *"Lấy tất cả cửa hàng: full scan (số lượng ít, chấp nhận được)"*

Đúng là chấp nhận được khi ít cửa hàng, nhưng:
- Full scan trong ScyllaDB là **anti-pattern** — nó query tất cả partition trên tất cả node
- Nếu sau này mở rộng ra nhiều thành phố, số cửa hàng tăng → vẫn full scan?
- Không có cách filter theo `city` hay `is_active` hiệu quả

**Gợi ý:** Thêm bảng `stores_by_city` nếu cần query theo thành phố, hoặc cache danh sách stores trong Redis (vì data ít thay đổi).

**Hành động:**
- [ ] Cân nhắc cache stores trong Redis hoặc thêm bảng query-optimized

---

### 2.7 ~~Thiếu xử lý khi khách hủy đơn chủ động~~ ✅ ĐÃ SỬA

> BA_Document.md mục 4 đã bổ sung đầy đủ:
> - Ai có thể hủy đơn: Khách hàng, Nhân viên, Cửa hàng trưởng, Admin, System
> - Khi nào có thể hủy: Khách hàng (PENDING/CONFIRMED), Staff/Manager (PENDING/CONFIRMED/PREPARING), Admin (bất kỳ)
> - Lý do hủy đơn: Định nghĩa cho từng role
> - Luồng xử lý khi hủy đơn: CANCELLED → hoàn trả inventory → gửi email → hoàn tiền
>
> Database_Schema.md đã thêm `cancelled_by`, `cancel_reason`, `cancelled_at` vào orders table
> Technical_Architect.md mục 5.2.1 đã thêm luồng hủy đơn chủ động
> Database_Schema.md inventory_transactions đã thêm type: CANCEL

---

### 2.8 ~~SQS message schema chưa được define~~ ✅ ĐÃ SỬA

> Technical_Architect.md mục 4.1 đã bổ sung đầy đủ:
> - Standard message fields (version, message_id, correlation_id, timestamp, event_type, data)
> - Message schema cho 4 queues: order-confirmed-queue, payment-queue, order-status-queue, product-updated-queue
> - Special cases: auto-cancel, proactive cancellation, return completed
> - Message processing guidelines: idempotency, error handling, versioning, correlation ID, timestamp

---

### 2.9 ~~`order_events` trong DynamoDB có thể bị duplicate~~ ✅ ĐÃ SỬA

> Database_Schema.md Analytics section đã cập nhật:
> - Chỉ aggregate khi status = `DELIVERED`, reverse aggregate khi `RETURN_COMPLETED`
> - SK thêm `#{status}` để phân biệt event cùng order
> - Dùng DynamoDB conditional write (attribute_not_exists) để dedup

---

### 2.10 Thiếu pagination cho inventory_by_store

**File:** Database_Schema.md (Store & Inventory Service)

Query `WHERE store_id = ?` trên `inventory_by_store` trả về **tất cả variant** tại 1 cửa hàng. Nếu cửa hàng có hàng nghìn variant → response rất lớn.

ScyllaDB hỗ trợ paging qua driver, nhưng tài liệu chưa đề cập pagination strategy cho các query trả về nhiều row.

**Hành động:**
- [ ] Document pagination strategy cho ScyllaDB queries

---

## 3. Vấn đề thiết kế cũ còn tồn đọng

### 3.1 ~~Bảng `payments` vi phạm database-per-service~~ ✅ ĐÃ SỬA

> Database_Schema.md đã cập nhật:
> - Tách payments ra Payment Service DB (MySQL)
> - Thêm bảng `payments` và `payment_transactions` vào Payment Service
> - Cập nhật Order Service DB: thêm `payment_id` và `payment_status` vào bảng `orders`
> - Technical_Architect.md mục 2 đã cập nhật: Payment Service database = MySQL
> - BA_Document.md đã xóa note technical debt

---

### 3.2 Retry policy chưa chi tiết

**Trạng thái:** BA mục 10 có DLQ maxReceiveCount: 3, nhưng chưa có:
- Exponential backoff config (initial delay, max delay)
- Poison message handling
- DLQ monitoring/alarm

**Hành động:**
- [ ] Bổ sung retry policy chi tiết

---

## 4. Thiếu sót trong tài liệu

### 4.1 Thiếu observability strategy

Không có tài liệu nào đề cập distributed tracing, centralized logging, metrics, health check.

**Hành động:**
- [ ] Thêm mục "Observability" vào Technical_Architect.md (OpenTelemetry + X-Ray, CloudWatch Logs, health check endpoints, k8s probes)

---

### 4.2 Thiếu security concerns ngoài JWT

Chưa document: rate limiting, CORS, input validation, secrets management, k8s network policy, data encryption.

**Hành động:**
- [ ] Thêm mục "Security" vào Technical_Architect.md

---

### 4.3 Thiếu data migration và seeding strategy

Chưa document: EF Core Migrations, CQL scripts, seed data, rollback strategy, migration order khi deploy nhiều service.

**Hành động:**
- [ ] Thêm mục "Data Migration" vào Technical_Architect.md

---

### 4.4 Thiếu Docker Compose / local development setup

**File:** BA_Document.md (Tech stack ghi Docker + Docker Compose)

Tech stack liệt kê Docker Compose nhưng không có tài liệu nào mô tả:
- Làm sao chạy toàn bộ hệ thống locally?
- Docker Compose file cần những service gì? (MySQL, ScyllaDB, Redis, LocalStack cho SQS/SES/S3/Cognito/DynamoDB)
- Environment variables cho local vs production

**Hành động:**
- [ ] Tạo docker-compose.yml hoặc document local development setup

---

### 4.5 Thiếu testing strategy

Không có tài liệu nào đề cập:
- Unit test, integration test, e2e test cho từng service
- Test strategy cho async flows (SQS message processing)
- Test strategy cho cross-service flows (đặt hàng = Order + Inventory + Payment)
- Contract testing giữa các service

**Hành động:**
- [ ] Document testing strategy

---

## 5. Gợi ý cải thiện (Nice to have)

### 5.1 Thêm monthly aggregation cho Analytics

Query 30 ngày rồi sum ở application layer sẽ chậm khi data lớn. Cân nhắc thêm `monthly_product_stats` / `monthly_category_stats`.

---

### 5.2 Cân nhắc Redis cho giỏ hàng thay vì MySQL

Giỏ hàng là data tạm thời, read/write thường xuyên. Redis sẽ nhanh hơn. Trade-off: mất data khi restart.

---

### 5.3 Bổ sung Notification Service database

Không có DB → không lưu lịch sử email, không retry được email fail, không có notification preferences.

---

### 5.4 Cân nhắc tách/gộp Identity Service

Nếu Identity Service chỉ wrap Cognito → có thể gộp vào Envoy Gateway + User Service.

---

### 5.5 Thêm `order_number` human-readable cho orders

Orders hiện chỉ có UUID làm ID. Khách hàng và nhân viên cần mã đơn hàng dễ đọc (ví dụ: `ORD-20250423-0001`). Cân nhắc thêm column `order_number VARCHAR(20) UNIQUE`.

---

### 5.6 Cân nhắc search engine cho sản phẩm

BA ghi *"Hỗ trợ tìm kiếm và lọc sản phẩm theo danh mục, giá"* nhưng chỉ dùng MySQL. Khi sản phẩm nhiều, full-text search trên MySQL sẽ chậm và thiếu tính năng (fuzzy search, tiếng Việt không dấu). Cân nhắc Elasticsearch/OpenSearch cho product search.

---

## 6. Các vấn đề đã được sửa (từ review lần 1-2)

| # | Vấn đề | Sửa ở đâu |
|---|---|---|
| ✅ | Cache strategy mâu thuẫn | BA mục 9 → cache-aside |
| ✅ | ScyllaDB không có transaction | BA mục 10 → Saga pattern |
| ✅ | Race condition đặt hàng | BA mục 10 → Saga (CREATED/FAILED), DB Schema cập nhật |
| ✅ | Concurrent inventory overselling | BA mục 10 → conditional update CQL |
| ✅ | Auto-cancel thiếu idempotency | BA mục 10 → optimistic locking + outbox |
| ✅ | analytics-queue gộp | DB Schema Analytics → dùng order-status-queue |
| ✅ | Order status state machine | BA mục 10 + DB Schema → có diagram |
| ✅ | price_history thiếu soft delete | DB Schema → ghi rõ append-only exception |
| ✅ | Thiếu sync service calls | BA mục 10 → có bảng sync calls |
| ✅ | Thiếu DLQ | BA mục 10 → maxReceiveCount: 3 |
| ✅ | Thiếu API versioning | BA mục 10 → URL versioning /api/v1/ |
| ✅ | POS offline | BA mục 10 → yêu cầu internet, offline out of scope |
| ✅ | Bảng payments vi phạm database-per-service | Database_Schema.md → tách ra Payment Service DB, Technical_Architect.md → Payment Service database = MySQL |
| ✅ | Thiếu validation khi thêm sản phẩm vào giỏ hàng | BA_Document.md mục 3 → bổ sung 4 validation rules (product status, inventory check, variant deletion, quantity limits) |
| ✅ | Thiếu xử lý khi khách hủy đơn chủ động | BA_Document.md mục 4 → bổ sung business rules hủy đơn, Database_Schema.md → thêm cancelled_by, cancel_reason, cancelled_at, Technical_Architect.md mục 5.2.1 → luồng hủy đơn chủ động |
| ✅ | SQS message schema chưa được define | Technical_Architect.md mục 4.1 → bổ sung message schema cho 4 queues (order-confirmed-queue, payment-queue, order-status-queue, product-updated-queue) + message processing guidelines |

---

## Tổng kết

| Mức độ | Tổng | Hành động |
|---|---|---|
| **Mâu thuẫn tài liệu** | 6 | ✅ Đã sửa 6/6 |
| **Thiết kế mới** | 10 | ✅ Đã sửa 6/10, còn 4 issue |
| **Thiết kế cũ tồn đọng** | 2 | ✅ Đã sửa 1/2 |
| **Thiếu sót tài liệu** | 5 | Bổ sung vào Technical_Architect.md |
| **Gợi ý cải thiện** | 6 | Nice to have, làm sau |

### Ưu tiên cao nhất

1. ~~**Đồng bộ Technical_Architect.md** (mục 1.1 → 1.4)~~ ✅ ĐÃ SỬA
2. ~~**Sửa mâu thuẫn nội bộ BA** (mục 1.5, 1.6)~~ ✅ ĐÃ SỬA
3. ~~**Define analytics aggregation trigger** (mục 2.9)~~ ✅ ĐÃ SỬA
4. ~~**Define checkout khi thiếu hàng** (mục 2.1)~~ ✅ ĐÃ SỬA
5. ~~**Define hoàn hàng business rules** (mục 2.3)~~ ✅ ĐÃ SỬA
6. ~~**Tách payments ra Payment Service DB** (mục 3.1)~~ ✅ ĐÃ SỬA
7. ~~**Define validation rules khi thêm sản phẩm vào giỏ hàng** (mục 2.2)~~ ✅ ĐÃ SỬA
8. ~~**Define xử lý khi khách hủy đơn chủ động** (mục 2.7)~~ ✅ ĐÃ SỬA
9. ~~**Define SQS message schema** (mục 2.8)~~ ✅ ĐÃ SỬA

---

*Review lần 1: 23/04/2026*
*Cross-check lần 2: 23/04/2026*
*Deep review lần 3: 23/04/2026*
*Review lần 4: 25/04/2026*
*Review lần 5: 25/04/2026*