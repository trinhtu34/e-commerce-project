# Issues Checklist — Hệ thống E-Commerce Microservice

> Checklist theo dõi tiến độ giải quyết các issues từ Review_Issues.md
>
> Cập nhật lần cuối: 25/04/2026

---

## Tổng quan

| Mức độ | Tổng | Đã giải quyết | Cần giải quyết | Nice to have |
|---|---|---|---|---|
| **Mâu thuẫn tài liệu** | 6 | 6 | 0 | 0 |
| **Thiết kế mới** | 10 | 5 | 5 | 0 |
| **Thiết kế cũ tồn đọng** | 2 | 1 | 1 | 0 |
| **Thiếu sót tài liệu** | 5 | 0 | 5 | 0 |
| **Gợi ý cải thiện** | 6 | 0 | 0 | 6 |
| **TỔNG** | 29 | 12 | 11 | 6 |

---

## 🔴 Ưu tiên cao (Cần giải quyết ngay)

### 3.1 ~~Bảng `payments` vi phạm database-per-service~~ ✅ ĐÃ GIẢI QUYẾT

**File:** Technical_Architect.md mục 2

**Mô tả:**
- Payment Service database = `-` mà không có note giải thích
- Bảng `payments` nằm trong Order Service DB (vi phạm database-per-service)
- BA_Document đã ghi nhận là "Technical Debt có kế hoạch"

**Giải pháp đã thực hiện:**
- ✅ Tách payments ra Payment Service DB (MySQL)
- ✅ Thêm bảng `payments` và `payment_transactions` vào Payment Service
- ✅ Cập nhật Order Service DB: thêm `payment_id` và `payment_status` vào bảng `orders`
- ✅ Cập nhật Technical_Architect.md: Payment Service database = MySQL
- ✅ Xóa note technical debt khỏi BA_Document.md

**Hành động:**
- [x] Thêm note vào Technical_Architect.md mục 2 giải thích tại sao Payment Service không có DB
- [x] Hoặc: Tách payments ra Payment Service DB (giải pháp dài hạn) ✅ ĐÃ CHỌN

**Ưu tiên:** 🔴 Cao
**Phức tạp:** Trung bình
**Thời gian ước tính:** 2-4 giờ (thêm note) hoặc 1-2 ngày (tách DB) ✅ HOÀN THÀNH

---

### 2.2 ~~Thiếu validation khi thêm sản phẩm vào giỏ hàng~~ ✅ ĐÃ GIẢI QUYẾT

**File:** BA_Document.md mục 3

**Mô tả:**
Khi khách thêm variant vào giỏ, tài liệu chưa define:
- Có kiểm tra product status không? (SUSPENDED, OUT_OF_STOCK → có cho thêm vào giỏ?)
- Có kiểm tra variant còn tồn kho ở bất kỳ cửa hàng nào không?
- Nếu variant bị xóa (soft delete) sau khi đã nằm trong giỏ → xử lý thế nào?
- Giới hạn quantity tối đa mỗi variant trong giỏ?

**Giải pháp đã thực hiện:**
- ✅ Define validation rules khi add to cart
- ✅ Define xử lý khi variant trong giỏ bị thay đổi/xóa

**Validation Rules đã thêm vào BA_Document.md:**
1. **Product Status Validation**: ACTIVE (cho phép), SUSPENDED (warning), OUT_OF_STOCK (chặn)
2. **Inventory Check Strategy**: Không check khi thêm, check khi hiển thị giỏ và checkout
3. **Xử lý khi variant bị thay đổi/xóa**: Soft delete (disable), Hard delete (xóa), Price change (warning)
4. **Quantity Limits**: Max per variant (10), Max total items (50), Max per product (20), Min quantity (1)

**Hành động:**
- [x] Define validation rules khi add to cart
- [x] Define xử lý khi variant trong giỏ bị thay đổi/xóa

**Ưu tiên:** 🔴 Cao
**Phức tạp:** Trung bình
**Thời gian ước tính:** 2-3 giờ ✅ HOÀN THÀNH

