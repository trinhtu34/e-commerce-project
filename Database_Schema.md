# Database Schema Design

## User Service (MySQL)

### 1. users

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | Cognito `sub` UUID |
| full_name | VARCHAR(100) | NOT NULL | Họ tên |
| phone | VARCHAR(20) | UNIQUE, NOT NULL | Số điện thoại |
| email | VARCHAR(255) | UNIQUE, NOT NULL | Email |
| gender | ENUM('MALE','FEMALE','OTHER') | NULL | Giới tính |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

### 2. user_addresses

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| user_id | CHAR(36) | FK → users.id, NOT NULL | |
| label | VARCHAR(50) | NOT NULL | Nhà, Công ty, ... |
| street | VARCHAR(255) | NOT NULL | Địa chỉ đường |
| district | VARCHAR(100) | NOT NULL | Quận/Huyện |
| city | VARCHAR(100) | NOT NULL | Thành phố |
| latitude | DECIMAL(10,7) | NOT NULL | Vĩ độ |
| longitude | DECIMAL(10,7) | NOT NULL | Kinh độ |
| is_default | BOOLEAN | NOT NULL, DEFAULT FALSE | Địa chỉ mặc định |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Business rule:** Mỗi user chỉ có 1 address với `is_default = true`

### 3. store_staff

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| user_id | CHAR(36) | FK → users.id, NOT NULL | |
| store_id | CHAR(36) | NOT NULL | ID cửa hàng (từ Store & Inventory Service) |
| role | ENUM('STAFF','STORE_MANAGER') | NOT NULL | Vai trò tại cửa hàng |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE | Đang làm việc hay đã nghỉ |
| started_at | DATE | NOT NULL | Ngày bắt đầu làm việc |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `UNIQUE(user_id, store_id)` — 1 nhân viên chỉ gắn 1 lần với 1 cửa hàng

### ERD

```
users (1) ──── (N) user_addresses
users (1) ──── (N) store_staff
```

---

## Product Service (MySQL)

### 1. categories

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| name | VARCHAR(100) | NOT NULL | Tên danh mục |
| parent_id | CHAR(36) | FK → categories.id, NULL | NULL = cấp gốc |
| path | VARCHAR(500) | NOT NULL | Materialized path, ví dụ: `/1/2/3` |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `INDEX(path)` — tối ưu query `WHERE path LIKE '/1/2/%'`
- `INDEX(parent_id)`

### 2. products

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| category_id | CHAR(36) | FK → categories.id, NOT NULL | Danh mục cấp lá |
| name | VARCHAR(255) | NOT NULL | Tên sản phẩm |
| description | TEXT | NULL | Mô tả sản phẩm |
| base_price | DECIMAL(12,2) | NOT NULL | Giá hiển thị chung (ví dụ: "từ 8.000đ") |
| status | ENUM('ACTIVE','OUT_OF_STOCK','SUSPENDED') | NOT NULL, DEFAULT 'ACTIVE' | Hệ thống tự cập nhật |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `INDEX(category_id)`
- `INDEX(status)`

### 3. product_images

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| product_id | CHAR(36) | FK → products.id, NOT NULL | |
| image_url | VARCHAR(500) | NOT NULL | URL ảnh trên S3 |
| sort_order | INT | NOT NULL, DEFAULT 0 | Thứ tự hiển thị |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

### 4. product_variants

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| product_id | CHAR(36) | FK → products.id, NOT NULL | |
| sku | VARCHAR(50) | UNIQUE, NOT NULL | Mã SKU, ví dụ: `PEPSI-330-COLA` |
| attributes | JSON | NOT NULL | Tổ hợp thuộc tính, ví dụ: `{"Dung tích": "330ml", "Hương vị": "Cola"}` |
| price | DECIMAL(12,2) | NOT NULL | Giá thực tế của variant |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `INDEX(product_id)`

### 5. price_history

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| product_id | CHAR(36) | FK → products.id, NOT NULL | |
| variant_id | CHAR(36) | FK → product_variants.id, NULL | NULL = thay đổi giá product level |
| old_price | DECIMAL(12,2) | NOT NULL | Giá cũ |
| new_price | DECIMAL(12,2) | NOT NULL | Giá mới |
| changed_at | DATETIME | NOT NULL, DEFAULT NOW() | Thời điểm thay đổi |
| changed_by | CHAR(36) | NOT NULL | User ID người thay đổi |

