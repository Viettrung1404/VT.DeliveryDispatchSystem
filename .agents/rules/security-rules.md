# Security Rules — Quy tắc bảo mật

> Bảo mật là **ưu tiên số 1**. Mọi code đều phải được review dưới góc độ security.

---

## 1. Nguyên tắc cốt lõi

- **Defense in Depth** — Bảo mật nhiều tầng, không dựa vào một lớp duy nhất
- **Least Privilege** — Chỉ cấp quyền tối thiểu cần thiết
- **Fail Secure** — Khi lỗi xảy ra, mặc định là **từ chối** (deny by default)
- **Never Trust Client Input** — Validate mọi input từ client
- **Secrets KHÔNG BAO GIỜ vào code** — Dùng env variables hoặc secret manager

---

## 2. Authentication (Xác thực)

### JWT + Spring Security + OAuth2

```
Client → [Login] → Backend → [Verify credentials] → Generate JWT
Client → [Request + JWT] → Backend → [Validate JWT] → Process request
```

### Quy tắc JWT

| Thuộc tính | Giá trị |
|---|---|
| Algorithm | RS256 (RSA) — **KHÔNG** dùng HS256 trong production |
| Access Token TTL | 15 phút |
| Refresh Token TTL | 7 ngày |
| Token storage (Web) | HttpOnly Secure Cookie — **KHÔNG** localStorage |
| Token storage (Mobile) | Secure Storage (flutter_secure_storage) |

### Refresh Token Flow

1. Access token hết hạn → client gửi refresh token
2. Backend validate refresh token → issue access token mới + refresh token mới
3. Refresh token cũ bị **invalidate** (rotation)
4. Nếu refresh token bị dùng lại → **revoke tất cả** tokens của user (phát hiện đánh cắp)

### OAuth2 Integration

- Hỗ trợ: Google, Facebook (tùy yêu cầu)
- OAuth2 chỉ để **xác thực danh tính**, sau đó issue JWT nội bộ
- Lưu `oauth_provider` và `oauth_id` trong bảng users
- **KHÔNG** lưu OAuth access token của third-party

---

## 3. Authorization (Phân quyền)

### Role-Based Access Control (RBAC)

```
USER → có nhiều ROLES → mỗi ROLE có nhiều PERMISSIONS
```

### Các roles mặc định

| Role | Mô tả | Ví dụ quyền |
|---|---|---|
| `ADMIN` | Quản trị hệ thống | Toàn quyền |
| `MANAGER` | Quản lý điều vận | CRUD đơn hàng, assign shipper |
| `DISPATCHER` | Điều phối viên | Xem đơn, assign shipper |
| `SHIPPER` | Người giao hàng | Xem đơn được assign, cập nhật trạng thái |
| `CUSTOMER` | Khách hàng | Tạo đơn, theo dõi đơn của mình |

### Quy tắc phân quyền

- **Mọi endpoint** phải có annotation `@PreAuthorize` hoặc config trong SecurityFilterChain
- **Không** hardcode role check trong business logic — dùng Spring Security expressions
- **Row-level security**: User chỉ truy cập dữ liệu thuộc về mình
- API mặc định là **authenticated** — phải explicit mark `permitAll()` cho public endpoints

```java
// ✅ Đúng
@PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
@GetMapping("/orders")
public ResponseEntity<List<OrderDto>> getAllOrders() { ... }

// ✅ Row-level security
@PreAuthorize("hasRole('CUSTOMER')")
@GetMapping("/orders/my")
public ResponseEntity<List<OrderDto>> getMyOrders(@AuthenticationPrincipal UserDetails user) {
    return orderService.findByCustomerId(user.getId());
}

// ❌ Sai — hardcode role check
if (user.getRole().equals("ADMIN")) { ... }
```

---

## 4. Input Validation

### Backend Validation (Bắt buộc)

```java
// Dùng Jakarta Validation annotations
public record CreateOrderRequest(
    @NotBlank(message = "Tên khách hàng không được để trống")
    @Size(max = 100, message = "Tên khách hàng tối đa 100 ký tự")
    String customerName,

    @NotBlank(message = "Địa chỉ không được để trống")
    @Size(max = 500)
    String address,

    @NotNull(message = "Phương thức thanh toán là bắt buộc")
    PaymentMethod paymentMethod,

    @Positive(message = "Số tiền phải lớn hơn 0")
    BigDecimal amount
) {}
```

### Quy tắc validation

| Loại tấn công | Phòng chống |
|---|---|
| SQL Injection | Dùng JPA/Hibernate parameterized queries — **KHÔNG** string concatenation |
| XSS | Escape output, CSP headers, dùng React (auto-escape) |
| CSRF | CSRF token cho web, SameSite cookie |
| Path Traversal | Validate file paths, whitelist allowed directories |
| Mass Assignment | Dùng DTO — **KHÔNG** bind trực tiếp vào Entity |
| Integer Overflow | Validate ranges, dùng `BigDecimal` cho tiền tệ |

### Frontend Validation (Phụ trợ)

- Validate ở frontend **chỉ để UX** — KHÔNG thay thế backend validation
- Dùng Zod (Next.js) hoặc form validation (Flutter) để validate trước khi gửi

---

## 5. Secret Management

### TUYỆT ĐỐI KHÔNG

- ❌ Commit secrets vào Git (kể cả branch cá nhân)
- ❌ Hardcode API keys, passwords trong code
- ❌ Log secrets ra console/file
- ❌ Gửi secrets qua chat/email không mã hóa
- ❌ Dùng chung credentials cho dev/staging/production

### Cách quản lý đúng

```
# .env.example (commit vào repo — chỉ chứa key names)
DATABASE_URL=
REDIS_URL=
JWT_SECRET=
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=

# .env (KHÔNG commit — nằm trong .gitignore)
DATABASE_URL=postgresql://user:pass@localhost:5432/delivery
JWT_SECRET=super-secret-key-here
```

### .gitignore bắt buộc

```gitignore
# Secrets
.env
.env.local
.env.production
*.pem
*.key

# IDE
.idea/
.vscode/settings.json

# Build
build/
node_modules/
.dart_tool/
```

---

## 6. API Security

### Headers bắt buộc

```yaml
# Response headers
Content-Security-Policy: default-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### Rate Limiting

| Endpoint | Limit |
|---|---|
| Login/Register | 5 requests / phút / IP |
| API chung | 100 requests / phút / user |
| File upload | 10 requests / phút / user |
| Password reset | 3 requests / giờ / email |

### CORS Configuration

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "https://delivery.example.com",  // Production
            "http://localhost:3000"           // Dev only
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        config.setAllowCredentials(true);
        // KHÔNG dùng "*" cho origins trong production
    }
}
```

---

## 7. Database Security

- **KHÔNG** dùng root/admin account cho application
- Tạo database user riêng với quyền tối thiểu (SELECT, INSERT, UPDATE, DELETE)
- **KHÔNG** grant DROP, ALTER cho app user
- Encrypt sensitive columns (email, phone) nếu cần
- Enable SSL/TLS cho database connections
- Passwords phải hash bằng **BCrypt** (cost factor ≥ 10)

---

## 8. Quy tắc cho AI Agent

- **KHÔNG BAO GIỜ** tạo code chứa hardcoded secrets
- **LUÔN** dùng environment variables cho configuration
- **LUÔN** thêm input validation cho mọi endpoint mới
- **LUÔN** thêm authorization check cho mọi endpoint mới
- **KHÔNG** disable security features "để test" mà không ghi chú rõ ràng
- Khi tạo endpoint mới → xác định role nào được truy cập
- Khi thêm field mới → đánh giá có phải sensitive data không