---

### 2.7 ~~Thiếu xử lý khi khách hủy đơn chủ động~~ ✅ ĐÃ GIẢI QUYẾT

**File:** BA_Document.md mục 4, Database_Schema.md (orders state machine)

**Mô tả:**
State machine cho phép `PENDING → CANCELLED` và `CONFIRMED → CANCELLED`, nhưng tài liệu chỉ đề cập **auto-cancel** (quá hạn). Chưa define:
- Khách có thể **chủ động hủy** đơn không? Ở trạng thái nào?
- Cửa hàng có thể hủy đơn không? (hết hàng thực tế, sản phẩm lỗi...)
- Khi khách/cửa hàng hủy → luồng hoàn trả inventory giống auto-cancel?
- Cần lưu `cancelled_by` (user/staff/system) và `cancel_reason`?

**Giải pháp đã thực hiện:**
- ✅ Define rõ ai được hủy đơn, ở trạng thái nào
- ✅ Thêm `cancelled_by`, `cancel_reason`, `cancelled_at` vào orders table
- ✅ Cập nhật BA_Document.md mục 4 với business rules hủy đơn
- ✅ Cập nhật Technical_Architect.md mục 5.2.1 với luồng hủy đơn chủ động
- ✅ Cập nhật Database_Schema.md inventory_transactions với type: CANCEL

**Business Rules đã thêm vào BA_Document.md:**
- **Ai có thể hủy:** Khách hàng, Nhân viên, Cửa hàng trưởng, Admin, System
- **Khi nào có thể hủy:** Khách hàng (PENDING/CONFIRMED), Staff/Manager (PENDING/CONFIRMED/PREPARING), Admin (bất kỳ)
- **Lý do hủy:** Định nghĩa cho từng role
- **Luồng xử lý:** CANCELLED → hoàn trả inventory → gửi email → hoàn tiền

**Hành động:**
- [x] Define rõ ai được hủy đơn, ở trạng thái nào
- [x] Thêm `cancelled_by` và `cancel_reason` vào orders table

**Ưu tiên:** 🔴 Cao
**Phức tạp:** Trung bình
**Thời gian ước tính:** 2-3 giờ ✅ HOÀN THÀNH

---

### 2.8 SQS message schema chưa được define

**File:** Technical_Architect.md mục 4

**Mô tả:**
Có 4 queue nhưng chưa define message format cho bất kỳ queue nào. Developer implement publisher và consumer cần biết:
- Message body chứa những field gì?
- Format: JSON? Có version field không?
- Có correlation ID để trace không?

**Hành động:**
- [ ] Define message schema cho `order-confirmed-queue`
- [ ] Define message schema cho `payment-queue`
- [ ] Define message schema cho `order-status-queue`
- [ ] Define message schema cho `product-updated-queue`

**Ưu tiên:** 🔴 Cao
**Phức tạp:** Trung bình
**Thời gian ước tính:** 3-4 giờ

---

## 🟡 Ưu tiên trung bình (Cần giải quyết)

### 2.4 Thiếu `level` hoặc `depth` column trong bảng categories

**File:** Database_Schema.md (Product Service — categories)

**Mô tả:**
Bảng `categories` có `path` (materialized path) nhưng thiếu `level`/`depth`. Khi cần:
- Query tất cả danh mục cấp 2 → phải parse path hoặc join, không có cách query trực tiếp
- Validate sản phẩm chỉ gắn vào danh mục cấp lá → cần biết danh mục nào là lá

**Hành động:**
- [ ] Cân nhắc thêm `level` hoặc `is_leaf` vào bảng categories

**Ưu tiên:** 🟡 Trung bình
**Phức tạp:** Thấp
**Thời gian ước tính:** 1-2 giờ

---

### 2.5 Materialized path dùng UUID sẽ rất dài

**File:** Database_Schema.md (Product Service — categories)

