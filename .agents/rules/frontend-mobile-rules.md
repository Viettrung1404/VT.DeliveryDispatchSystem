# Frontend & Mobile Rules — Quy tắc phát triển giao diện

> Áp dụng cho **Next.js 14+ (Web)** và **Flutter (Mobile)**.

---

## 1. Nguyên tắc chung

- **Component-driven** — Xây dựng từ component nhỏ → lớn
- **Separation of Concerns** — UI, logic, data access tách biệt
- **Responsive/Adaptive** — Web responsive, mobile adaptive
- **Accessibility** — Web: WCAG 2.1 AA, Mobile: platform guidelines
- **Offline-first thinking** — Mobile phải xử lý được mất mạng

---

## PHẦN A: NEXT.JS (WEB FRONTEND)

---

## 2. Cấu trúc dự án Next.js

```
frontend/
├── src/
│   ├── app/                    # App Router
│   │   ├── (auth)/             # Route group: auth pages
│   │   │   ├── login/page.tsx
│   │   │   └── register/page.tsx
│   │   ├── (dashboard)/        # Route group: authenticated pages
│   │   │   ├── layout.tsx      # Dashboard layout (sidebar, header)
│   │   │   ├── orders/
│   │   │   │   ├── page.tsx    # Danh sách đơn hàng
│   │   │   │   └── [id]/page.tsx
│   │   │   └── shippers/
│   │   ├── api/                # API routes (nếu cần BFF)
│   │   ├── layout.tsx          # Root layout
│   │   └── page.tsx            # Landing page
│   ├── components/
│   │   ├── ui/                 # Primitive: Button, Input, Modal, Badge
│   │   ├── layout/             # Header, Sidebar, Footer, PageWrapper
│   │   └── features/           # Feature-specific: OrderTable, ShipperMap
│   ├── hooks/                  # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useOrders.ts
│   │   └── useDebounce.ts
│   ├── lib/                    # Utilities
│   │   ├── api-client.ts       # Axios/fetch wrapper
│   │   ├── utils.ts            # Helper functions
│   │   └── constants.ts
│   ├── services/               # API service layer
│   │   ├── order.service.ts
│   │   ├── auth.service.ts
│   │   └── shipper.service.ts
│   ├── stores/                 # Zustand stores
│   │   ├── auth.store.ts
│   │   └── ui.store.ts
│   ├── types/                  # TypeScript types
│   │   ├── order.types.ts
│   │   ├── user.types.ts
│   │   └── api.types.ts
│   └── styles/
│       └── globals.css
├── public/
├── next.config.ts
├── tsconfig.json
└── package.json
```

---

## 3. Component Rules (Next.js)

### Phân loại components

| Loại | Thư mục | Đặc điểm | Ví dụ |
|---|---|---|---|
| **UI (Primitive)** | `components/ui/` | Không chứa business logic, tái sử dụng cao | Button, Input, Modal |
| **Layout** | `components/layout/` | Cấu trúc trang | Header, Sidebar |
| **Feature** | `components/features/` | Chứa business logic, specific cho feature | OrderTable, ShipperCard |

### Quy tắc viết component

```tsx
// ✅ Component tốt
// - Props có interface rõ ràng
// - Dùng composition thay vì prop drilling
// - Single responsibility

interface IOrderTableProps {
  orders: IOrder[];
  isLoading: boolean;
  onRowClick: (orderId: string) => void;
}

export function OrderTable({ orders, isLoading, onRowClick }: IOrderTableProps) {
  if (isLoading) return <TableSkeleton />;
  if (orders.length === 0) return <EmptyState message="Chưa có đơn hàng" />;

  return (
    <table>
      {orders.map((order) => (
        <OrderRow key={order.id} order={order} onClick={onRowClick} />
      ))}
    </table>
  );
}
```

### Quy tắc cứng

- **Một file = Một component** (trừ sub-components nhỏ)
- File name = Component name: `OrderTable.tsx` → `export function OrderTable`
- **KHÔNG** dùng default export (trừ pages)
- **KHÔNG** đặt logic fetch data trong component — dùng hooks hoặc server components
- Props interface prefix `I`: `IButtonProps`, `IOrderTableProps`
- Component ≤ 150 dòng (tách nếu lớn hơn)

---

## 4. State Management (Zustand)

### Khi nào dùng gì

| Loại state | Giải pháp | Ví dụ |
|---|---|---|
| Server state (API data) | React Query / SWR | Orders list, user profile |
| Client state (UI) | Zustand | Sidebar open/close, theme |
| Form state | React Hook Form | Login form, create order form |
| URL state | Next.js searchParams | Filters, pagination |
| Auth state | Zustand + Cookie | User session |

### Zustand store pattern

```typescript
// stores/auth.store.ts
interface IAuthState {
  user: IUser | null;
  isAuthenticated: boolean;
  login: (credentials: ILoginRequest) => Promise<void>;
  logout: () => void;
}

export const useAuthStore = create<IAuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  login: async (credentials) => {
    const response = await authService.login(credentials);
    set({ user: response.data, isAuthenticated: true });
  },
  logout: () => {
    set({ user: null, isAuthenticated: false });
  },
}));
```

---

## 5. API Integration (Next.js)

### API Client

```typescript
// lib/api-client.ts
const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
});

// Interceptor: tự động attach token
apiClient.interceptors.request.use((config) => {
  const token = getAccessToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Interceptor: tự động refresh token khi 401
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      await refreshToken();
      return apiClient(error.config);
    }
    return Promise.reject(error);
  }
);
```

### Service Layer Pattern

