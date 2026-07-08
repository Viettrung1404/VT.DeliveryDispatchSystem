# Git Workflow — Quy trình quản lý mã nguồn

> Áp dụng cho mọi thay đổi code trong dự án DeliveryDispatchSystem.

---

## 1. Branch Strategy — GitFlow

### Các nhánh chính

| Nhánh | Mục đích | Bảo vệ |
|---|---|---|
| `main` | Code production, luôn stable | ✅ Protected, chỉ merge từ `release/*` hoặc `hotfix/*` |
| `dev` | Nhánh tích hợp, chứa code mới nhất | ✅ Protected, chỉ merge qua PR |

### Các nhánh phụ

| Loại nhánh | Format | Tạo từ | Merge vào | Ví dụ |
|---|---|---|---|---|
| Feature | `feature/<tên-tính-năng>` | `dev` | `dev` | `feature/order-tracking` |
| Bugfix | `bugfix/<mô-tả-bug>` | `dev` | `dev` | `bugfix/fix-login-redirect` |
| Hotfix | `hotfix/<mô-tả>` | `main` | `main` + `dev` | `hotfix/fix-payment-crash` |
| Release | `release/<version>` | `dev` | `main` + `dev` | `release/1.2.0` |

### Quy tắc đặt tên nhánh

- Dùng **kebab-case** (chữ thường, phân tách bằng dấu `-`)
- Ngắn gọn, mô tả rõ mục đích
- Có thể thêm ticket ID: `feature/DDS-123-order-tracking`
- **KHÔNG** dùng tên cá nhân: ~~`feature/nam-fix`~~
- **KHÔNG** dùng tên chung chung: ~~`feature/update`~~, ~~`feature/fix`~~

---

## 2. Commit Conventions — Conventional Commits

### Format

```
<type>(<scope>): <mô tả ngắn>

[body - tùy chọn]

[footer - tùy chọn]
```

### Các loại commit (type)

| Type | Khi nào dùng | Ví dụ |
|---|---|---|
| `feat` | Thêm tính năng mới | `feat(order): thêm chức năng theo dõi đơn hàng` |
| `fix` | Sửa bug | `fix(auth): sửa lỗi refresh token hết hạn` |
| `docs` | Thay đổi documentation | `docs(api): cập nhật API docs cho endpoint mới` |
| `refactor` | Tái cấu trúc code (không thay đổi behavior) | `refactor(service): tách OrderService thành module` |
| `test` | Thêm hoặc sửa test | `test(order): thêm unit test cho OrderService` |
| `chore` | Cập nhật build, config, dependencies | `chore(deps): nâng version Spring Boot lên 3.2` |
| `style` | Format code (không thay đổi logic) | `style(controller): format lại code theo convention` |
| `perf` | Cải thiện performance | `perf(query): tối ưu query lấy danh sách đơn hàng` |
| `ci` | Thay đổi CI/CD config | `ci(github): thêm workflow chạy test tự động` |

### Quy tắc commit message

1. **Dòng đầu** (subject): ≤ 72 ký tự, viết bằng tiếng Việt
2. **Không** kết thúc subject bằng dấu chấm
3. **Dùng** thể mệnh lệnh: "thêm", "sửa", "xóa" (KHÔNG phải "đã thêm", "đang sửa")
4. **Body** (nếu có): giải thích **tại sao** thay đổi, không phải **thay đổi gì**
5. **Footer**: ghi ticket reference: `Refs: DDS-123` hoặc `Closes: DDS-456`
6. **Breaking change**: thêm `!` sau type hoặc ghi `BREAKING CHANGE:` trong footer

### Ví dụ commit TỐT ✅

```
feat(delivery): thêm tính năng tự động phân bổ shipper

Hệ thống sẽ tự động chọn shipper gần nhất dựa trên
vị trí GPS và tình trạng hiện tại (rảnh/bận).

Thuật toán: nearest-neighbor với weight theo rating.

Refs: DDS-234
```

### Ví dụ commit XẤU ❌

```
fix bug                              # Quá chung chung
update code                          # Không mô tả gì
feat: đã thêm tính năng mới         # Sai thể, không rõ tính năng gì
WIP                                  # Không commit WIP lên dev/main
```

---

## 3. Pull Request (PR) Process

### Trước khi tạo PR

- [ ] Code build thành công
- [ ] Tất cả tests pass
- [ ] Không có lint errors
- [ ] Đã rebase với nhánh target (thường là `dev`)
- [ ] Không có merge conflicts
- [ ] Đã self-review code

### PR Template

```markdown
## Mô tả
[Mô tả ngắn gọn thay đổi và lý do]

## Loại thay đổi
- [ ] Feature mới
- [ ] Bug fix
- [ ] Refactoring
- [ ] Documentation
- [ ] CI/CD

## Ticket liên quan
- DDS-XXX

## Checklist
- [ ] Code tuân thủ coding conventions
- [ ] Đã viết tests cho code mới
- [ ] Tất cả tests pass
- [ ] Đã cập nhật documentation (nếu cần)
- [ ] Không có breaking changes (hoặc đã ghi chú rõ)
- [ ] Đã test trên local

## Screenshots (nếu có thay đổi UI)

## Ghi chú cho Reviewer
```

### Quy tắc PR

1. **Một PR = Một mục đích**: Không trộn feature + bugfix + refactor
2. **Kích thước PR**: ≤ 400 dòng thay đổi (khuyến nghị), ≤ 800 (tối đa)
3. **Reviewers**: Tối thiểu 1 approve, khuyến nghị 2 cho core changes
4. **Merge strategy**: **Squash and Merge** cho feature → dev
5. **Không** self-merge (trừ hotfix khẩn cấp có ghi chú)
6. **Response time**: Review trong vòng **24 giờ** làm việc

---

## 4. Quy trình Release

```bash
# Tạo release branch
git checkout dev && git pull
git checkout -b release/1.2.0

# Trên release: chỉ fix bug, cập nhật version, CHANGELOG
# Merge vào main + tag
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"

# Merge ngược về dev
git checkout dev
git merge --no-ff release/1.2.0
```

---

## 5. Hotfix Process

```bash
git checkout main
git checkout -b hotfix/fix-critical-bug

# Fix xong → merge vào BOTH main và dev
git checkout main
git merge --no-ff hotfix/fix-critical-bug
git tag -a v1.2.1 -m "Hotfix: fix critical bug"

git checkout dev
git merge --no-ff hotfix/fix-critical-bug
```

---

## 6. Quy tắc cho AI Agent

- **LUÔN** tạo branch mới cho mỗi task
- **LUÔN** tuân thủ conventional commits
- **KHÔNG** push trực tiếp lên `main` hoặc `dev`
- **KHÔNG** force push
- **LUÔN** rebase trước khi tạo PR
- Mô tả PR bằng tiếng Việt, rõ ràng và đầy đủ
