# Testing Rules — Quy tắc kiểm thử

> Mọi code mới **BẮT BUỘC** có test. Không có test = không merge.

---

## 1. Nguyên tắc cốt lõi

- **Test là tài liệu sống** — đọc test phải hiểu được behavior
- **Test trước, code sau** (TDD) — khuyến khích nhưng không bắt buộc
- **Mỗi test chỉ test một thứ** — một assertion chính per test
- **Test phải deterministic** — chạy 100 lần đều ra cùng kết quả
- **Test phải nhanh** — unit test < 100ms, integration test < 5s
- **Test phải độc lập** — không phụ thuộc thứ tự chạy hoặc test khác

---

## 2. Phân loại Tests

### 2.1. Unit Test (Bắt buộc)

| Thuộc tính | Chi tiết |
|---|---|
| **Phạm vi** | Một class/function duy nhất |
| **Dependencies** | Mock tất cả external dependencies |
| **Tốc độ** | < 100ms per test |
| **Khi nào viết** | Cho MỌI business logic, service, utility |
| **Coverage yêu cầu** | ≥ 80% cho business logic |

### 2.2. Integration Test (Bắt buộc cho API)

| Thuộc tính | Chi tiết |
|---|---|
| **Phạm vi** | Nhiều components phối hợp |
| **Dependencies** | Database thật (testcontainers), HTTP thật |
| **Tốc độ** | < 5s per test |
| **Khi nào viết** | API endpoints, database queries, external services |
| **Coverage yêu cầu** | Tất cả API endpoints phải có ít nhất 1 integration test |

### 2.3. E2E Test (Cho critical flows)

| Thuộc tính | Chi tiết |
|---|---|
| **Phạm vi** | Toàn bộ user flow từ UI đến DB |
| **Dependencies** | Hệ thống chạy đầy đủ |
| **Tốc độ** | < 30s per test |
| **Khi nào viết** | Login, checkout, payment, core business flows |

---

## 3. Testing theo Tech Stack

### 3.1. Backend — Java Spring Boot

**Framework**: JUnit 5 + Mockito + Spring Boot Test + Testcontainers

```java
// Naming convention: methodName_condition_expectedResult
@Test
void createOrder_validInput_returnsCreatedOrder() { ... }

@Test
void createOrder_nullAddress_throwsValidationException() { ... }

@Test
void findOrdersByStatus_noOrders_returnsEmptyList() { ... }
```

**Cấu trúc test file**:

```
src/test/java/com/delivery/
├── service/
│   └── OrderServiceTest.java           # Unit test
├── controller/
│   └── OrderControllerIntegrationTest.java  # Integration test
├── repository/
│   └── OrderRepositoryTest.java        # Repository test
└── testutil/
    └── TestDataFactory.java            # Test data builders
```

**Quy tắc**:
- Unit test: Mock Repository, mock external services
- Controller test: Dùng `@WebMvcTest` + MockMvc
- Repository test: Dùng `@DataJpaTest` + Testcontainers (PostgreSQL)
- Service test: Dùng `@ExtendWith(MockitoExtension.class)`
- Dùng `TestDataFactory` hoặc Builder pattern cho test data

### 3.2. Frontend — Next.js (React + TypeScript)

**Framework**: Jest + React Testing Library + MSW (Mock Service Worker)

```typescript
// Naming convention: describe + it (hành vi)
describe('OrderTable', () => {
  it('hiển thị danh sách đơn hàng khi có dữ liệu', () => { ... });
  it('hiển thị empty state khi không có dữ liệu', () => { ... });
  it('gọi onDelete khi click nút xóa', () => { ... });
});
```

**Quy tắc**:
- Test **behavior**, không test implementation details
- Dùng `screen.getByRole()`, `getByText()` — tránh `getByTestId()` trừ khi bắt buộc
- Mock API calls bằng MSW
- Snapshot test chỉ cho UI ít thay đổi (icons, static layouts)
- Không test internal state của component

### 3.3. Mobile — Flutter (Dart)

**Framework**: flutter_test + mockito + bloc_test

```dart
// Naming convention: group + test (hành vi)
group('OrderBloc', () {
  test('phát ra OrderLoaded khi fetch thành công', () { ... });
  test('phát ra OrderError khi API trả lỗi', () { ... });
});
```

**Quy tắc**:
- Widget test cho mọi screen và reusable widget
- Bloc/Cubit test cho mọi state management logic
- Dùng `mocktail` hoặc `mockito` cho dependency injection
- Golden tests cho UI critical components

---

## 4. Test Coverage

### Yêu cầu tối thiểu

| Layer | Coverage tối thiểu | Ghi chú |
|---|---|---|
| Service/UseCase | **80%** | Business logic — ưu tiên cao nhất |
| Controller/Handler | **70%** | API layer |
| Repository | **60%** | Data access (integration test) |
| Utility/Helper | **90%** | Shared code dùng nhiều nơi |
| DTO/Model | Không yêu cầu | Chỉ là data class |
| Config | Không yêu cầu | Cấu hình |

### Không tính coverage cho

- Generated code (Lombok, MapStruct, protobuf)
- Configuration classes
- DTO/Entity classes thuần túy
- Main application class

---

## 5. Test Data Management

### Quy tắc

- **KHÔNG** dùng dữ liệu thật (production data) trong test
- **KHÔNG** hardcode test data rải rác — tập trung vào factory/builder
- **LUÔN** cleanup data sau test (dùng `@Transactional` hoặc teardown)
- Dùng **TestDataFactory** pattern:

```java
public class TestDataFactory {
    public static Order createValidOrder() {
        return Order.builder()
            .customerName("Test Customer")
            .address("123 Test Street")
            .status(OrderStatus.PENDING)
            .build();
    }

    public static Order createOrderWithStatus(OrderStatus status) {
        return createValidOrder().toBuilder()
            .status(status)
            .build();
    }
}
```

---

## 6. Những gì KHÔNG cần test

- Getter/Setter đơn thuần
- Framework code (Spring Boot auto-config, Next.js routing)
- Third-party library internals
- Generated code
- Constants và Enums (trừ khi có logic)

---

## 7. CI/CD Integration

```yaml
# Test phải pass trong CI trước khi merge
test-pipeline:
  steps:
    - unit-tests          # Chạy đầu tiên, nhanh nhất
    - integration-tests   # Chạy sau, cần database
    - coverage-check      # Kiểm tra coverage threshold
    - lint-check          # Code style
```

### Quy tắc CI

- **Tất cả tests phải pass** — không cho phép skip
- Coverage drop > 5% → **block merge**
- New code coverage < 80% → **warning** (< 60% → block)
- Flaky tests phải fix trong **48 giờ** hoặc skip có comment giải thích

---

## 8. Quy tắc cho AI Agent

- **LUÔN** viết test cho code mới
- **LUÔN** chạy tests trước khi commit
- Test phải cover cả **happy path** và **error cases**
- Đặt tên test rõ ràng, mô tả behavior bằng tiếng Việt hoặc Anh
- **KHÔNG** viết test chỉ để tăng coverage mà không có giá trị
- **KHÔNG** disable hoặc skip test mà không có lý do
- Nếu fix bug → viết test reproduce bug trước, rồi mới fix