> **Lưu ý:** `price_history` là **append-only log**, không áp dụng soft delete (exception của quy tắc chung).

**Logic:**
- `variant_id = NULL` → lịch sử thay đổi `base_price` của product
- `variant_id != NULL` → lịch sử thay đổi `price` của variant

### ERD

```
categories (1) ──── (N) products
products   (1) ──── (N) product_images
products   (1) ──── (N) product_variants
products   (1) ──── (N) price_history
product_variants (1) ──── (N) price_history
```

---

## Order Service (MySQL + Redis)

### 1. carts

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| user_id | CHAR(36) | UNIQUE, NOT NULL | 1 user chỉ có 1 cart active |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

### 2. cart_items

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| cart_id | CHAR(36) | FK → carts.id, NOT NULL | |
| product_variant_id | CHAR(36) | NOT NULL | ID variant từ Product Service |
| quantity | INT | NOT NULL, DEFAULT 1 | Số lượng, cộng dồn khi thêm trùng variant |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `UNIQUE(cart_id, product_variant_id)` — đảm bảo 1 variant chỉ có 1 dòng trong cart, cộng dồn quantity

### 3. orders

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| user_id | CHAR(36) | NULL | NULL = khách vãng lai (POS) |
| staff_id | CHAR(36) | NULL | Nhân viên tạo đơn POS, NULL = đơn online |
| store_id | CHAR(36) | NOT NULL | Cửa hàng xử lý đơn |
| type | ENUM('ONLINE','POS') | NOT NULL | Loại đơn hàng |
| status | ENUM('CREATED','PENDING','CONFIRMED','PREPARING','SHIPPING','DELIVERED','FAILED','CANCELLED','RETURN_REQUESTED','RETURN_PICKING','RETURN_SHIPPING','RETURN_COMPLETED') | NOT NULL, DEFAULT 'CREATED' | |
| shipping_address | TEXT | NULL | Địa chỉ giao hàng (snapshot), NULL cho POS |
| shipping_latitude | DECIMAL(10,7) | NULL | |
| shipping_longitude | DECIMAL(10,7) | NULL | |
| total_amount | DECIMAL(12,2) | NOT NULL | Tổng tiền đơn hàng |
| note | TEXT | NULL | Ghi chú đơn hàng |
| return_reason | TEXT | NULL | Lý do hoàn hàng (chỉ khi có yêu cầu hoàn) |
| return_requested_at | DATETIME | NULL | Thời điểm khách yêu cầu hoàn hàng |
| expires_at | DATETIME | NULL | Thời hạn xác nhận (PENDING + 1 ngày), NULL cho POS |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Order Status State Machine:**
```
CREATED → PENDING → CONFIRMED → PREPARING → SHIPPING → DELIVERED
CREATED → FAILED
PENDING → CANCELLED
CONFIRMED → CANCELLED
DELIVERED → RETURN_REQUESTED → RETURN_PICKING → RETURN_SHIPPING → RETURN_COMPLETED
```

**Return validation:**
- `DELIVERED → RETURN_REQUESTED` chỉ được phép trong vòng **2 ngày** kể từ `updated_at` của DELIVERED
- Chỉ áp dụng cho đơn `type = ONLINE`
- Chỉ hỗ trợ hoàn toàn bộ đơn hàng (không partial return)

**Index:**
- `INDEX(user_id)`
- `INDEX(store_id)`
- `INDEX(status)`
- `INDEX(type)`
- `INDEX(expires_at)` — tối ưu background job auto-cancel

