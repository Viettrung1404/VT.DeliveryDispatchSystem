# Documentation Rules — Quy tắc viết tài liệu

> Tài liệu tốt = onboarding nhanh + ít hỏi lại + ít bug. Documentation là **code** — phải maintain.

---

## 1. Nguyên tắc cốt lõi

- **Documentation as Code** — Tài liệu nằm cùng repo, review cùng PR
- **Living Documents** — Cập nhật tài liệu khi code thay đổi
- **DRY** — Không duplicate thông tin, link đến source of truth
- **Audience-aware** — Biết viết cho ai (developer, QA, stakeholder)
- **Ngôn ngữ**: Tài liệu viết bằng **tiếng Việt**, code examples bằng tiếng Anh

---

## 2. Cấu trúc Documentation

```
docs/
├── README.md                    # Tổng quan dự án
├── CONTRIBUTING.md              # Hướng dẫn đóng góp
├── CHANGELOG.md                 # Lịch sử thay đổi
├── adr/                         # Architecture Decision Records
│   ├── 001-chon-spring-boot.md
│   ├── 002-chon-postgresql.md
│   └── template.md
├── api/                         # API documentation
│   └── openapi.yml              # Auto-generated từ code
├── guides/                      # Hướng dẫn
│   ├── setup-dev-environment.md
│   ├── deployment-guide.md
│   └── troubleshooting.md
└── diagrams/                    # Sơ đồ kiến trúc
    ├── architecture-overview.png
    └── database-erd.png
```

---

## 3. README.md

### Template README

```markdown
# DeliveryDispatchSystem

> Hệ thống quản lý điều vận giao hàng

## Tổng quan

[Mô tả ngắn gọn dự án: mục đích, đối tượng sử dụng, core features]

## Tech Stack

| Thành phần | Công nghệ |
|---|---|
| Backend | Java 17 / Spring Boot 3.x |
| Web Frontend | Next.js 14 (React, TypeScript) |
| Mobile | Flutter |
| Database | PostgreSQL 15 + Redis 7 |
| Auth | JWT + Spring Security + OAuth2 |

## Yêu cầu hệ thống

- Java 17+
- Node.js 20+
- Flutter 3.x
- Docker & Docker Compose
- PostgreSQL 15+
- Redis 7+

## Cài đặt & Chạy

### Sử dụng Docker (khuyến nghị)

\`\`\`bash
git clone https://github.com/team/DeliveryDispatchSystem.git
cd DeliveryDispatchSystem
cp .env.example .env           # Cấu hình environment variables
docker compose up -d           # Khởi chạy tất cả services
\`\`\`

### Chạy từng service

#### Backend
\`\`\`bash
cd backend
./gradlew bootRun
\`\`\`

#### Frontend
\`\`\`bash
cd frontend
npm install
npm run dev
\`\`\`

#### Mobile
\`\`\`bash
cd mobile
flutter pub get
flutter run
\`\`\`

## Cấu trúc dự án

[Link đến AGENTS.md mục 3]

## API Documentation

- Swagger UI: http://localhost:8080/api/docs
- [Xem chi tiết API](docs/api/)

## Contributing

Xem [CONTRIBUTING.md](docs/CONTRIBUTING.md)

## License

[Ghi license]
```

---

## 4. Architecture Decision Records (ADR)

### Khi nào viết ADR

- Chọn/thay đổi technology
- Quyết định kiến trúc quan trọng
- Trade-off đáng ghi nhận
- Quyết định ảnh hưởng nhiều người/module

### Template ADR

```markdown
# ADR-{số thứ tự}: {Tiêu đề quyết định}

**Ngày**: YYYY-MM-DD
**Trạng thái**: Đề xuất | Chấp nhận | Từ chối | Thay thế bởi ADR-XXX
**Người quyết định**: [Tên]

## Bối cảnh

[Mô tả vấn đề cần quyết định. Tại sao cần đưa ra quyết định này?]

## Các phương án

### Phương án 1: [Tên]
- **Ưu điểm**: ...
- **Nhược điểm**: ...

### Phương án 2: [Tên]
- **Ưu điểm**: ...
- **Nhược điểm**: ...

### Phương án 3: [Tên] (nếu có)
- **Ưu điểm**: ...
- **Nhược điểm**: ...

## Quyết định

Chọn **Phương án X** vì [lý do cụ thể].

## Hệ quả

- [Những thay đổi cần thực hiện]
- [Những ràng buộc từ quyết định này]
- [Rủi ro cần lưu ý]
```

### Ví dụ ADR

