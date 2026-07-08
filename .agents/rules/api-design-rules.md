# API Design Rules — Quy tắc thiết kế API

> Tất cả API tuân thủ chuẩn **RESTful**. Consistency là chìa khóa.

---

## 1. Nguyên tắc cốt lõi

- **Consistency** — Tất cả endpoints theo cùng pattern
- **Predictability** — Developer nhìn URL đoán được behavior
- **Versioning** — Luôn có version trong URL
- **Documentation-first** — Thiết kế API trước khi code

---

## 2. URL Conventions

### Base URL

```
/api/v1/{resource}
```

### Quy tắc đặt tên

| Quy tắc | Ví dụ đúng ✅ | Ví dụ sai ❌ |
|---|---|---|
| Dùng **danh từ số nhiều** | `/api/v1/orders` | `/api/v1/order`, `/api/v1/getOrders` |
| Dùng **kebab-case** | `/api/v1/delivery-orders` | `/api/v1/deliveryOrders`, `/api/v1/delivery_orders` |
| Nested resources max 2 cấp | `/api/v1/orders/{id}/items` | `/api/v1/orders/{id}/items/{id}/details/{id}` |
| Không dùng **động từ** trong URL | `POST /api/v1/orders` | `/api/v1/createOrder` |
| Actions đặc biệt dùng sub-resource | `POST /api/v1/orders/{id}/cancel` | `POST /api/v1/cancelOrder/{id}` |

### CRUD Mapping

| Hành động | HTTP Method | URL | Ví dụ |
|---|---|---|---|
| Lấy danh sách | `GET` | `/api/v1/orders` | Lấy tất cả đơn hàng |
| Lấy chi tiết | `GET` | `/api/v1/orders/{id}` | Lấy đơn hàng theo ID |
| Tạo mới | `POST` | `/api/v1/orders` | Tạo đơn hàng mới |
| Cập nhật toàn bộ | `PUT` | `/api/v1/orders/{id}` | Cập nhật toàn bộ đơn hàng |
| Cập nhật một phần | `PATCH` | `/api/v1/orders/{id}` | Cập nhật trạng thái |
| Xóa | `DELETE` | `/api/v1/orders/{id}` | Xóa đơn hàng |

---

## 3. Request Format

### Query Parameters (cho GET)

```
GET /api/v1/orders?status=PENDING&page=0&size=20&sort=createdAt,desc
```

| Parameter | Mục đích | Format |
|---|---|---|
| `page` | Số trang (0-indexed) | Integer, default = 0 |
| `size` | Số item per page | Integer, default = 20, max = 100 |
| `sort` | Sắp xếp | `field,direction` (ví dụ: `createdAt,desc`) |
| `search` | Tìm kiếm full-text | String |
| `status` | Filter theo trạng thái | Enum value |
| `from` / `to` | Filter theo thời gian | ISO 8601 (`2024-01-01T00:00:00Z`) |

### Request Body (cho POST/PUT/PATCH)

```json
{
  "customerName": "Nguyễn Văn A",
  "address": "123 Nguyễn Huệ, Q1, TP.HCM",
  "phoneNumber": "0901234567",
  "items": [
    {
      "productId": 1,
      "quantity": 2
    }
  ],
  "paymentMethod": "COD",
  "note": "Giao giờ hành chính"
}
```

**Quy tắc**:
- Dùng **camelCase** cho field names
- Dates dùng **ISO 8601** format
- Tiền tệ dùng **BigDecimal** (string trong JSON nếu cần precision)
- Enum gửi dưới dạng **string** (UPPER_CASE)

---

## 4. Response Format

### Response thành công

```json
// Single item
{
  "status": "success",
  "data": {
    "id": 1,
    "customerName": "Nguyễn Văn A",
    "status": "PENDING",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}

// List với pagination
{
  "status": "success",
  "data": {
    "content": [...],
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8,
    "first": true,
    "last": false
  }
}

// Tạo mới thành công
{
  "status": "success",
  "data": { ... },
  "message": "Order created successfully"
}
```