### 4. order_items

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| order_id | CHAR(36) | FK → orders.id, NOT NULL | |
| product_variant_id | CHAR(36) | NOT NULL | ID variant gốc (reference) |
| snapshot_product_name | VARCHAR(255) | NOT NULL | Tên sản phẩm tại thời điểm đặt |
| snapshot_sku | VARCHAR(50) | NOT NULL | SKU tại thời điểm đặt |
| snapshot_price | DECIMAL(12,2) | NOT NULL | Giá variant tại thời điểm đặt |
| snapshot_image_url | VARCHAR(500) | NULL | Ảnh sản phẩm tại thời điểm đặt |
| snapshot_attributes | JSON | NOT NULL | Thuộc tính variant, ví dụ: `{"Dung tích": "330ml"}` |
| quantity | INT | NOT NULL | Số lượng |
| subtotal | DECIMAL(12,2) | NOT NULL | snapshot_price × quantity |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `INDEX(order_id)`

### 5. payments

| Column | Type | Constraints | Mô tả |
|---|---|---|---|
| id | CHAR(36) | PK | UUID |
| order_id | CHAR(36) | FK → orders.id, NOT NULL | |
| method | ENUM('CASH','TRANSFER','ONLINE') | NOT NULL | Phương thức thanh toán |
| status | ENUM('PENDING','SUCCESS','FAILED') | NOT NULL, DEFAULT 'PENDING' | |
| amount | DECIMAL(12,2) | NOT NULL | Số tiền thanh toán |
| paid_at | DATETIME | NULL | Thời điểm thanh toán thành công |
| created_at | DATETIME | NOT NULL, DEFAULT NOW() | |
| updated_at | DATETIME | NULL | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft delete |
| deleted_at | DATETIME | NULL | |

**Index:**
- `INDEX(order_id)`
- `INDEX(status)`

### ERD

```
carts      (1) ──── (N) cart_items
orders     (1) ──── (N) order_items
orders     (1) ──── (1) payments
```

### Redis Cache Structure

```
Key:    product:{variant_id}
Value:  {
          "product_name": "Pepsi",
          "sku": "PEPSI-330-COLA",
          "price": 8000,
          "image_url": "https://s3.../pepsi.jpg",
          "attributes": {"Dung tích": "330ml", "Hương vị": "Cola"}
        }
TTL:    Không set (cache-aside, invalidate qua SQS)
```

---

## Store & Inventory Service (ScyllaDB)

> ScyllaDB thiết kế theo query-driven: mỗi bảng tối ưu cho 1 pattern query cụ thể.

### 1. stores

Lưu thông tin cửa hàng + kho tổng.

```cql
CREATE TABLE stores (
    id          UUID PRIMARY KEY,
    name        TEXT,
    type        TEXT,          -- 'STORE' hoặc 'WAREHOUSE'
    address     TEXT,
    district    TEXT,
    city        TEXT,
    latitude    DECIMAL,
    longitude   DECIMAL,
    is_active   BOOLEAN,
    created_at  TIMESTAMP,
    updated_at  TIMESTAMP
);
```

**Query hỗ trợ:**
- Lấy thông tin 1 cửa hàng: `WHERE id = ?`
- Lấy tất cả cửa hàng: full scan (số lượng ít, chấp nhận được)

### 2. inventory_by_store

Xem tồn kho tất cả variant tại 1 cửa hàng (cho cửa hàng trưởng).

```cql
CREATE TABLE inventory_by_store (
    store_id            UUID,
    product_variant_id  UUID,
    quantity            INT,
    updated_at          TIMESTAMP,
    PRIMARY KEY (store_id, product_variant_id)
);
```

**Query hỗ trợ:**
- Tồn kho tất cả variant tại 1 cửa hàng: `WHERE store_id = ?`
- Tồn kho 1 variant tại 1 cửa hàng: `WHERE store_id = ? AND product_variant_id = ?`

### 3. inventory_by_variant

Xem tồn kho của 1 variant tại tất cả cửa hàng (khi khách checkout).

```cql
CREATE TABLE inventory_by_variant (
    product_variant_id  UUID,
    store_id            UUID,
    quantity            INT,
    updated_at          TIMESTAMP,
    PRIMARY KEY (product_variant_id, store_id)
);
```

**Query hỗ trợ:**
- Tồn kho 1 variant tại tất cả cửa hàng: `WHERE product_variant_id = ?`
- Tồn kho 1 variant tại 1 cửa hàng cụ thể: `WHERE product_variant_id = ? AND store_id = ?`

