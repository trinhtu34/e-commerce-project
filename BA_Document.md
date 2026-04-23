# Tech stack sử dụng

Frontend	Vue 3 + TypeScript
Backend	ASP.NET Core (C#)
Message Queue	AWS SQS
DB 1	MySQL + EF Core
DB 2	DynamoDB + AWS SDK
DB 3	ScyllaDB + CQL driver
Cache	Redis (AWS ElastiCache)
Container	Docker + Docker Compose
API Gateway	Envoy API Gateway

Vì sẽ triển khai bằng k8s

---

# Tổng quan hệ thống

Website thương mại điện tử cho một chuỗi cửa hàng bán lẻ đa danh mục (mỹ phẩm, vật dụng gia đình, đồ ăn đóng gói, v.v.) hoạt động trong phạm vi một thành phố. Hệ thống bao gồm nhiều cửa hàng thuộc cùng một doanh nghiệp, mỗi cửa hàng có tồn kho riêng và một kho tổng duy nhất cho toàn chuỗi.

---

# Quy tắc chung

- Toàn hệ thống áp dụng **soft delete** (is_deleted + deleted_at), không xóa dữ liệu thật
- Mọi entity đều có `created_at`, `updated_at`, `is_deleted`, `deleted_at`

---

# Nghiệp vụ hệ thống

## 1. Tài khoản & Xác thực

- Sử dụng **AWS Cognito** để quản lý xác thực
- Hỗ trợ đăng nhập bằng:
  - Số điện thoại
  - Email
  - Google (OAuth2)
- Xác thực API bằng **JWT token** do Cognito cấp
- Phân quyền gồm 4 role:
  - **Khách hàng**: đặt hàng, xem lịch sử đơn hàng
  - **Nhân viên cửa hàng**: xử lý đơn hàng, cập nhật trạng thái
  - **Cửa hàng trưởng**: quản lý nhân viên, xem báo cáo cửa hàng
  - **Admin (chủ chuỗi)**: toàn quyền, xem báo cáo toàn chuỗi
- Thông tin profile người dùng (họ tên, SĐT, email, giới tính) được quản lý bởi **User Service**, dùng Cognito `sub` làm định danh
- Nhân viên và cửa hàng trưởng được gắn với cửa hàng qua bảng trung gian `store_staff` (lưu trong User Service)

### Địa chỉ người dùng:
- 1 user có **nhiều địa chỉ** giao hàng
- Mỗi địa chỉ có **tọa độ** (latitude, longitude) để tính cửa hàng gần nhất
- Có 1 địa chỉ **mặc định** (is_default), hệ thống tự chọn khi đặt hàng
- Khi set địa chỉ khác làm default → địa chỉ cũ tự chuyển về non-default

## 2. Sản phẩm & Danh mục

- Sản phẩm đa danh mục: mỹ phẩm, vật dụng gia đình, đồ ăn đóng gói, v.v.
- Danh mục **dynamic cấp** sử dụng Materialized Path (không giới hạn số cấp), ví dụ: `Vật dụng → Đồ gia dụng → Bếp`
- Sản phẩm gắn vào danh mục cấp lá (cấp cuối cùng)
- Mỗi sản phẩm có thể có nhiều variant, mỗi variant là **1 tổ hợp thuộc tính** (ví dụ: `Dung tích: 330ml + Hương vị: Cola`)
- Thuộc tính variant lưu dạng **JSON** (linh hoạt, mỗi sản phẩm có thuộc tính khác nhau)
- Mỗi variant có **mã SKU riêng** (ví dụ: `PEPSI-330-COLA`)
- Đơn vị tính đồng nhất: **cái** cho tất cả sản phẩm

### Giá sản phẩm:
- **Giá đồng nhất toàn chuỗi**, không phân biệt cửa hàng
- Giá có **2 level**:
  - **Product level (base_price)**: giá hiển thị chung (ví dụ: "Pepsi từ 8.000đ")
  - **Variant level (price)**: giá thực tế khi mua (ví dụ: 330ml = 8.000đ, 1.5L = 18.000đ)
- Có lưu **lịch sử thay đổi giá** cho cả product level và variant level

### Trạng thái sản phẩm:
- `ACTIVE`: đang kinh doanh
- `OUT_OF_STOCK`: hết hàng (hệ thống tự cập nhật)
- `SUSPENDED`: tạm dừng kinh doanh (admin set thủ công)

- Ảnh sản phẩm lưu trên **AWS S3**, hệ thống chỉ lưu URL, truy cập qua Pre-Signed URL (Lambda + API Gateway)
- Hỗ trợ tìm kiếm và lọc sản phẩm theo danh mục, giá, v.v.

## 3. Cửa hàng & Kho

- Nhiều cửa hàng trong cùng một thành phố, mỗi cửa hàng có tồn kho riêng
- Một **kho tổng** duy nhất cho toàn chuỗi, được coi là **1 cửa hàng đặc biệt** với type = `WAREHOUSE`
- Tồn kho lưu theo dạng: **số lượng còn lại của từng cửa hàng** + **số lượng còn lại của kho tổng**
- Có nghiệp vụ **nhập hàng từ kho tổng vào cửa hàng**: số lượng kho tổng giảm, số lượng cửa hàng tăng tương ứng (dùng **Saga pattern** — xem mục 10)

### Luồng mua hàng online của khách:

1. Khách chọn sản phẩm và variant muốn mua
2. Khách thêm vào **giỏ hàng** (giỏ hàng gắn với tài khoản, chưa gắn cửa hàng)
3. Khi chuẩn bị thanh toán, hệ thống hiển thị **danh sách cửa hàng**:
   - Chỉ cửa hàng có **đủ tất cả variant** trong giỏ hàng (đúng số lượng) mới được **enable** để chọn (**all-or-nothing**)
   - Cửa hàng thiếu bất kỳ variant nào hoặc không đủ số lượng → hiển thị nhưng bị **disable**, kèm thông tin variant nào thiếu
   - Cửa hàng gần nhất với địa chỉ khách được **gợi ý ưu tiên** lên đầu (trong số các cửa hàng đủ hàng)
   - Nếu **không có cửa hàng nào** đủ tất cả variant → hiển thị thông báo, gợi ý khách bớt sản phẩm trong giỏ
4. Khách **chọn cửa hàng** và **địa chỉ giao hàng** → xác nhận đặt hàng
5. Đơn hàng được gắn với cửa hàng đã chọn, tồn kho trừ tạm thời tại cửa hàng đó

> **Lưu ý:** Không hỗ trợ partial order (đặt 1 phần giỏ hàng). Khách phải chọn cửa hàng có đủ tất cả variant hoặc tự điều chỉnh giỏ hàng.

### Giỏ hàng:
- Gắn với **tài khoản người dùng**, không gắn cửa hàng
- 1 user chỉ có **1 cart active** tại 1 thời điểm
- Mỗi cart item lưu `product_variant_id + quantity`
- Thêm cùng 1 variant 2 lần → **cộng dồn số lượng**

## 4. Đơn hàng & Tracking

### Luồng đặt hàng bình thường:

```
Tạo đơn hàng (CREATED)
        ↓
Trừ tồn kho tạm thời tại cửa hàng
  - Thành công → chuyển sang Chờ xác nhận (PENDING)
  - Thất bại  → đơn hàng FAILED
        ↓
Chờ xác nhận (PENDING) — quá 1 ngày tự động hủy (CANCELLED)
        ↓
Xác nhận thanh toán (CONFIRMED)
        ↓
Đang chuẩn bị hàng (PREPARING)
        ↓
Đang vận chuyển (SHIPPING)
        ↓
Đã giao hàng thành công (DELIVERED)
```

### Luồng hoàn hàng:

```
Yêu cầu hoàn hàng (RETURN_REQUESTED)
        ↓
Đang chờ lấy hàng từ khách (RETURN_PICKING)
        ↓
Đang vận chuyển hoàn về shop (RETURN_SHIPPING)
        ↓
Người bán xác nhận đã nhận hàng hoàn (RETURN_COMPLETED)
        ↓
Nhân viên xử lý hoàn tiền thủ công
```

### Business rules hoàn hàng:
- Khách chỉ được yêu cầu hoàn hàng trong vòng **2 ngày** kể từ khi đơn hàng DELIVERED. Quá 2 ngày → không cho phép tạo yêu cầu hoàn
- Chỉ hỗ trợ **hoàn toàn bộ đơn hàng**, không hỗ trợ hoàn từng sản phẩm riêng lẻ (partial return)
- Chỉ đơn hàng **ONLINE** mới được hoàn hàng. Đơn **POS** không hỗ trợ hoàn qua hệ thống
- Yêu cầu hoàn hàng **không tự động hủy** — sẽ giữ nguyên trạng thái cho đến khi nhân viên xử lý
- Khi RETURN_COMPLETED:
  - Tồn kho được **hoàn trả** về cửa hàng đã xử lý đơn
  - Nhân viên **hoàn tiền thủ công** cho khách (chuyển khoản hoặc tiền mặt tùy thỏa thuận)
  - Giai đoạn sau khi tích hợp payment gateway thật → hoàn tiền tự động
- Khách cần cung cấp **lý do hoàn hàng** khi tạo yêu cầu (text tự do, không giới hạn lý do)

### Business rules đơn hàng:
- Đơn hàng online **snapshot** thông tin sản phẩm tại thời điểm đặt: **tên, giá variant, SKU, ảnh, thông tin variant (JSON)**
- Đơn hàng quá hạn xác nhận (1 ngày) sẽ **tự động hủy** bởi background job, tồn kho được hoàn trả
- Giá snapshot vào order item là **variant price** (giá thực tế)

## 5. Mua hàng tại quầy (POS)

- Nhân viên sử dụng **giao diện web** để tạo đơn tại quầy
- Khách mua tại quầy **không cần tài khoản**, ghi nhận là **khách vãng lai** (user_id = NULL)
- Đơn POS lưu **staff_id** (nhân viên tạo đơn)
- Luồng xử lý:

```
Nhân viên chọn sản phẩm + số lượng
        ↓
Xác nhận thanh toán (tiền mặt / chuyển khoản)
        ↓
Hệ thống trừ tồn kho cửa hàng ngay lập tức
        ↓
Lưu lịch sử + In hóa đơn
```

- Thanh toán giai đoạn đầu: **mock** (tiền mặt và chuyển khoản)
- Đơn tại quầy được lưu riêng trong lịch sử bán, phục vụ analytics (ghi nhận là **khách mua tại cửa hàng**)

## 6. Thanh toán (Online)

- Giai đoạn đầu: **Mock payment** (giả lập thanh toán thành công/thất bại)
- Giai đoạn sau: Tích hợp cổng thanh toán thật (VNPay, MoMo hoặc Stripe)
- Phương thức thanh toán: CASH (POS) / TRANSFER (POS) / ONLINE
- Lưu lịch sử giao dịch cho mỗi đơn hàng

## 7. Thông báo (Notification)

- Gửi email qua **AWS SES**
- Trigger khi trạng thái đơn hàng thay đổi
- Các email cần gửi:
  - Xác nhận đặt hàng thành công
  - Đơn hàng đang được chuẩn bị
  - Đơn hàng đang vận chuyển
  - Giao hàng thành công
  - Đơn hàng bị **tự động hủy** do quá hạn xác nhận
  - Xác nhận yêu cầu hoàn hàng
  - Hoàn hàng hoàn tất

## 8. Analytics & Báo cáo

- Track **danh mục sản phẩm** đang được ưa chuộng (theo số lượng đơn hàng, doanh thu)
- Track **sản phẩm cụ thể** đang bán chạy
- Báo cáo theo: **ngày / tuần / tháng / tùy chọn khoảng thời gian**
- Hỗ trợ **so sánh giữa các kỳ** (ví dụ: tháng 1 vs tháng 2) cho cả danh mục và sản phẩm
- Phân cấp báo cáo:
  - **Cửa hàng trưởng**: xem báo cáo của cửa hàng mình
  - **Admin**: xem báo cáo từng cửa hàng + tổng hợp toàn chuỗi

## 9. Cache Strategy

- **Order Service** cache thông tin product (tên, giá, SKU, ảnh, variant attributes) trong **Redis (ElastiCache)**
- Chiến lược: **Cache-aside + SQS invalidation**
  - Order Service tự đọc từ Redis, nếu miss → gọi HTTP sang Product Service → ghi vào Redis
  - Khi product thay đổi → Product Service publish `product-updated-queue` → Order Service consume → invalidate cache
  - Product Service **không** ghi trực tiếp vào Redis của Order Service (đảm bảo service independence)

## 10. Quy tắc kỹ thuật

### Order Status State Machine

Chỉ cho phép các transition sau, implement validation trong Order Service Domain layer:

```
PENDING → CONFIRMED → PREPARING → SHIPPING → DELIVERED
PENDING → CANCELLED
CONFIRMED → CANCELLED
DELIVERED → RETURN_REQUESTED → RETURN_PICKING → RETURN_SHIPPING → RETURN_COMPLETED
```

### Auto-cancel Idempotency

- Background job chỉ cancel nếu `status = PENDING` (optimistic locking)
- Dùng **outbox pattern**: ghi cancel event trong cùng transaction với update order
- Đảm bảo inventory restoration không bị duplicate

### Concurrent Inventory Deduction

- Dùng ScyllaDB **conditional update** để tránh overselling:
```cql
UPDATE inventory_by_store
SET quantity = quantity - N
WHERE store_id = ? AND product_variant_id = ?
IF quantity >= N;
```

### Inventory Transfer (Saga Pattern)

- Nhập hàng từ kho tổng vào cửa hàng dùng **Saga pattern**
- Ghi `inventory_transactions` trước làm source of truth
- Retry dựa trên transaction log nếu bước nào fail

### Đặt hàng Online (Saga Pattern)

```
1. Order Service tạo đơn (status: CREATED)
2. Gọi Store & Inventory Service trừ kho
3. Thành công → update status: PENDING
4. Thất bại → update status: FAILED
```

### Technical Debt có kế hoạch

- Bảng `payments` hiện nằm trong Order Service DB (giai đoạn mock)
- Khi tích hợp payment gateway thật → tách ra **Payment Service** với DB riêng
- Order Service chỉ lưu `payment_status` + `payment_id` (reference)

### POS Connectivity

- POS yêu cầu **kết nối internet** để hoạt động
- Offline mode không nằm trong scope hiện tại

### API Versioning

- Dùng **URL versioning**: `/api/v1/...`

### price_history

- Là **append-only log**, không áp dụng soft delete (exception của quy tắc chung)

### SQS Queue Design

- Mỗi queue có **Dead Letter Queue (DLQ)** riêng
- `maxReceiveCount: 3` — sau 3 lần retry thất bại → chuyển vào DLQ
- Đơn POS publish qua `order-status-queue` (status: DELIVERED) thay vì `analytics-queue` riêng

### Sync Service-to-Service Calls

| Caller | Callee | Mục đích | Khi nào |
|---|---|---|---|
| Order Service | Store & Inventory Service | Kiểm tra + trừ tồn kho | Khi tạo đơn hàng |
| Order Service | Product Service | Lấy product info (cache miss) | Khi Redis cache miss |
| Order Service | User Service | Lấy thông tin địa chỉ | Khi tạo đơn hàng online |
| Order Service | Store & Inventory Service | Hoàn trả tồn kho | Khi cancel/return đơn |

- Tất cả sync call cần có **timeout** và **retry** policy
- Cần implement **circuit breaker** cho các call quan trọng