**Mô tả:**
Path ví dụ: `/1/2/3` nhưng thực tế ID là UUID (36 ký tự). Với 5 cấp:
```
/550e8400-e29b-41d4-a716-446655440000/6ba7b810-9dad-11d1-80b4-00c04fd430c8/...
```
→ Dễ vượt `VARCHAR(500)`, và `LIKE` query trên chuỗi dài sẽ chậm.

**Hành động:**
- [ ] Xem lại chiến lược materialized path với UUID
- [ ] Cân nhắc dùng integer auto-increment ID riêng cho categories
- [ ] Hoặc dùng short code thay vì UUID trong path

**Ưu tiên:** 🟡 Trung bình
**Phức tạp:** Trung bình
**Thời gian ước tính:** 2-3 giờ

---

### 2.6 ScyllaDB stores table dùng full scan

**File:** Database_Schema.md (Store & Inventory Service — stores)

**Mô tả:**
Ghi chú: *"Lấy tất cả cửa hàng: full scan (số lượng ít, chấp nhận được)"*

Đúng là chấp nhận được khi ít cửa hàng, nhưng:
- Full scan trong ScyllaDB là **anti-pattern** — nó query tất cả partition trên tất cả node
- Nếu sau này mở rộng ra nhiều thành phố, số lượng cửa hàng tăng → vẫn full scan?
- Không có cách filter theo `city` hay `is_active` hiệu quả

**Hành động:**
- [ ] Cân nhắc cache stores trong Redis
- [ ] Hoặc thêm bảng `stores_by_city` nếu cần query theo thành phố

**Ưu tiên:** 🟡 Trung bình
**Phức tạp:** Thấp
**Thời gian ước tính:** 1-2 giờ

---

### 2.10 Thiếu pagination cho inventory_by_store

**File:** Database_Schema.md (Store & Inventory Service)

**Mô tả:**
Query `WHERE store_id = ?` trên `inventory_by_store` trả về **tất cả variant** tại 1 cửa hàng. Nếu cửa hàng có hàng nghìn variant → response rất lớn.

ScyllaDB hỗ trợ paging qua driver, nhưng tài liệu chưa đề cập pagination strategy cho các query trả về nhiều row.

**Hành động:**
- [ ] Document pagination strategy cho ScyllaDB queries

**Ưu tiên:** 🟡 Trung bình
**Phức tạp:** Thấp
**Thời gian ước tính:** 1-2 giờ

---

### 3.2 Retry policy chưa chi tiết

**File:** BA_Document.md mục 10

**Mô tả:**
BA mục 10 có DLQ maxReceiveCount: 3, nhưng chưa có:
- Exponential backoff config (initial delay, max delay)
- Poison message handling
- DLQ monitoring/alarm

**Hành động:**
- [ ] Bổ sung retry policy chi tiết
- [ ] Define exponential backoff config
- [ ] Define poison message handling
- [ ] Define DLQ monitoring/alarm

**Ưu tiên:** 🟡 Trung bình
**Phức tạp:** Thấp
**Thời gian ước tính:** 1-2 giờ

---

## 🟢 Ưu tiên thấp (Thiếu sót tài liệu)

### 4.1 Thiếu observability strategy

**Mô tả:**
Không có tài liệu nào đề cập distributed tracing, centralized logging, metrics, health check.

**Hành động:**
- [ ] Thêm mục "Observability" vào Technical_Architect.md
- [ ] Document OpenTelemetry + X-Ray
- [ ] Document CloudWatch Logs
- [ ] Document health check endpoints
- [ ] Document k8s probes

**Ưu tiên:** 🟢 Thấp
**Phức tạp:** Trung bình
**Thời gian ước tính:** 3-4 giờ

---

### 4.2 Thiếu security concerns ngoài JWT

**Mô tả:**
Chưa document: rate limiting, CORS, input validation, secrets management, k8s network policy, data encryption.

**Hành động:**
- [ ] Thêm mục "Security" vào Technical_Architect.md
- [ ] Document rate limiting
- [ ] Document CORS
- [ ] Document input validation
- [ ] Document secrets management
- [ ] Document k8s network policy
- [ ] Document data encryption