### 4. inventory_transactions

Lịch sử nhập/xuất kho để audit và truy vết.

```cql
CREATE TABLE inventory_transactions (
    store_id            UUID,
    created_at          TIMESTAMP,
    id                  UUID,
    product_variant_id  UUID,
    type                TEXT,      -- 'SALE', 'RETURN', 'TRANSFER_IN', 'TRANSFER_OUT'
    quantity_change     INT,       -- Dương = nhập, Âm = xuất
    reference_id        TEXT,      -- Order ID hoặc Transfer ID
    PRIMARY KEY (store_id, created_at, id)
) WITH CLUSTERING ORDER BY (created_at DESC, id ASC);
```

**Query hỗ trợ:**
- Lịch sử giao dịch kho của 1 cửa hàng (mới nhất trước): `WHERE store_id = ?`

### Lưu ý thiết kế

**Denormalization:** `inventory_by_store` và `inventory_by_variant` lưu cùng 1 data nhưng khác partition key. Khi cập nhật tồn kho phải **ghi cả 2 bảng** để đảm bảo đồng bộ.

**Luồng cập nhật tồn kho:**

```
Bán hàng (online/POS):
    inventory_by_store:   quantity -= N
    inventory_by_variant: quantity -= N
    inventory_transactions: INSERT (type: SALE, quantity_change: -N)

Hoàn hàng:
    inventory_by_store:   quantity += N
    inventory_by_variant: quantity += N
    inventory_transactions: INSERT (type: RETURN, quantity_change: +N)

Nhập hàng từ kho tổng vào cửa hàng:
    Kho tổng (WAREHOUSE):
        inventory_by_store:   quantity -= N
        inventory_by_variant: quantity -= N
        inventory_transactions: INSERT (type: TRANSFER_OUT, quantity_change: -N)
    Cửa hàng:
        inventory_by_store:   quantity += N
        inventory_by_variant: quantity += N
        inventory_transactions: INSERT (type: TRANSFER_IN, quantity_change: +N)
```

---

## Analytics Service (DynamoDB)

> DynamoDB thiết kế theo access pattern. Dùng single-table design với PK/SK linh hoạt.

### 1. order_events

Lưu raw event từ SQS, làm source of truth cho mọi aggregation.

| Attribute | Type | Mô tả |
|---|---|---|
| PK | String | `STORE#{store_id}` |
| SK | String | `ORDER#{created_at}#{order_id}#{status}` |
| order_id | String | ID đơn hàng |
| store_id | String | ID cửa hàng |
| type | String | `ONLINE` / `POS` |
| status | String | Trạng thái đơn hàng |
| items | List | Danh sách sản phẩm: `[{variant_id, product_name, category_id, category_path, sku, quantity, price, subtotal}]` |
| total_amount | Number | Tổng tiền |
| created_at | String | ISO 8601, ví dụ: `2025-01-15T10:30:00Z` |

**Query hỗ trợ:**
- Đơn hàng của 1 cửa hàng theo thời gian: `PK = STORE#xxx AND SK BETWEEN ORDER#2025-01-01 AND ORDER#2025-01-31`
- Dedup check: `PK = STORE#xxx AND SK = ORDER#2025-01-15T10:30:00Z#order-123#DELIVERED`

### 2. daily_product_stats

Thống kê sản phẩm theo ngày — aggregated từ order_events.

| Attribute | Type | Mô tả |
|---|---|---|
| PK | String | `STORE#{store_id}#DATE#{yyyy-MM-dd}` |
| SK | String | `VARIANT#{variant_id}` |
| product_name | String | Tên sản phẩm |
| sku | String | Mã SKU |
| category_id | String | ID danh mục |
| category_path | String | Materialized path, ví dụ: `/1/2/3` |
| total_quantity | Number | Tổng số lượng bán |
| total_revenue | Number | Tổng doanh thu |

**GSI: GSI_AllStores_DailyProduct**
| GSI-PK | GSI-SK |
|---|---|
| `DATE#{yyyy-MM-dd}` | `VARIANT#{variant_id}` |

