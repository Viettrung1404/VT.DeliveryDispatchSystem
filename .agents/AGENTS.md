# DeliveryDispatchSystem — Quy tắc dự án

> Tài liệu này định nghĩa các quy tắc chung áp dụng cho **toàn bộ dự án**. Mọi AI agent và developer đều **BẮT BUỘC** tuân thủ.

---

## 1. Tổng quan dự án

- **Tên**: DeliveryDispatchSystem — Hệ thống quản lý điều vận giao hàng
- **Kiến trúc**: Monolithic modular (có thể tách microservices sau)
- **Team**: 5–10 người, quy trình chặt chẽ

### Tech Stack

| Thành phần | Công nghệ |
|---|---|
| Backend | Java 17+ / Spring Boot 3.x |
| Web Frontend | Next.js 14+ (React, TypeScript) |
| Mobile | Flutter (Dart) |
| Database | PostgreSQL 15+ |
| Cache | Redis 7+ |
| Auth | JWT + Spring Security + OAuth2 |
| Container | Docker + Docker Compose |

---

## 2. Quy tắc ngôn ngữ

- **Code**: Tiếng Anh (biến, hàm, class, comment trong code)
- **Documentation, rules, commit message, PR**: Tiếng Việt
- **API response messages**: Tiếng Anh (i18n-ready)
- **Tên file, thư mục**: Tiếng Anh, dùng `kebab-case` hoặc `PascalCase` tùy convention của từng tech

---

## 3. Cấu trúc dự án

```
DeliveryDispatchSystem/
├── .agents/                    # Rules & workflows cho AI agents
│   ├── AGENTS.md               # File này
│   ├── rules/                  # Quy tắc chi tiết cho từng lĩnh vực
│   └── skills/                 # Các skill chuyên biệt (brainstorming, ...)
├── backend/                    # Java Spring Boot
│   ├── src/main/java/
│   │   └── com/delivery/
│   │       ├── config/         # Cấu hình (Security, Redis, CORS, ...)
│   │       ├── controller/     # REST controllers
│   │       ├── service/        # Business logic
│   │       ├── repository/     # Data access (JPA)
│   │       ├── model/          # Entity classes
│   │       ├── dto/            # Data Transfer Objects
│   │       ├── mapper/         # DTO <-> Entity mappers
│   │       ├── exception/      # Custom exceptions & handlers
│   │       ├── security/       # Auth & authorization
│   │       ├── util/           # Utility classes
│   │       └── constant/       # Constants & enums
│   ├── src/main/resources/
│   │   ├── application.yml
│   │   ├── application-dev.yml
│   │   ├── application-prod.yml
│   │   └── db/migration/       # Flyway migrations
│   └── src/test/
├── frontend/                   # Next.js web app
│   ├── src/
│   │   ├── app/                # App Router pages
│   │   ├── components/         # Reusable UI components
│   │   │   ├── ui/             # Primitive components (Button, Input, ...)
│   │   │   ├── layout/         # Layout components (Header, Sidebar, ...)
│   │   │   └── features/       # Feature-specific components
│   │   ├── hooks/              # Custom React hooks
│   │   ├── lib/                # Utilities, API client
│   │   ├── services/           # API service layer
│   │   ├── stores/             # State management (Zustand)
│   │   ├── types/              # TypeScript types/interfaces
│   │   └── styles/             # Global styles
│   └── public/
├── mobile/                     # Flutter app
│   └── lib/
│       ├── core/               # Core utilities, theme, constants
│       ├── data/               # Data layer (repositories, data sources)
│       ├── domain/             # Domain layer (entities, use cases)
│       ├── presentation/       # UI layer (screens, widgets, controllers)
│       └── di/                 # Dependency injection
├── docs/                       # Documentation
│   ├── adr/                    # Architecture Decision Records
│   ├── api/                    # API documentation
│   └── guides/                 # Hướng dẫn setup, deploy
├── docker/                     # Docker configs
│   ├── backend/Dockerfile
│   ├── frontend/Dockerfile
│   └── docker-compose.yml
└── scripts/                    # Build & deploy scripts
```

---

## 4. Quy tắc coding chung

### 4.1. Naming Conventions

| Ngữ cảnh | Convention | Ví dụ |
|---|---|---|
| Java class | PascalCase | `OrderService`, `DeliveryDto` |
| Java method/field | camelCase | `findByStatus()`, `deliveryDate` |
| Java constant | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Java package | lowercase | `com.delivery.service` |
| TypeScript component | PascalCase | `OrderTable.tsx` |
| TypeScript function/var | camelCase | `useOrderList()` |
| TypeScript type/interface | PascalCase + prefix `I` cho interface | `IOrderResponse` |
| Dart class | PascalCase | `OrderScreen` |
| Dart var/function | camelCase | `fetchOrders()` |
| Dart file | snake_case | `order_screen.dart` |
| Database table | snake_case, số nhiều | `delivery_orders` |
| Database column | snake_case | `created_at`, `order_status` |
| API endpoint | kebab-case, số nhiều | `/api/v1/delivery-orders` |
| Env variable | UPPER_SNAKE_CASE | `DATABASE_URL` |

