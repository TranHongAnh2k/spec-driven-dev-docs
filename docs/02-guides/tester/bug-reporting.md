[📚 Docs](../../README.md) › [Guides](../README.md) › [Tester](README.md) › Báo cáo bug

# Khi Tìm Thấy Bug — Quy Trình Trace

Khi test fail, báo cáo phải có đủ **spec context** để Dev fix đúng chỗ.

## Cách nhanh nhất: `/report-bug`

```bash
/report-bug FT-001 tài khoản khoá sau 6 lần sai, spec ghi 5
```

Lệnh tự động:
- Resolve spec-context từ `spec-manifest.yaml`: PRD path + version, BDD scenario fail, tech-doc
- Tìm **AC bị vi phạm** trong PRD
- **Phân loại layer** theo BUG_FLOW (Code / BDD / PRD / Design Spec / Env) → route đúng người
- Phát hiện **coverage gap** → gợi ý `/propose-scenario`
- Ghi report vào **spec repo** (`{spec_source}/feedback/bug-reports/{BUG-ID}.md`) → **commit + push**

Hoàn toàn **read-only** trên spec/code chính thức — chỉ ghi vào vùng `feedback/`.

> **Làm sao PO/Dev biết?** Report được commit+push vào **spec repo dùng chung**. PO/Dev chạy `/sync` hàng ngày sẽ thấy dòng `📥 New tester feedback`. File nằm local một mình thì không ai biết — chính bước push + `/sync` mới khép vòng. (Không có quyền push → tạo PR/MR.)

## Khi bug là "thiếu scenario" — `/propose-scenario`

Nếu `/report-bug` báo *coverage gap* (behavior đúng nhưng chưa có scenario nào kiểm):

```bash
/propose-scenario FT-001 login với email có khoảng trắng ở cuối vẫn phải thành công
```

- Nếu behavior **đã nằm trong một AC của PRD** → lệnh draft Gherkin (tag `@proposed @from-test`), ghi vào `{spec_source}/feedback/bdd-proposals/` → commit + push. PO/Dev review rồi đưa vào `.feature`.
- Nếu behavior **chưa có trong PRD** → lệnh dừng và xuất *PRD change request* cho PO.

> Tester không tự ghi vào BDD — chỉ đề xuất. Giữ đúng ownership.

## Template báo cáo bug

*(Tham khảo — `/report-bug` tự sinh theo format này. Dùng khi báo cáo thủ công.)*

```
Bug ID   : BUG-{date}-{seq}
Feature  : FT-{xxx} — {feature name}
Service  : BE / Web / App
Severity : Critical / Major / Minor

Spec context:
  PRD      : specs/prd/{domain}/{TICKET-ID}.md (v{x.x})
  BDD      : {bdd path} → Scenario: "{scenario title}"
  Tech Doc : {tech_docs path}

AC bị vi phạm:
  AC{N}: "{AC text từ PRD}"

BDD Scenario bị fail:
  "{Scenario title}"
  Given: {given state}
  When : {action}
  Then : {expected theo spec}

Actual behavior:
  {what actually happened}

Steps to reproduce:
  1. {step}
  2. {step}

Environment: staging / production
```

## Ví dụ báo cáo thực tế

```
Bug ID   : BUG-20260605-003
Feature  : FT-001 — User Login
Service  : BE
Severity : Major

Spec context:
  PRD      : specs/prd/auth/FT-001-login.md (v1.0)
  BDD      : free-trial-be/specs/bdd/auth/FT-001-login.feature
             → Scenario: "Lock account after 5 failed attempts"
  Tech Doc : free-trial-specs/specs/tech-docs/auth/FT-001-auth-api.md

AC bị vi phạm:
  AC3: "Sai password 5 lần liên tiếp → khoá tài khoản 30 phút"

BDD Scenario bị fail:
  "Lock account after 5 failed attempts"
  Given : user "alice@example.com" has 0 failed attempts
  When  : login with wrong password 5 times
  Then  : 5th attempt returns 423 Locked
          AND retry_after = 1800

Actual behavior:
  5th attempt returns 401 Unauthorized (không phải 423)
  Không có retry_after header
  → Tài khoản không bị khoá, vẫn cho thử tiếp

Steps to reproduce:
  POST /api/v1/auth/login × 5 với password sai
  → lần thứ 5 vẫn nhận 401

Environment: staging (deploy 2026-06-05 09:00)
```

## Sau khi gửi bug report

Dev sẽ xác định bug thuộc layer nào (code / BDD / PRD / env) và phản hồi với root cause + scenario cần re-test.

> Xem flow phối hợp đầy đủ giữa Tester ↔ Dev ↔ PO trong **[Operations › Bug Flow](../../04-operations/bug-flow.md)**.

> **Nếu AI lặp lại cùng kiểu lỗi qua nhiều feature:** ghi chú trong bug report. Dev chạy `/learn` để biến nó thành guardrail — AI sẽ không sinh lại lỗi đó ở các UC sau.

---

← [Tình huống thực tế](scenarios.md) · Tiếp theo: [Checklist test pass](test-checklist.md)