### Response lỗi

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "customerName",
        "message": "Customer name must not be blank"
      },
      {
        "field": "address",
        "message": "Address must not be blank"
      }
    ]
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/api/v1/orders"
}
```

---

## 5. HTTP Status Codes

### Thành công (2xx)

| Code | Khi nào dùng | Ví dụ |
|---|---|---|
| `200 OK` | Request thành công, trả về data | GET, PUT, PATCH |
| `201 Created` | Tạo resource mới thành công | POST |
| `204 No Content` | Thành công nhưng không trả body | DELETE |

### Lỗi client (4xx)

| Code | Khi nào dùng | Error Code |
|---|---|---|
| `400 Bad Request` | Input không hợp lệ | `VALIDATION_ERROR` |
| `401 Unauthorized` | Chưa đăng nhập / token hết hạn | `UNAUTHORIZED` |
| `403 Forbidden` | Không có quyền truy cập | `FORBIDDEN` |
| `404 Not Found` | Resource không tồn tại | `RESOURCE_NOT_FOUND` |
| `409 Conflict` | Conflict (duplicate, state conflict) | `CONFLICT` |
| `422 Unprocessable Entity` | Logic error (business rule) | `BUSINESS_ERROR` |
| `429 Too Many Requests` | Rate limit exceeded | `RATE_LIMITED` |

### Lỗi server (5xx)

| Code | Khi nào dùng | Error Code |
|---|---|---|
| `500 Internal Server Error` | Lỗi không mong đợi | `INTERNAL_ERROR` |
| `503 Service Unavailable` | Service tạm ngưng | `SERVICE_UNAVAILABLE` |

---

## 6. Error Handling (Backend)

### Custom Error Codes

```java
public enum ErrorCode {
    // Validation
    VALIDATION_ERROR("Validation failed"),
    
    // Auth
    UNAUTHORIZED("Authentication required"),
    FORBIDDEN("Access denied"),
    TOKEN_EXPIRED("Token has expired"),
    
    // Business
    ORDER_NOT_FOUND("Order not found"),
    ORDER_ALREADY_CANCELLED("Order is already cancelled"),
    SHIPPER_NOT_AVAILABLE("No shipper available"),
    
    // System
    INTERNAL_ERROR("An unexpected error occurred"),
    SERVICE_UNAVAILABLE("Service temporarily unavailable");
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<?> handleValidation(MethodArgumentNotValidException ex) { ... }

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<?> handleNotFound(ResourceNotFoundException ex) { ... }

    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ApiResponse<?> handleBusiness(BusinessException ex) { ... }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<?> handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        // KHÔNG trả stack trace cho client
        return ApiResponse.error(ErrorCode.INTERNAL_ERROR);
    }
}
```

---

## 7. Versioning

- Version trong URL: `/api/v1/...`, `/api/v2/...`
- Khi thay đổi breaking: tạo version mới
- Version cũ deprecated nhưng vẫn hoạt động **ít nhất 6 tháng**
- Response header: `X-API-Version: 1`
- Deprecation header: `Sunset: Sat, 01 Jul 2025 00:00:00 GMT`

---

## 8. Pagination, Filtering, Sorting

### Pagination (bắt buộc cho list API)

```
GET /api/v1/orders?page=0&size=20
```

- **Mặc định**: page=0, size=20
- **Tối đa**: size=100 (server enforce)
- Trả về metadata: `totalElements`, `totalPages`, `first`, `last`

### Filtering

```
GET /api/v1/orders?status=PENDING&customerId=123&from=2024-01-01&to=2024-01-31
```

### Sorting

```
GET /api/v1/orders?sort=createdAt,desc&sort=customerName,asc
```

---

## 9. API Documentation

- Dùng **OpenAPI 3.0** (Swagger) — tự generate từ code
- Mọi endpoint phải có:
  - Description (mô tả mục đích)
  - Request/Response examples
  - Error responses
  - Authentication requirements
- Swagger UI available tại `/api/docs` (chỉ dev/staging)

---

## 10. Quy tắc cho AI Agent

- **LUÔN** tuân thủ URL naming convention ở mục 2
- **LUÔN** dùng response format chuẩn ở mục 4
- **LUÔN** thêm pagination cho list API
- **LUÔN** thêm validation cho request body
- **LUÔN** return đúng HTTP status code
- **KHÔNG** trả stack trace trong error response
- **KHÔNG** tạo endpoint trùng chức năng với endpoint đã có
- Mọi endpoint mới phải có Swagger annotations