**Ưu tiên:** 🟢 Thấp
**Phức tạp:** Trung bình
**Thời gian ước tính:** 3-4 giờ

---

### 4.3 Thiếu data migration và seeding strategy

**Mô tả:**
Chưa document: EF Core Migrations, CQL scripts, seed data, rollback strategy, migration order khi deploy nhiều service.

**Hành động:**
- [ ] Thêm mục "Data Migration" vào Technical_Architect.md
- [ ] Document EF Core Migrations
- [ ] Document CQL scripts
- [ ] Document seed data
- [ ] Document rollback strategy
- [ ] Document migration order khi deploy nhiều service

**Ưu tiên:** 🟢 Thấp
**Phức tạp:** Trung bình
**Thời gian ước tính:** 2-3 giờ

---

### 4.4 Thiếu Docker Compose / local development setup

**File:** BA_Document.md (Tech stack ghi Docker + Docker Compose)

**Mô tả:**
Tech stack liệt kê Docker Compose nhưng không có tài liệu nào mô tả:
- Làm sao chạy toàn bộ hệ thống locally?
- Docker Compose file cần những service gì?
- Environment variables cho local vs production

**Hành động:**
- [ ] Tạo docker-compose.yml
- [ ] Document local development setup
- [ ] Define environment variables cho local vs production

**Ưu tiên:** 🟢 Thấp
**Phức tạp:** Trung bình
**Thời gian ước tính:** 4-6 giờ

---

### 4.5 Thiếu testing strategy

**Mô tả:**
Không có tài liệu nào đề cập:
- Unit test, integration test, e2e test cho từng service
- Test strategy cho async flows (SQS message processing)
- Test strategy cho cross-service flows (đặt hàng = Order + Inventory + Payment)
- Contract testing giữa các service

**Hành động:**
- [ ] Document testing strategy
- [ ] Define unit test strategy
- [ ] Define integration test strategy
- [ ] Define e2e test strategy
- [ ] Define test strategy cho async flows
- [ ] Define test strategy cho cross-service flows
- [ ] Define contract testing strategy

**Ưu tiên:** 🟢 Thấp
**Phức tạp:** Trung bình
**Thời gian ước tính:** 3-4 giờ

---

## 💡 Nice to have (Có thể làm sau)

### 5.1 Thêm monthly aggregation cho Analytics

**Mô tả:**
Query 30 ngày rồi sum ở application layer sẽ chậm khi data lớn. Cân nhắc thêm `monthly_product_stats` / `monthly_category_stats`.

**Hành động:**
- [ ] Cân nhắc thêm monthly aggregation tables

**Ưu tiên:** 💡 Nice to have
**Phức tạp:** Trung bình
**Thời gian ước tính:** 2-3 giờ

---

### 5.2 Cân nhắc Redis cho giỏ hàng thay vì MySQL

**Mô tả:**
Giỏ hàng là data tạm thời, read/write thường xuyên. Redis sẽ nhanh hơn. Trade-off: mất data khi restart.

**Hành động:**
- [ ] Cân nhắc migrate giỏ hàng sang Redis

**Ưu tiên:** 💡 Nice to have
**Phức tạp:** Trung bình
**Thời gian ước tính:** 4-6 giờ

---

### 5.3 Bổ sung Notification Service database

**Mô tả:**
Không có DB → không lưu lịch sử email, không retry được email fail, không có notification preferences.

**Hành động:**
- [ ] Cân nhắc thêm DB cho Notification Service

**Ưu tiên:** 💡 Nice to have
**Phức tạp:** Trung bình
**Thời gian ước tính:** 3-4 giờ

---

### 5.4 Cân nhắc tách/gộp Identity Service

**Mô tả:**
Nếu Identity Service chỉ wrap Cognito → có thể gộp vào Envoy Gateway + User Service.

**Hành động:**
- [ ] Cân nhắc tách/gộp Identity Service