### 4.2. Nguyên tắc bất di bất dịch

1. **DRY** — Don't Repeat Yourself. Tái sử dụng code thông qua abstraction.
2. **SOLID** — Áp dụng cho backend Java. Đặc biệt Single Responsibility và Dependency Inversion.
3. **YAGNI** — Không viết code cho tính năng chưa cần.
4. **KISS** — Ưu tiên giải pháp đơn giản, dễ hiểu.
5. **Fail Fast** — Validate input sớm, throw exception rõ ràng.

### 4.3. Giới hạn kích thước

| Đối tượng | Giới hạn |
|---|---|
| Một method/function | ≤ 30 dòng (khuyến nghị), ≤ 50 dòng (tối đa) |
| Một class/file | ≤ 300 dòng (khuyến nghị), ≤ 500 dòng (tối đa) |
| Parameters của method | ≤ 4 params (nếu nhiều hơn → dùng object/DTO) |
| Độ sâu nesting | ≤ 3 levels (if/for/while lồng nhau) |

### 4.4. Error Handling

- **KHÔNG BAO GIỜ** nuốt exception (catch rồi không làm gì)
- **KHÔNG** dùng exception cho control flow
- Backend: Dùng custom exception + global exception handler (`@RestControllerAdvice`)
- Frontend: Dùng error boundary + try-catch cho async calls
- Mobile: Dùng `Result<T>` pattern hoặc Either type

### 4.5. Logging

- **Dùng SLF4J** (backend), **console abstraction** (frontend)
- Log levels: `ERROR` > `WARN` > `INFO` > `DEBUG` > `TRACE`
- **BẮT BUỘC** log: request/response API, exception, business-critical events
- **CẤM** log: passwords, tokens, PII (Personally Identifiable Information)
- Format: `[timestamp] [level] [traceId] [class] — message`

---

## 5. Quy tắc Git

- Xem chi tiết tại: [rules/git-workflow.md](rules/git-workflow.md)
- Branch strategy: **GitFlow** (main, develop, feature/*, hotfix/*, release/*)
- Commit convention: **Conventional Commits** (feat, fix, docs, refactor, test, chore)

---

## 6. Quy tắc Code Review

- Xem chi tiết tại: [rules/code-review-rules.md](rules/code-review-rules.md)
- Mọi PR **BẮT BUỘC** có ít nhất 1 reviewer approve
- Không self-merge

---

## 7. Quy tắc Testing

- Xem chi tiết tại: [rules/testing-rules.md](rules/testing-rules.md)
- Coverage tối thiểu: **80%** cho business logic
- Mọi PR phải pass tất cả tests

---

## 8. Quy tắc bảo mật

- Xem chi tiết tại: [rules/security-rules.md](rules/security-rules.md)
- **KHÔNG BAO GIỜ** commit secrets, API keys, passwords vào repo
- Dùng `.env` + `.gitignore`

---

## 9. Workflow cho AI Agent

Khi AI agent nhận task, **BẮT BUỘC** tuân thủ thứ tự:

1. **Đọc AGENTS.md** và các skill liên quan trước khi bắt đầu
2. **Hiểu context** — đọc code hiện tại, hiểu kiến trúc
3. **Lên kế hoạch** — tạo plan trước khi code (với task phức tạp)
4. **Implement** — viết code theo đúng conventions
5. **Test** — chạy test, đảm bảo không break existing code
6. **Review** — tự review code trước khi submit

### Quy tắc cho AI Agent

- **KHÔNG** tự ý thay đổi kiến trúc mà không hỏi user
- **KHÔNG** xóa code/comment hiện có mà không có lý do
- **KHÔNG** thêm dependency mới mà không giải thích
- **LUÔN** giữ backward compatibility khi sửa API
- **LUÔN** viết test cho code mới
- **LUÔN** cập nhật documentation khi thay đổi logic

---

## 10. Tham chiếu Rules

Tất cả quy tắc chi tiết nằm trong thư mục `.agents/rules/`:

| File | Mô tả |
|---|---|
| [git-workflow.md](rules/git-workflow.md) | Quy trình Git, branch, commit, PR |
| [code-review-rules.md](rules/code-review-rules.md) | Checklist review, quality gates |
| [testing-rules.md](rules/testing-rules.md) | Unit/Integration test, coverage |
| [security-rules.md](rules/security-rules.md) | Auth, secrets, validation |
| [api-design-rules.md](rules/api-design-rules.md) | REST conventions, error handling |
| [frontend-mobile-rules.md](rules/frontend-mobile-rules.md) | Next.js + Flutter patterns |
| [backend-rules.md](rules/backend-rules.md) | Spring Boot architecture |
| [devops-cicd-rules.md](rules/devops-cicd-rules.md) | Docker, CI/CD, deployment |
| [documentation-rules.md](rules/documentation-rules.md) | ADR, README, API docs |