```markdown
# ADR-001: Chọn PostgreSQL làm database chính

**Ngày**: 2024-01-15
**Trạng thái**: Chấp nhận
**Người quyết định**: Team Lead

## Bối cảnh

Dự án cần database quan hệ hỗ trợ:
- ACID transactions cho đơn hàng
- Geospatial queries cho tracking shipper
- JSON support cho flexible data

## Các phương án

### MySQL 8
- Ưu: phổ biến, team quen thuộc
- Nhược: Geospatial hạn chế, JSON support kém hơn

### PostgreSQL 15
- Ưu: Geospatial mạnh (PostGIS), JSON native, extensible
- Nhược: Team cần học thêm

### MongoDB
- Ưu: Flexible schema, scale horizontal tốt
- Nhược: Không ACID cross-document, overkill cho use case này

## Quyết định

Chọn **PostgreSQL 15** vì hỗ trợ geospatial tốt (PostGIS)
cho tracking shipper, đồng thời đáp ứng ACID cho transactions.

## Hệ quả

- Cần setup PostGIS extension
- Team cần familiar với PostgreSQL syntax
- Dùng Flyway cho migration management
```

---

## 5. CHANGELOG.md

### Format — Keep a Changelog

```markdown
# Changelog

Tất cả thay đổi đáng chú ý sẽ được ghi nhận ở đây.
Format dựa trên [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Thêm mới
- Tính năng tự động phân bổ shipper (#DDS-234)

### Thay đổi
- Cập nhật API response format theo chuẩn mới

### Sửa lỗi
- Sửa lỗi refresh token không rotate (#DDS-301)

## [1.1.0] - 2024-02-01

### Thêm mới
- API theo dõi đơn hàng real-time
- Push notification cho shipper

### Sửa lỗi
- Sửa lỗi pagination trả sai total count

## [1.0.0] - 2024-01-15

### Thêm mới
- Hệ thống quản lý đơn hàng cơ bản
- Authentication JWT
- Dashboard web cho manager
```

---

## 6. Code Documentation

### Khi nào comment

| Nên comment | KHÔNG nên comment |
|---|---|
| **Tại sao** làm vậy (reasoning) | **Cái gì** code đang làm (tự mô tả) |
| Workaround cho bug/limitation | Getter/Setter hiển nhiên |
| Business rule phức tạp | Code đã rõ ràng |
| TODO với ticket reference | Comment cũ, outdated |
| Public API (Javadoc/JSDoc) | Comment bị vô hiệu hóa thay vì xóa |

### Java — Javadoc

```java
/**
 * Tự động phân bổ shipper gần nhất cho đơn hàng.
 *
 * <p>Thuật toán: nearest-neighbor dựa trên GPS,
 * có weight theo rating của shipper.</p>
 *
 * @param orderId ID đơn hàng cần assign
 * @return Shipper được assign
 * @throws ResourceNotFoundException nếu đơn hàng không tồn tại
 * @throws BusinessException nếu không có shipper available
 */
public ShipperResponse assignShipper(Long orderId) { ... }
```

### TypeScript — JSDoc

```typescript
/**
 * Hook quản lý danh sách đơn hàng với pagination và filter.
 *
 * @param initialFilters - Bộ lọc mặc định
 * @returns Object chứa orders, loading state, và pagination controls
 *
 * @example
 * ```tsx
 * const { orders, isLoading, nextPage } = useOrders({ status: 'PENDING' });
 * ```
 */
export function useOrders(initialFilters?: IOrderFilters) { ... }
```

---

## 7. API Documentation

### Quy tắc Swagger/OpenAPI

- **Mọi endpoint** phải có annotations đầy đủ
- Mô tả endpoint bằng tiếng Việt
- Include request/response examples
- Document tất cả error responses có thể xảy ra
- Group endpoints theo `@Tag`

```java
@Operation(
    summary = "Tạo đơn hàng mới",
    description = "Tạo đơn hàng mới. Yêu cầu role CUSTOMER hoặc ADMIN.",
    responses = {
        @ApiResponse(responseCode = "201", description = "Tạo thành công"),
        @ApiResponse(responseCode = "400", description = "Dữ liệu không hợp lệ"),
        @ApiResponse(responseCode = "401", description = "Chưa đăng nhập"),
        @ApiResponse(responseCode = "403", description = "Không có quyền")
    }
)
@PostMapping
public ResponseEntity<ApiResponse<OrderResponse>> createOrder(
    @Valid @RequestBody CreateOrderRequest request) { ... }
```

---

## 8. Quy tắc cho AI Agent

- **LUÔN** cập nhật documentation khi thay đổi behavior
- **LUÔN** viết Javadoc/JSDoc cho public methods
- **LUÔN** viết ADR khi đề xuất quyết định kiến trúc
- **KHÔNG** để comment outdated — xóa hoặc cập nhật
- **KHÔNG** comment code thay vì xóa — dùng Git để lưu lịch sử
- Khi tạo API mới → cập nhật Swagger annotations
- Khi thay đổi setup → cập nhật README
- Comment bằng tiếng Anh trong code, tiếng Việt trong docs
