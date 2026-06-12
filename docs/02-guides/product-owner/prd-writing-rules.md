[📚 Docs](../../README.md) › [Guides](../README.md) › [Product Owner](README.md) › Quy tắc viết PRD

# Quy Tắc Viết PRD Hiệu Quả

## PRD là platform-agnostic

| ✅ Viết trong PRD | ❌ Không viết trong PRD |
|---|---|
| "Đăng nhập thành công → truy cập tính năng" | "Click button Login → hiển thị spinner" |
| "Sai password 5 lần → khoá 30 phút" | "API POST /auth/login trả về JWT" |
| "Session hết hạn → yêu cầu xác thực lại" | "Toast message màu đỏ ở góc phải" |
| "User có thể reset password qua email" | "Gọi sendgrid API để gửi email" |

UI details và API specs thuộc về:
- **Design Spec** → UI/UX cho FE/App
- **Tech Docs** → API contract, technical design (dev team tự generate). BE tech-design (API contract) nằm trong shared spec module khi `spec_source` được set.

## UC và AC phải testable

Mỗi AC phải trả lời được câu hỏi: **"Làm sao biết tính năng này hoạt động đúng?"**

❌ Không testable: `AC: "Hệ thống xử lý nhanh"` · `AC: "UI thân thiện với người dùng"`

✅ Testable: `AC: "Trang login load trong vòng 3 giây"` · `AC: "Error message hiển thị trong vòng 1 giây sau khi submit"`

## BR phải có ID liên tục

```
UC1 → BR1, BR2, BR3
UC2 → BR4, BR5          ← tiếp tục từ BR3, không reset về BR1
UC3 → BR6
```

## Luôn có negative path

Với mỗi AC happy path, cần có ít nhất 1 AC cho error case:
```
AC1 (happy): Đăng nhập thành công với email + password hợp lệ
AC2 (error): Đăng nhập thất bại khi password sai → hiển thị lỗi
AC3 (error): Tài khoản bị khoá → hiển thị thông báo + thời gian còn lại
```

---

← [Tình huống thực tế](scenarios.md) · Tiếp theo: [Checklist handoff](handoff-checklist.md)
