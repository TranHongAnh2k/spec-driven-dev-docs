[📚 Docs](../../README.md) › [Guides](../README.md) › Product Owner / BA

# Hướng Dẫn PO/BA — Spec-Driven Dev

Tài liệu dành cho **Product Owner (PO)** và **Business Analyst (BA)** — vai trò, commands, các tình huống thực tế, và quy tắc viết PRD.

## Mục Lục

| Trang | Nội dung |
|---|---|
| [Commands](commands.md) | Bảng lệnh cho PO/BA · project lessons · xử lý feedback tester |
| [Tình huống thực tế](scenarios.md) | 11 scenario: tính năng mới, design spec, BDD, PRD thay đổi, conflict, brownfield, ... |
| [Quy tắc viết PRD](prd-writing-rules.md) | Platform-agnostic · testable AC · negative path · BR numbering |
| [Checklist handoff](handoff-checklist.md) | Checklist verify trước khi thông báo dev team bắt đầu |

## Vai Trò PO/BA Trong Framework

PO/BA là người duy nhất viết và approve tài liệu đầu vào + BDD:

```
PO/BA                     Dev team
──────────────────────    ──────────────────────────────
/define-product           đọc product-definition + BDD
/generate-prd        →    /review-context (xác nhận PRD)
/refine-prd (SA review)   /generate-tech-docs
/review-context           /generate-code + tests
/generate-design-spec →   /review-code / /dev-run-test
/generate-bdd (web)
/generate-bdd (app)
/generate-bdd (system)
```

**PO/BA chịu trách nhiệm:**
- Đảm bảo PRD platform-agnostic (không chứa UI details hay API specs)
- Đặt `@trace.domain` đúng để dev team routing hoạt động
- Cập nhật `@trace.status: approved` trước khi handoff
- **Generate BDD cho tất cả platforms** (web, app, system) trong spec repo
- Thông báo dev team khi có PRD hoặc BDD mới/update

**PO/BA KHÔNG cần quan tâm:**
- Tech docs — dev team tự generate
- Source code — hoàn toàn dev team

### Tại sao BDD thuộc trách nhiệm của PO?

Đặt BDD trong spec repo giúp **tổng hợp toàn bộ tài liệu nghiệp vụ về một chỗ**.

```
PRD (WHAT + WHY)  →  BDD (HOW verified)  →  Code (HOW built)
   ↑ PO/BA            ↑ PO/BA              ↑ Dev
   spec repo          spec repo             dev repo
```

> **Sau khi BDD approved → QC chạy pipeline tự động:** Khi BDD của một UC được approve, QC team chạy bộ lệnh native `/qc-analyze → /qc-plan → /qc-design-test → /qc-review → /qc-run-test → /qc-report` — đọc official `.feature`, generate + run Playwright/pytest, rồi ghi `qc_status` (official QC coverage) vào Living Docs. PO không chạy QC, nhưng nên biết bước này tồn tại ở downstream. Chi tiết: [QC Automation Guide](../qc-automation.md).

**3 lý do chính BDD thuộc PO:**

**1. Tổng hợp tài liệu nghiệp vụ tại một nơi** — Khi BDD nằm trong spec repo cùng PRD, mọi spec nghiệp vụ đều ở một chỗ — dễ review, audit, và generate báo cáo. Dev team đọc spec từ submodule, không cần tự suy ra scenarios.

**2. BDD viết ở mức nghiệp vụ — PO hiểu được** — BDD trong spec repo mô tả **hành vi hệ thống** bằng ngôn ngữ nghiệp vụ, không phải implementation detail:

| Platform | BDD trong spec repo (PO viết) | Test code (Dev viết) |
|---|---|---|
| Web | `When user submits login → sees dashboard` | `await page.click('#login-btn')` |
| App | `When user taps Login → navigates to Home` | `element(by.id('login')).tap()` |
| System | `When login request received → returns auth token` | `POST /api/auth/login assertions` |

**3. System BDD là nguồn sự thật cho BE** — tổng hợp từ FE + App BDD:
- FE BDD: "User expects auth token để truy cập tiếp"
- App BDD: "App expects user profile sau khi đăng nhập"
- System BDD: "BE phải trả `{ token, user_profile }` để thỏa mãn cả 2 platform"

> Xem [Concepts › Traceability](../../03-concepts/traceability.md) để hiểu trace chain PRD → BDD → Code.

---

*Xem thêm:* [Developer Guide](../developer/README.md) · [Tester Guide](../tester/README.md) · [QC Automation Guide](../qc-automation.md) · [Reference › Commands](../../05-reference/commands.md)
