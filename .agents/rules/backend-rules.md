# Backend Rules — Quy tắc phát triển Backend

> Áp dụng cho **Java 17+ / Spring Boot 3.x**. Kiến trúc layered, module-based.

---

## 1. Kiến trúc tổng quan

```
Client Request
    ↓
Controller (REST API layer)
    ↓ DTO
Service (Business logic layer)
    ↓ Entity
Repository (Data access layer)
    ↓ SQL
Database (PostgreSQL)
```

### Quy tắc layer

| Layer | Chức năng | Biết layer nào | KHÔNG được làm |
|---|---|---|---|
| **Controller** | Nhận request, validate, trả response | Service | ❌ Business logic, ❌ Truy cập DB |
| **Service** | Xử lý business logic | Repository, các Service khác | ❌ HTTP logic, ❌ Trả ResponseEntity |
| **Repository** | Truy cập database | Entity | ❌ Business logic |
| **DTO** | Transfer data giữa layers | Không biết ai | ❌ Logic |
| **Entity** | Map với database table | Không biết ai | ❌ Logic phức tạp |
| **Mapper** | Convert DTO ↔ Entity | DTO, Entity | ❌ Logic |

---

## 2. Cấu trúc Package

```
com.delivery/
├── DeliveryApplication.java        # Main class
├── config/
│   ├── SecurityConfig.java         # Spring Security config
│   ├── RedisConfig.java
│   ├── CorsConfig.java
│   ├── SwaggerConfig.java
│   └── WebMvcConfig.java
├── controller/
│   ├── OrderController.java
│   ├── ShipperController.java
│   └── AuthController.java
├── service/
│   ├── OrderService.java           # Interface
│   ├── impl/
│   │   └── OrderServiceImpl.java   # Implementation
│   └── ShipperService.java
├── repository/
│   ├── OrderRepository.java
│   └── ShipperRepository.java
├── model/
│   ├── entity/
│   │   ├── Order.java
│   │   ├── User.java
│   │   └── BaseEntity.java        # Audit fields
│   └── enums/
│       ├── OrderStatus.java
│       └── UserRole.java
├── dto/
│   ├── request/
│   │   ├── CreateOrderRequest.java
│   │   └── UpdateOrderRequest.java
│   ├── response/
│   │   ├── OrderResponse.java
│   │   └── ApiResponse.java
│   └── mapper/
│       └── OrderMapper.java
├── exception/
│   ├── GlobalExceptionHandler.java
│   ├── ResourceNotFoundException.java
│   ├── BusinessException.java
│   └── ErrorCode.java
├── security/
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java
│   └── CustomUserDetailsService.java
├── util/
│   └── DateUtils.java
└── constant/
    └── AppConstants.java
```

---

## 3. Controller Layer

### Pattern chuẩn

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Order", description = "API quản lý đơn hàng")
public class OrderController {

    private final OrderService orderService;

    @GetMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER', 'DISPATCHER')")
    @Operation(summary = "Lấy danh sách đơn hàng")
    public ResponseEntity<ApiResponse<Page<OrderResponse>>> getOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) OrderStatus status) {
        Page<OrderResponse> orders = orderService.getOrders(page, size, status);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER', 'CUSTOMER')")
    @Operation(summary = "Tạo đơn hàng mới")
    public ResponseEntity<ApiResponse<OrderResponse>> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        OrderResponse order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(order, "Order created successfully"));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Lấy chi tiết đơn hàng")
    public ResponseEntity<ApiResponse<OrderResponse>> getOrder(
            @PathVariable Long id) {
        OrderResponse order = orderService.getOrderById(id);
        return ResponseEntity.ok(ApiResponse.success(order));
    }
}
```

### Quy tắc Controller

- **CHỈ** làm: nhận request, validate (via `@Valid`), gọi service, trả response
- **KHÔNG** chứa business logic
- **KHÔNG** inject Repository — chỉ inject Service
- Mọi endpoint có `@PreAuthorize` hoặc security config
- Mọi endpoint có Swagger annotations (`@Operation`, `@Tag`)
- Return `ResponseEntity<ApiResponse<T>>`

---

## 4. Service Layer

### Pattern chuẩn

```java
// Interface
public interface OrderService {
    Page<OrderResponse> getOrders(int page, int size, OrderStatus status);
    OrderResponse getOrderById(Long id);
    OrderResponse createOrder(CreateOrderRequest request);
    OrderResponse updateOrderStatus(Long id, OrderStatus status);
    void deleteOrder(Long id);
}

