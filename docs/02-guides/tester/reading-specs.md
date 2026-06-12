[📚 Docs](../../README.md) › [Guides](../README.md) › [Tester](README.md) › Đọc Spec Chain

# Đọc Và Hiểu Spec Chain

## Thứ tự đọc và mục đích

| Tài liệu | Đọc để | Ví dụ thực tế |
|---|---|---|
| **PRD** | Hiểu WHAT — business requirement | "Sai password 5 lần → khoá 30 phút" |
| **BDD** | Biết HOW VERIFIED — exact scenarios | `Given 5 failed logins, Then 423 Locked` |
| **Tech Docs BE** | Biết API contract khi test BE | `POST /api/v1/auth/login`, error codes |
| **Tech Docs Web** | Biết UI states khi test Web | Screen names, component behaviors |

## Ví dụ đọc spec chain — FT-001 Login

Sau khi chạy `/generate-spec-manifest`, tra manifest để lấy paths:

```yaml
# spec-manifest.yaml
FT-001:
  prd:          "my-project-specs/specs/prd/auth/FT-001-login.md"
  bdd:
    be:   "free-trial-be/specs/bdd/auth/system/FT-001-UC1-login-system.feature"
    web:  "free-trial-web/specs/bdd/auth/web/FT-001-UC1-login-web.feature"
  tech_docs:
    be:   "my-project-specs/specs/tech-docs/auth/FT-001-auth-api.md"
```

**Bước 1: Đọc PRD** tại `my-project-specs/specs/prd/auth/FT-001-login.md`
*(nằm trong spec submodule — shared repo của PO)*

```markdown
# AC trong PRD FT-001:
AC1: Đăng nhập thành công → truy cập được hệ thống
AC2: Sai password → hiển thị lỗi, không tiết lộ thông tin user
AC3: Sai password 5 lần liên tiếp → khoá tài khoản 30 phút
AC4: Tài khoản bị khoá → thông báo thời gian mở khoá
```

**Bước 2: Đọc BDD System (cho BE)** tại `free-trial-be/specs/bdd/auth/system/FT-001-UC1-login-system.feature`
*(nằm trong service submodule của BE — PO gen, chứa contract BE phải đáp ứng)*

```gherkin
Scenario: Successful login
  Given user "alice@example.com" exists
  When POST /api/v1/auth/login {"email": "alice@...", "password": "correct"}
  Then status 200, body has access_token and refresh_token

Scenario: Lock after 5 failures
  Given user has 0 failed attempts
  When login with wrong password 5 times
  Then 5th attempt returns 423
  And response has retry_after: 1800
```

**Bước 3: Đọc Tech Docs BE** tại `my-project-specs/specs/tech-docs/auth/FT-001-auth-api.md`
*(nằm trong spec submodule — Dev gen, SA sign-off, đây là API contract chính thức)*

```markdown
# Auth API — Error Codes
401 — sai password (failed_attempts < 5)
423 — tài khoản bị khoá (retry_after tính bằng giây)
404 — email không tồn tại (không phân biệt với 401 — security)

# Headers bắt buộc
Authorization: Bearer {token}
X-Request-ID: uuid (optional, dùng để trace)
```

**Kết quả: tester biết cần test:**
- `POST /auth/login` với đúng credentials → 200 + token
- `POST /auth/login` với sai password → 401 (message không reveal user exists/not)
- 5 lần sai → 423 + `retry_after: 1800`
- Email không tồn tại → 404 (message = 401 message — security by design)
- Sau 30 phút → có thể login lại

---

← [Workflow](workflow.md) · Tiếp theo: [Tình huống thực tế](scenarios.md)