**Query hỗ trợ:**
- Top sản phẩm bán chạy tại 1 cửa hàng theo ngày: `PK = STORE#xxx#DATE#2025-01-15`
- Top sản phẩm bán chạy toàn chuỗi theo ngày: `GSI PK = DATE#2025-01-15`

### 3. daily_category_stats

Thống kê danh mục theo ngày.

| Attribute | Type | Mô tả |
|---|---|---|
| PK | String | `STORE#{store_id}#DATE#{yyyy-MM-dd}` |
| SK | String | `CATEGORY#{category_id}` |
| category_name | String | Tên danh mục |
| category_path | String | Materialized path |
| total_quantity | Number | Tổng số lượng bán |
| total_revenue | Number | Tổng doanh thu |

**GSI: GSI_AllStores_DailyCategory**
| GSI-PK | GSI-SK |
|---|---|
| `DATE#{yyyy-MM-dd}` | `CATEGORY#{category_id}` |

**Query hỗ trợ:**
- Top danh mục tại 1 cửa hàng theo ngày: `PK = STORE#xxx#DATE#2025-01-15`
- Top danh mục toàn chuỗi theo ngày: `GSI PK = DATE#2025-01-15`

### Aggregation Strategy

```
Báo cáo theo ngày:
    → Query trực tiếp daily_product_stats / daily_category_stats

Báo cáo theo tuần:
    → Query 7 ngày → sum ở application layer

Báo cáo theo tháng:
    → Query 28-31 ngày → sum ở application layer

Báo cáo tùy chọn (date range):
    → Query từ ngày A đến ngày B → sum ở application layer

So sánh giữa các kỳ:
    → Query kỳ hiện tại + kỳ trước → tính % thay đổi ở application layer
    Ví dụ: tháng 1 vs tháng 2
        → Query DATE#2025-01-01 đến DATE#2025-01-31
        → Query DATE#2025-02-01 đến DATE#2025-02-28
        → So sánh total_quantity, total_revenue
```

### Luồng ghi data

```
SQS (order-status-queue)
        ↓
[Analytics Worker] Consume message
        ↓
1. Ghi raw event → order_events (ghi MỌI status change, dùng làm audit log)
2. Kiểm tra status trong message:
   - status = DELIVERED (online hoặc POS) → thực hiện aggregation
   - status = RETURN_COMPLETED           → thực hiện reverse aggregation (trừ quantity, trừ revenue)
   - status khác (PENDING, CONFIRMED, PREPARING, SHIPPING, ...) → chỉ ghi raw event, KHÔNG aggregate
3. Aggregate (chỉ khi DELIVERED):
   → update daily_product_stats (quantity += N, revenue += subtotal)
   → update daily_category_stats (quantity += N, revenue += subtotal)
4. Reverse aggregate (chỉ khi RETURN_COMPLETED):
   → update daily_product_stats (quantity -= N, revenue -= subtotal)
   → update daily_category_stats (quantity -= N, revenue -= subtotal)
```

> Đơn POS và đơn Online đều được consume từ `order-status-queue` (không có `analytics-queue` riêng).

### Idempotency cho Aggregation

SQS có thể deliver message **at-least-once** → cùng 1 event có thể đến 2 lần → aggregate bị duplicate.

**Giải pháp:** Dùng DynamoDB conditional write với dedup key:

```
Dedup key: order_id + status (ví dụ: "order-123#DELIVERED")

Trước khi aggregate:
1. Kiểm tra order_events đã có record với order_id + status này chưa
2. Dùng ConditionExpression: attribute_not_exists(PK) AND attribute_not_exists(SK)
   khi ghi raw event → nếu đã tồn tại → skip aggregate
3. Nếu ghi thành công (chưa tồn tại) → thực hiện aggregate
```

**Lưu ý:** `order_events` SK hiện tại là `ORDER#{created_at}#{order_id}` — cần thêm status vào SK để phân biệt các event khác nhau của cùng 1 order:

```
SK: ORDER#{created_at}#{order_id}#{status}
Ví dụ: ORDER#2025-01-15T10:30:00Z#abc-123#DELIVERED
```

Điều này đảm bảo mỗi cặp (order_id, status) chỉ được ghi 1 lần → aggregate không bị duplicate.