```typescript
// services/order.service.ts
export const orderService = {
  getAll: (params: IOrderFilterParams) =>
    apiClient.get<IApiResponse<IPaginatedData<IOrder>>>('/orders', { params }),

  getById: (id: string) =>
    apiClient.get<IApiResponse<IOrder>>(`/orders/${id}`),

  create: (data: ICreateOrderRequest) =>
    apiClient.post<IApiResponse<IOrder>>('/orders', data),

  updateStatus: (id: string, status: OrderStatus) =>
    apiClient.patch<IApiResponse<IOrder>>(`/orders/${id}`, { status }),
};
```

---

## PHẦN B: FLUTTER (MOBILE)

---

## 6. Cấu trúc dự án Flutter

```
mobile/
└── lib/
    ├── core/                      # Shared across features
    │   ├── constants/
    │   │   ├── app_colors.dart
    │   │   ├── app_text_styles.dart
    │   │   └── api_endpoints.dart
    │   ├── theme/
    │   │   └── app_theme.dart
    │   ├── utils/
    │   │   ├── date_formatter.dart
    │   │   └── validators.dart
    │   └── widgets/               # Shared widgets
    │       ├── app_button.dart
    │       ├── app_text_field.dart
    │       └── loading_overlay.dart
    ├── data/                      # Data layer
    │   ├── datasources/
    │   │   ├── remote/
    │   │   │   └── order_remote_datasource.dart
    │   │   └── local/
    │   │       └── order_local_datasource.dart
    │   ├── models/                # API response models (JSON)
    │   │   ├── order_model.dart
    │   │   └── user_model.dart
    │   └── repositories/         # Repository implementations
    │       └── order_repository_impl.dart
    ├── domain/                    # Domain layer (pure Dart)
    │   ├── entities/
    │   │   ├── order.dart
    │   │   └── user.dart
    │   ├── repositories/         # Repository interfaces
    │   │   └── order_repository.dart
    │   └── usecases/
    │       ├── get_orders.dart
    │       └── create_order.dart
    ├── presentation/             # UI layer
    │   ├── screens/
    │   │   ├── home/
    │   │   │   ├── home_screen.dart
    │   │   │   └── widgets/
    │   │   └── order/
    │   │       ├── order_list_screen.dart
    │   │       ├── order_detail_screen.dart
    │   │       └── widgets/
    │   ├── blocs/                # BLoC / Cubit
    │   │   ├── order/
    │   │   │   ├── order_bloc.dart
    │   │   │   ├── order_event.dart
    │   │   │   └── order_state.dart
    │   │   └── auth/
    │   └── routes/
    │       └── app_router.dart
    ├── di/                       # Dependency Injection
    │   └── injection_container.dart
    └── main.dart
```

---

## 7. Architecture — Clean Architecture (Flutter)

```
Presentation → Domain ← Data
    ↓              ↑         ↓
  BLoC/Cubit    UseCases   Repository Impl
    ↓              ↑         ↓
  Widgets      Entities   DataSources (API/Local)
```

### Quy tắc layer

| Layer | Biết layer nào | Chứa gì | Dependencies |
|---|---|---|---|
| **Domain** | Không biết ai | Entities, Repository interfaces, UseCases | Không có (pure Dart) |
| **Data** | Domain | Models, Repository impls, DataSources | Domain, HTTP client |
| **Presentation** | Domain | Screens, Widgets, BLoC/Cubit | Domain, DI |

### State Management — BLoC/Cubit

```dart
// Dùng Cubit cho logic đơn giản
class OrderListCubit extends Cubit<OrderListState> {
  final GetOrders getOrders;

  OrderListCubit(this.getOrders) : super(OrderListInitial());

  Future<void> fetchOrders() async {
    emit(OrderListLoading());
    final result = await getOrders();
    result.fold(
      (failure) => emit(OrderListError(failure.message)),
      (orders) => emit(OrderListLoaded(orders)),
    );
  }
}

// Dùng BLoC cho logic phức tạp (nhiều events)
class OrderDetailBloc extends Bloc<OrderDetailEvent, OrderDetailState> { ... }
```

---

## 8. Mobile-Specific Rules

### Error Handling

```dart
// Dùng Either pattern (dartz hoặc fpdart)
Future<Either<Failure, List<Order>>> getOrders() async {
  try {
    final response = await remoteDataSource.getOrders();
    return Right(response.map((m) => m.toEntity()).toList());
  } on ServerException catch (e) {
    return Left(ServerFailure(e.message));
  } on NetworkException {
    return Left(NetworkFailure('Không có kết nối mạng'));
  }
}
```

### Offline Support

- Cache API responses với `shared_preferences` hoặc `hive`
- Hiển thị cached data khi offline
- Queue actions khi offline, sync khi có mạng
- Hiển thị indicator: online/offline status

### Performance

- Dùng `const` constructors cho widgets không thay đổi
- Dùng `ListView.builder` cho danh sách dài (lazy loading)
- Image caching với `cached_network_image`
- Tránh rebuild không cần thiết (dùng `BlocSelector`, `Selector`)

---

## 9. Quy tắc cho AI Agent

### Chung

- **LUÔN** tách component/widget nhỏ, tái sử dụng được
- **KHÔNG** đặt business logic trong UI component/widget
- **LUÔN** handle loading, error, empty states
- **LUÔN** responsive (web) / adaptive (mobile)

### Next.js

- Ưu tiên **Server Components** khi có thể
- Dùng **'use client'** chỉ khi cần interactivity
- **KHÔNG** fetch data trong client component — dùng server component hoặc hooks

### Flutter

- **LUÔN** tuân thủ Clean Architecture layers
- **KHÔNG** import data layer trong presentation layer
- **LUÔN** dùng dependency injection (get_it)
- **LUÔN** handle trạng thái mạng (online/offline)
