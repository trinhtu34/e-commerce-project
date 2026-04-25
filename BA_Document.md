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

### Validation Rules khi thêm sản phẩm vào giỏ hàng:

#### 1. Product Status Validation
- **ACTIVE**: Cho phép thêm vào giỏ hàng bình thường
- **SUSPENDED**: Cho phép thêm vào giỏ hàng nhưng hiển thị warning "Sản phẩm tạm ngừng kinh doanh"
- **OUT_OF_STOCK**: Chặn ngay, không cho thêm vào giỏ hàng + hiển thị thông báo "Sản phẩm hiện hết hàng"

#### 2. Inventory Check Strategy
- **Khi thêm vào giỏ hàng**: Không kiểm tra tồn kho (tối ưu UX, khách có thể thêm nhiều sản phẩm trước khi checkout)
- **Khi hiển thị giỏ hàng**: Check xem có cửa hàng nào đủ hàng cho tất cả variant trong giỏ không
- **Khi checkout**: Validate lại tồn kho thực tế tại cửa hàng đã chọn

#### 3. Xử lý khi variant bị thay đổi/xóa
- **Soft delete (is_deleted = TRUE)**:
  - Hiển thị trong giỏ hàng nhưng disable (không thể tăng số lượng)
  - Hiển thị thông báo "Sản phẩm không còn tồn tại"
  - Không cho phép checkout với variant này
- **Hard delete (xóa thật khỏi database)**:
  - Tự động xóa khỏi giỏ hàng
  - Hiển thị thông báo "Sản phẩm đã bị xóa khỏi hệ thống"
- **Price change**:
  - Giỏ hàng lưu snapshot price tại thời điểm thêm
  - Hiển thị warning "Giá sản phẩm đã thay đổi" nếu price hiện tại khác snapshot

#### 4. Quantity Limits
- **Max per variant**: 10 cái (configurable)
- **Max total items**: 50 cái (configurable) - tổng số lượng tất cả variant trong giỏ
- **Max per product**: 20 cái (tất cả variant của cùng 1 sản phẩm)
- **Min quantity**: 1 cái (không cho phép quantity = 0)

**Error messages:**
- "Số lượng tối đa cho mỗi sản phẩm là 10 cái"
- "Giỏ hàng không được quá 50 sản phẩm"
- "Số lượng tối thiểu là 1 cái"

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

### Business rules hủy đơn chủ động:

#### Ai có thể hủy đơn:
- **Khách hàng:** Có thể hủy đơn hàng online của mình
- **Nhân viên cửa hàng:** Có thể hủy đơn hàng online (khi hết hàng, sản phẩm lỗi, khách yêu cầu)
- **Cửa hàng trưởng:** Có thể hủy đơn hàng online
- **Admin:** Có thể hủy bất kỳ đơn hàng nào
- **System:** Auto-cancel (quá hạn xác nhận)

#### Khi nào có thể hủy:
- **Khách hàng:** Có thể hủy khi đơn ở trạng thái `PENDING` hoặc `CONFIRMED`
- **Nhân viên/Cửa hàng trưởng:** Có thể hủy khi đơn ở trạng thái `PENDING`, `CONFIRMED`, hoặc `PREPARING`
- **Admin:** Có thể hủy ở bất kỳ trạng thái nào (trừ `DELIVERED` đã quá 2 ngày, `RETURN_COMPLETED`)
- **System:** Auto-cancel khi đơn `PENDING` quá 1 ngày chưa được xác nhận

#### Lý do hủy đơn:
- **Khách hàng:** "Khách hàng yêu cầu hủy", "Đổi ý", "Tìm được sản phẩm khác", "Lý do khác"
- **Nhân viên/Cửa hàng trưởng:** "Hết hàng", "Sản phẩm lỗi", "Khách hàng yêu cầu", "Lý do khác"
- **System:** "Quá hạn xác nhận"

#### Luồng xử lý khi hủy đơn:
1. Người có quyền hủy đơn → cập nhật status: `CANCELLED`
2. Lưu `cancelled_by` (user/staff/system) và `cancel_reason`
3. Hoàn trả tồn kho về cửa hàng đã xử lý đơn
4. Gửi email thông báo hủy đơn cho khách hàng
5. Nếu đã thanh toán → nhân viên hoàn tiền thủ công (giai đoạn sau → hoàn tiền tự động)

