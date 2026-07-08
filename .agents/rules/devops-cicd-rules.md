# DevOps & CI/CD Rules — Quy tắc triển khai và vận hành

> Áp dụng cho toàn bộ pipeline từ development đến production.

---

## 1. Nguyên tắc cốt lõi

- **Infrastructure as Code** — Mọi cấu hình phải version control
- **Immutable Deployments** — Build một lần, deploy nhiều nơi
- **Environment Parity** — Dev, staging, production càng giống nhau càng tốt
- **Automate Everything** — Tự động hóa build, test, deploy
- **Rollback Ready** — Luôn có khả năng rollback nhanh

---

## 2. Docker

### Cấu trúc Docker files

```
docker/
├── backend/
│   └── Dockerfile
├── frontend/
│   └── Dockerfile
├── docker-compose.yml          # Development
├── docker-compose.prod.yml     # Production
└── .dockerignore
```

### Backend Dockerfile (Multi-stage build)

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew bootJar --no-daemon -x test

# Stage 2: Run
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Frontend Dockerfile (Multi-stage build)

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Run
FROM node:20-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

### Quy tắc Docker

- **LUÔN** dùng multi-stage build (giảm image size)
- **LUÔN** dùng non-root user
- **LUÔN** có HEALTHCHECK
- **LUÔN** pin version cụ thể cho base image (không dùng `:latest`)
- **LUÔN** có `.dockerignore` (loại node_modules, .git, build, ...)
- **KHÔNG** copy secrets vào image — dùng env variables hoặc Docker secrets

---

## 3. Docker Compose (Development)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: delivery_db
      POSTGRES_USER: delivery_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U delivery_user -d delivery_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s

  backend:
    build:
      context: ../backend
      dockerfile: ../docker/backend/Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: dev
      DATABASE_URL: jdbc:postgresql://postgres:5432/delivery_db
      REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  frontend:
    build:
      context: ../frontend
      dockerfile: ../docker/frontend/Dockerfile
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8080/api
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  postgres_data:
```

---

## 4. Environments

### Danh sách môi trường

| Môi trường | Mục đích | Database | Ai truy cập |
|---|---|---|---|
| `local` | Dev trên máy cá nhân | Docker PostgreSQL | Developer |
| `dev` | Tích hợp, test tự động | Shared dev DB | Team dev |
| `staging` | QA, UAT, demo | Clone từ production | QA team, stakeholders |
| `production` | Live | Production DB | End users |

### Environment Variables

```bash
# Chung (tất cả môi trường)
APP_NAME=DeliveryDispatchSystem
APP_PORT=8080

# Database
DATABASE_URL=jdbc:postgresql://host:5432/db_name
DATABASE_USERNAME=
DATABASE_PASSWORD=

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=
JWT_ACCESS_TOKEN_EXPIRATION=900000    # 15 phút (ms)
JWT_REFRESH_TOKEN_EXPIRATION=604800000 # 7 ngày (ms)

# OAuth2
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=

# Logging
LOG_LEVEL=INFO     # dev=DEBUG, prod=INFO
```

### Quy tắc môi trường

- Mỗi môi trường có **file config riêng**: `application-dev.yml`, `application-prod.yml`
- Secrets **KHÔNG** nằm trong file config — dùng env variables
- **KHÔNG** dùng production data trong dev/staging (nếu cần → anonymize)
- Production **LUÔN** enable: HTTPS, rate limiting, error monitoring

---

## 5. CI/CD Pipeline

### Pipeline Overview

```
Push to branch
    ↓
┌─────────────────────────────────────┐
│ 1. Build & Compile                  │
│ 2. Lint & Code Style Check          │
│ 3. Unit Tests                       │
│ 4. Integration Tests                │
│ 5. Security Scan (dependencies)     │
│ 6. Coverage Check (≥80%)            │
├─────────────────────────────────────┤
│ 7. Build Docker Image               │ ← Chỉ khi merge vào develop/main
│ 8. Push to Container Registry        │
│ 9. Deploy to staging (auto)          │
│ 10. Deploy to production (manual)    │
└─────────────────────────────────────┘
```

### GitHub Actions (Ví dụ)

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop]

jobs:
  backend-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build & Test
        run: ./gradlew build
        working-directory: backend
      - name: Coverage Check
        run: ./gradlew jacocoTestReport
        working-directory: backend

  frontend-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
        working-directory: frontend
      - run: npm run lint
        working-directory: frontend
      - run: npm run test -- --coverage
        working-directory: frontend
```

---

## 6. Monitoring & Observability

### Bắt buộc thiết lập

| Thành phần | Tool gợi ý | Mục đích |
|---|---|---|
| Health Check | Spring Boot Actuator | Kiểm tra service alive |
| Metrics | Prometheus + Grafana | CPU, memory, request latency |
| Logging | ELK Stack hoặc Loki | Centralized log |
| Error Tracking | Sentry | Track exceptions real-time |
| Uptime | UptimeRobot / Pingdom | Alert khi service down |

### Health Check Endpoints

```
GET /actuator/health        → Service health
GET /actuator/health/db     → Database connectivity
GET /actuator/health/redis  → Redis connectivity
GET /actuator/info          → App version, build info
```

---

## 7. Deployment Strategy

### Chiến lược deploy

| Chiến lược | Khi nào dùng | Rủi ro |
|---|---|---|
| **Rolling Update** | Mặc định cho hầu hết deployments | Thấp |
| **Blue-Green** | Major releases, cần rollback nhanh | Trung bình |
| **Canary** | Tính năng mới cần A/B test | Thấp |

### Rollback Checklist

- [ ] Rollback image/tag về version trước
- [ ] Verify health checks pass
- [ ] Verify critical flows hoạt động
- [ ] Check error rates trở lại bình thường
- [ ] Thông báo team về rollback và lý do

---

## 8. Quy tắc cho AI Agent

- **KHÔNG** thay đổi Dockerfile mà không giải thích lý do
- **KHÔNG** thay đổi CI/CD pipeline mà không hỏi user
- **LUÔN** dùng multi-stage build cho Docker
- **LUÔN** pin version cho base images
- **LUÔN** dùng non-root user trong container
- **KHÔNG** hardcode env-specific values — dùng environment variables
- Khi thêm service mới → cập nhật docker-compose.yml
- Khi thêm dependency mới → verify CI pipeline vẫn pass