**Ưu tiên:** 💡 Nice to have
**Phức tạp:** Cao
**Thời gian ước tính:** 1-2 ngày

---

### 5.5 Thêm `order_number` human-readable cho orders

**Mô tả:**
Orders hiện chỉ có UUID làm ID. Khách hàng và nhân viên cần mã đơn hàng dễ đọc (ví dụ: `ORD-20250423-0001`).

**Hành động:**
- [ ] Cân nhắc thêm column `order_number VARCHAR(20) UNIQUE`

**Ưu tiên:** 💡 Nice to have
**Phức tạp:** Thấp
**Thời gian ước tính:** 1-2 giờ

---

### 5.6 Cân nhắc search engine cho sản phẩm

**Mô tả:**
BA ghi *"Hỗ trợ tìm kiếm và lọc sản phẩm theo danh mục, giá"* nhưng chỉ dùng MySQL. Khi sản phẩm nhiều, full-text search trên MySQL sẽ chậm và thiếu tính năng (fuzzy search, tiếng Việt không dấu).

**Hành động:**
- [ ] Cân nhắc Elasticsearch/OpenSearch cho product search

**Ưu tiên:** 💡 Nice to have
**Phức tạp:** Cao
**Thời gian ước tính:** 2-3 ngày

---

## 📊 Thống kê tiến độ

### Theo mức độ ưu tiên

| Mức độ | Tổng | Đã hoàn thành | Đang làm | Chưa bắt đầu |
|---|---|---|---|---|
| 🔴 Cao | 4 | 2 | 0 | 2 |
| 🟡 Trung bình | 5 | 0 | 0 | 5 |
| 🟢 Thấp | 5 | 0 | 0 | 5 |
| 💡 Nice to have | 6 | 0 | 0 | 6 |

### Theo loại vấn đề

| Loại | Tổng | Đã hoàn thành | Đang làm | Chưa bắt đầu |
|---|---|---|---|---|
| Thiết kế mới | 7 | 1 | 0 | 6 |
| Thiết kế cũ tồn đọng | 2 | 1 | 0 | 1 |
| Thiếu sót tài liệu | 5 | 0 | 0 | 5 |
| Gợi ý cải thiện | 6 | 0 | 0 | 6 |

---

## 🎯 Kế hoạch đề xuất

### Giai đoạn 1: Ưu tiên cao (Tuần 1)
1. ~~3.1 Bảng `payments` vi phạm database-per-service~~ ✅ Đã hoàn thành
2. ~~2.2 Thiếu validation khi thêm sản phẩm vào giỏ hàng~~ ✅ Đã hoàn thành
3. 2.7 Thiếu xử lý khi khách hủy đơn chủ động
4. 2.8 SQS message schema chưa được define

### Giai đoạn 2: Ưu tiên trung bình (Tuần 2)
1. 2.4 Thiếu `level` hoặc `depth` column trong bảng categories
2. 2.5 Materialized path dùng UUID sẽ rất dài
3. 2.6 ScyllaDB stores table dùng full scan
4. 2.10 Thiếu pagination cho inventory_by_store
5. 3.2 Retry policy chưa chi tiết

### Giai đoạn 3: Thiếu sót tài liệu (Tuần 3)
1. 4.1 Thiếu observability strategy
2. 4.2 Thiếu security concerns ngoài JWT
3. 4.3 Thiếu data migration và seeding strategy
4. 4.4 Thiếu Docker Compose / local development setup
5. 4.5 Thiếu testing strategy

### Giai đoạn 4: Nice to have (Khi có thời gian)
1. 5.1 Thêm monthly aggregation cho Analytics
2. 5.2 Cân nhắc Redis cho giỏ hàng thay vì MySQL
3. 5.3 Bổ sung Notification Service database
4. 5.4 Cân nhắc tách/gộp Identity Service
5. 5.5 Thêm `order_number` human-readable cho orders
6. 5.6 Cân nhắc search engine cho sản phẩm

---

*File được tạo từ Review_Issues.md ngày 25/04/2026*