// Implementation
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;
    private final ShipperService shipperService;

    @Override
    @Transactional(readOnly = true)
    public Page<OrderResponse> getOrders(int page, int size, OrderStatus status) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Page<Order> orders = (status != null)
            ? orderRepository.findByStatus(status, pageable)
            : orderRepository.findAll(pageable);
        return orders.map(orderMapper::toResponse);
    }

    @Override
    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        Order order = orderMapper.toEntity(request);
        order.setStatus(OrderStatus.PENDING);
        Order saved = orderRepository.save(order);
        log.info("Đơn hàng mới được tạo: id={}", saved.getId());
        return orderMapper.toResponse(saved);
    }

    @Override
    @Transactional(readOnly = true)
    public OrderResponse getOrderById(Long id) {
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
        return orderMapper.toResponse(order);
    }
}
```

### Quy tắc Service

- **Interface + Implementation** pattern (dễ test, dễ thay thế)
- `@Transactional` cho write operations, `@Transactional(readOnly = true)` cho read
- **KHÔNG** inject Controller, **KHÔNG** dùng HttpServletRequest
- **KHÔNG** return Entity ra ngoài — chỉ return DTO/Response
- Log business-critical events: tạo đơn, hủy đơn, assign shipper
- Throw custom exception khi business rule bị vi phạm

---

## 5. Entity & Database

### Base Entity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

### Quy tắc Entity

- Kế thừa `BaseEntity` cho audit fields
- Dùng `@Enumerated(EnumType.STRING)` cho enum fields
- Dùng `BigDecimal` cho tiền tệ — **KHÔNG** dùng `double`/`float`
- Relationships: ưu tiên `LAZY` fetch — **KHÔNG** dùng `EAGER` trừ khi có lý do
- **KHÔNG** dùng `@Data` của Lombok cho Entity (vấn đề equals/hashCode)
- Dùng `@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor`

### Database Migration — Flyway

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__create_orders_table.sql
├── V3__create_shippers_table.sql
└── V4__add_tracking_columns.sql
```

**Quy tắc migration**:
- Format: `V{number}__{description}.sql` (hai dấu gạch dưới)
- **KHÔNG BAO GIỜ** sửa migration đã chạy — tạo migration mới
- Migration phải **idempotent** nếu có thể
- Test migration trên dev trước khi merge

---

## 6. Redis Caching

### Khi nào cache

| Cache | Ví dụ | TTL |
|---|---|---|
| **Nên cache** | Config, danh sách tĩnh, user session | 5–60 phút |
| **Cân nhắc** | Kết quả query phức tạp, thống kê | 1–5 phút |
| **KHÔNG cache** | Dữ liệu thay đổi liên tục, sensitive data | — |

### Pattern

```java
@Cacheable(value = "orders", key = "#id")
public OrderResponse getOrderById(Long id) { ... }

@CacheEvict(value = "orders", key = "#id")
public OrderResponse updateOrder(Long id, UpdateOrderRequest request) { ... }

@CacheEvict(value = "orders", allEntries = true)
public void clearOrderCache() { ... }
```

---

## 7. Logging

```java
@Slf4j
public class OrderServiceImpl {

    public OrderResponse createOrder(CreateOrderRequest request) {
        log.info("Tạo đơn hàng mới: customer={}", request.getCustomerName());
        // ... logic
        log.info("Đơn hàng đã tạo thành công: id={}", order.getId());
        return orderMapper.toResponse(order);
    }

    public void deleteOrder(Long id) {
        log.warn("Xóa đơn hàng: id={}", id);
        // ...
    }
}
```

### Quy tắc Logging

| Level | Khi nào dùng | Ví dụ |
|---|---|---|
| `ERROR` | Lỗi cần xử lý ngay | Exception, service down |
| `WARN` | Tình huống bất thường nhưng xử lý được | Retry, degraded service |
| `INFO` | Business events quan trọng | Tạo/hủy đơn, login |
| `DEBUG` | Chi tiết kỹ thuật (chỉ bật ở dev) | Query params, intermediate values |

- **CẤM**: Log password, token, PII
- **BẮT BUỘC**: Log request/response ở controller level (via filter/interceptor)

---

## 8. Quy tắc cho AI Agent

- **LUÔN** tuân thủ layered architecture: Controller → Service → Repository
- **KHÔNG** đặt business logic trong Controller hoặc Repository
- **LUÔN** dùng DTO cho input/output — không expose Entity
- **LUÔN** thêm `@Transactional` cho service methods
- **LUÔN** thêm validation annotations cho request DTOs
- **LUÔN** thêm Swagger annotations cho endpoints
- **LUÔN** tạo migration file khi thay đổi database schema
- **KHÔNG** dùng `EAGER` fetch mà không có lý do rõ ràng
- **KHÔNG** dùng `@Data` cho Entity classes
