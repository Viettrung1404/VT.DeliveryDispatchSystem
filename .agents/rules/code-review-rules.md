# Code Review Rules — Quy tắc đánh giá mã nguồn

> Mọi thay đổi code **BẮT BUỘC** phải qua review trước khi merge. Không có ngoại lệ.

---

## 1. Nguyên tắc cốt lõi

- Code review là **bắt buộc**, không phải khuyến nghị
- Mục tiêu: nâng cao **chất lượng code** và **chia sẻ kiến thức** trong team
- Review **code**, không review **người viết**
- Comment mang tính **xây dựng**, kèm **đề xuất cụ thể**
- Không approve nếu còn **blocking issue**

---

## 2. Phân loại Review Comments

| Prefix | Ý nghĩa | Bắt buộc sửa? |
|---|---|---|
| `[BLOCKER]` | Lỗi nghiêm trọng, vi phạm security/logic | ✅ Bắt buộc |
| `[MAJOR]` | Vấn đề quan trọng về design/performance | ✅ Bắt buộc |
| `[MINOR]` | Naming, formatting, code style | ⚠️ Nên sửa |
| `[NIT]` | Góp ý nhỏ, preference cá nhân | ❌ Tùy chọn |
| `[QUESTION]` | Hỏi để hiểu rõ hơn, không phải yêu cầu sửa | ❌ Trả lời |
| `[PRAISE]` | Khen code tốt, giải pháp hay | ❌ Động viên |

---

## 3. Checklist Review

### 3.1. Correctness (Tính đúng đắn)

- [ ] Logic business đúng theo requirements/ticket
- [ ] Edge cases được xử lý (null, empty, boundary values)
- [ ] Error handling đầy đủ và hợp lý
- [ ] Không có race conditions hoặc deadlocks
- [ ] Data validation ở đúng layer

### 3.2. Security

- [ ] Input validation đầy đủ (SQL injection, XSS)
- [ ] Authentication/Authorization đúng
- [ ] Không log sensitive data (password, token, PII)
- [ ] Không hardcode secrets, API keys
- [ ] Endpoints mới có đúng permission check

### 3.3. Performance

- [ ] Không có N+1 query
- [ ] Không load dữ liệu không cần thiết
- [ ] Database queries có index phù hợp
- [ ] Không có memory leak tiềm ẩn
- [ ] Pagination cho list API

### 3.4. Code Quality

- [ ] Tuân thủ naming conventions (xem AGENTS.md)
- [ ] Method ≤ 50 dòng, class ≤ 500 dòng
- [ ] Nesting ≤ 3 levels
- [ ] Không có code duplicate
- [ ] Single Responsibility — mỗi class/method làm một việc
- [ ] Không có magic numbers/strings — dùng constants

### 3.5. Testing

- [ ] Unit tests cho business logic mới
- [ ] Test coverage ≥ 80% cho code mới
- [ ] Test cả happy path và error cases
- [ ] Test names rõ ràng, mô tả đúng behavior
- [ ] Không có test bị skip mà không có lý do

### 3.6. Documentation

- [ ] Public API có Javadoc/JSDoc đầy đủ
- [ ] Complex logic có comment giải thích **tại sao**
- [ ] README/docs được cập nhật nếu có thay đổi behavior
- [ ] API docs (Swagger/OpenAPI) được cập nhật

---

## 4. Quy trình Review

### Bước 1: Author tự review

Trước khi request review, author **BẮT BUỘC**:
1. Đọc lại diff từ đầu đến cuối
2. Chạy checklist ở mục 3
3. Thêm description cho PR (xem git-workflow.md)
4. Highlight điểm cần reviewer chú ý

### Bước 2: Assign Reviewer

- Tối thiểu **1 reviewer** (bắt buộc)
- **2 reviewers** cho: core logic, security, API changes, database migrations
- Reviewer nên là người **hiểu context** của module đang thay đổi

### Bước 3: Review

- Reviewer đọc description trước, hiểu mục đích
- Review từng file, để comment theo prefix classification
- **Approve** nếu không có BLOCKER/MAJOR
- **Request changes** nếu có BLOCKER/MAJOR

### Bước 4: Author respond

- Address mọi BLOCKER và MAJOR
- Reply giải thích nếu không đồng ý (với lý do cụ thể)
- Push fixes, **không force push** (để reviewer thấy diff)
- Re-request review sau khi sửa

### Bước 5: Final Approval → Merge

- Reviewer re-review, approve nếu OK
- Author merge (Squash and Merge)
- Xóa feature branch sau khi merge

---

## 5. Quality Gates (Tự động)

Các checks **BẮT BUỘC** pass trước khi merge:

| Gate | Tool | Quy tắc |
|---|---|---|
| Build | CI/CD | Build phải thành công |
| Unit Tests | JUnit / Jest / Flutter Test | Tất cả tests pass |
| Lint | Checkstyle / ESLint / Dart Analyzer | Không có errors |
| Coverage | JaCoCo / Istanbul / lcov | ≥ 80% cho code mới |
| Security Scan | Dependabot / Snyk | Không có critical vulnerabilities |

---

## 6. Thời gian Review

| Loại PR | Thời gian tối đa |
|---|---|
| Hotfix | 2 giờ |
| Bugfix nhỏ (< 100 dòng) | 4 giờ |
| Feature thường | 24 giờ |
| Core/Architecture changes | 48 giờ |

---

## 7. Quy tắc cho AI Agent

- **LUÔN** tự review code trước khi đề xuất
- **LUÔN** chạy qua checklist ở mục 3 cho code mình viết
- Nếu phát hiện vấn đề trong code hiện có → tạo comment/issue riêng, **KHÔNG** sửa trong PR hiện tại
- **KHÔNG** approve PR mà không đọc kỹ
- Cung cấp review comments theo đúng format prefix