#### Lưu ý:
- Đơn POS không hỗ trợ hủy (đã giao hàng ngay lập tức)
- Khi hủy đơn → luồng hoàn trả inventory giống auto-cancel
- Cần lưu `cancelled_by` và `cancel_reason` để audit và truy vết

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
PREPARING → CANCELLED (chỉ staff/manager/admin)
DELIVERED → RETURN_REQUESTED → RETURN_PICKING → RETURN_SHIPPING → RETURN_COMPLETED
```

**Quy tắc hủy đơn:**
- **Khách hàng:** Có thể hủy từ `PENDING` hoặc `CONFIRMED`
- **Nhân viên/Cửa hàng trưởng:** Có thể hủy từ `PENDING`, `CONFIRMED`, hoặc `PREPARING`
- **Admin:** Có thể hủy từ bất kỳ trạng thái nào (trừ `DELIVERED` đã quá 2 ngày, `RETURN_COMPLETED`)
- **System:** Auto-cancel từ `PENDING` khi quá 1 ngày

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

### Retry Policy

**Exponential Backoff:**
- **Initial delay:** 1 giây (lần retry đầu tiên)
- **Backoff multiplier:** 2 (mỗi lần retry tăng gấp đôi thời gian chờ)
- **Max delay:** 60 giây (không chờ quá 60 giây giữa các retry)
- **Retry sequence:**
  - Retry 1: chờ 1 giây
  - Retry 2: chờ 2 giây
  - Retry 3: chờ 4 giây
  - Retry 4: chờ 8 giây
  - Retry 5: chờ 16 giây
  - Retry 6: chờ 32 giây
  - Retry 7: chờ 60 giây (max)
  - Retry 8+: chờ 60 giây (max)

**Poison Message Handling:**
- Sau `maxReceiveCount: 3` lần retry thất bại → message chuyển vào DLQ
- DLQ message được giữ trong **14 ngày** (AWS SQS default retention period)
- Poison message cần được **manual review** bởi developer/ops team
- Sau khi fix bug → có thể **replay** message từ DLQ về main queue

**DLQ Monitoring & Alerting:**
- **CloudWatch Alarm:** Trigger khi DLQ có > 10 messages trong 5 phút
- **Alarm action:** Gửi email/SNS notification đến dev team
- **Dashboard:** Hiển thị số lượng messages trong DLQ theo thời gian
- **Daily report:** Gửi summary về DLQ messages vào cuối ngày

**Error Classification:**

**Transient Errors (có thể retry):**
- Network timeout
- Database connection error
- External API rate limit (429)
- Temporary service unavailable (503)
- Message processing timeout

**Permanent Errors (không nên retry, chuyển DLQ ngay):**
- Invalid message format (JSON parse error)
- Missing required fields
- Business logic validation error
- Data integrity violation
- Authentication/authorization error

**Implementation Guidelines:**
- Consumer phải log chi tiết error message (stack trace, error code)
- Dùng `message_id` để track message xuyên suốt retry flow
- Implement circuit breaker cho external API calls
- Monitor retry rate: nếu > 50% messages cần retry → có vấn đề nghiêm trọng

**DLQ Replay Strategy:**
- Sau khi fix bug → replay message từ DLQ
- Replay theo batch (max 100 messages/batch)
- Monitor replay success rate
- Nếu replay fail nhiều lần → cần manual investigation

### Sync Service-to-Service Calls

| Caller | Callee | Mục đích | Khi nào |
|---|---|---|---|
| Order Service | Store & Inventory Service | Kiểm tra + trừ tồn kho | Khi tạo đơn hàng |
| Order Service | Product Service | Lấy product info (cache miss) | Khi Redis cache miss |
| Order Service | User Service | Lấy thông tin địa chỉ | Khi tạo đơn hàng online |
| Order Service | Store & Inventory Service | Hoàn trả tồn kho | Khi cancel/return đơn |

- Tất cả sync call cần có **timeout** và **retry** policy
- Cần implement **circuit breaker** cho các call quan trọng
