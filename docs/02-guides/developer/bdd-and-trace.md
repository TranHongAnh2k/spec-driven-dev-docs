[📚 Docs](../../README.md) › [Guides](../README.md) › [Developer](README.md) › BDD & Trace System

# BDD & Trace System

- [Tại sao BDD quan trọng với Dev](#tại-sao-bdd-quan-trọng-với-dev)
- [Hiểu Trace System](#hiểu-trace-system)

## Tại Sao BDD Quan Trọng Với Dev

### BDD không phải "viết test thêm"

BDD là **spec thực thi được** — nó định nghĩa CHÍNH XÁC hệ thống phải làm gì trước khi viết một dòng code.
```
PRD (business language)  →  BDD (technical spec)  →  Code (implementation)
"Sai password 5 lần         Given 5 failed logins       if failCount >= 5:
 → khoá 30 phút"            Then account locked            lockAccount(30min)
                            And locked_until = now+30m
```

### BDD định hướng kiến trúc code

BDD scenario là unit of work — mỗi scenario ánh xạ thành một test case, một function, một API endpoint. Viết BDD trước buộc dev phải nghĩ về interface trước implementation.
```gherkin
# BDD này buộc dev phải tạo:
# - POST /auth/login endpoint
# - lockAccount(duration) service method
# - AccountLocked exception/response

Scenario: Lock account after 5 failed attempts
  Given user "alice@example.com" exists
  When user attempts login with wrong password 5 times
  Then account is locked for 30 minutes
  And login returns 423 Locked with "retry_after" header
```

### BDD là tài liệu sống

Khi BDD pass → code đang hoạt động đúng spec. Khi BDD fail → code lệch khỏi yêu cầu. Không cần đọc PRD để biết feature có đang hoạt động không — chạy BDD là biết ngay.

### BDD đến từ spec repo — Dev đọc, không tự gen

BDD được PO generate trong spec repo, nằm tại `specs/bdd/{domain}/`:

| Subfolder | Platform | Dev team đọc |
|---|---|---|
| `web/` | FE/Web (clicks, sees, navigates) | FE/Web dev |
| `app/` | Mobile (taps, sees screen, navigates) | App dev |
| `system/` | System/BE (request, response, business rules) | BE dev |

Cả 3 subfolder đều trace về **cùng 1 PRD**. BE không cần đọc BDD của FE và ngược lại.

## Hiểu Trace System

Framework dùng metadata `@trace.*` để liên kết PRD → BDD → Code. (Chi tiết khái niệm: [Concepts › Traceability](../../03-concepts/traceability.md).)

### Các trace fields quan trọng

| Field | Vị trí | Ý nghĩa |
|---|---|---|
| `@trace.domain` | PRD frontmatter | Domain của feature (auth, payment, ...) — dùng để route vào đúng service submodule |
| `@trace.module` | BDD / Tech Doc header | Module trong codebase sẽ implement |
| `@trace.prd` | BDD / Tech Doc header | Link về PRD gốc |
| `@trace.bdd` | Code comment / test | Link về BDD scenario |
| `@trace.status` | PRD frontmatter | `draft` / `approved` — chỉ code khi `approved` |
| `dev_selftest` | Trace TSV | `pass` / `fail` / `not_run` — kết quả dev self-check, set bởi `/dev-run-test`. Surfaced trong Living Docs để QC biết dev đã chạy self-check — **KHÔNG phải coverage chính thức** |
| `dev_selftest_at` | Trace TSV | Timestamp lần chạy `/dev-run-test` gần nhất |
| `qc_status` | Trace TSV | `pass` / `fail` / `skip` / `not_run` — kết quả **QC chính thức**, set bởi `/qc-run-test` (do QC chạy, KHÔNG phải dev). Orthogonal với `dev_selftest` và với coverage `status` |
| `qc_run_at` | Trace TSV | Timestamp lần chạy `/qc-run-test` gần nhất |

### Ví dụ trace chain hoàn chỉnh

```
specs/prd/auth/FT-001-login.md          ← @trace.domain: auth, @trace.status: approved
    ↓
specs/bdd/auth/FT-001-login.feature     ← @trace.prd: FT-001, @trace.module: AuthService
    ↓
src/auth/auth.service.ts                ← // @trace.bdd: FT-001-UC1-SC1
```

### Khi nào chạy /validate-traces?

- Sau khi refactor đổi tên file/function
- Sau khi PRD được PO cập nhật (version mới)
- Trước khi tạo PR lớn
- Khi CI báo trace validation fail
- **Sau mỗi codegen session trong umbrella mode** — để sync Living Docs panel

```
/validate-traces
→ Sẽ report: broken links, orphan BDD (không có PRD), dead code traces
```

**Lưu ý khi dùng umbrella (submodule):**
```
Vấn đề: Living Docs panel mở ở umbrella root (hoặc một service submodule đơn lẻ) → nếu không có mirror local → TRỐNG.
         TSV authoritative nằm committed trong từng service submodule: user-service/.trace/, order-service/.trace/

Giải pháp: /validate-traces (hoặc /sync) regenerate canonical trace-report.json + TSV mirror
          trong SPEC MODULE tại {spec_source}/.living-docs/ (gitignored), đồng thời ghi
          mirror local tại ./.trace của workspace hiện tại để panel không trống khi dev mở
          một service submodule đơn lẻ.

Lệnh chạy sau mỗi session:
/validate-traces
→ Reads .trace/*.tsv authoritative (committed) trong từng service: user-service/.trace/, order-service/.trace/, ...
→ Writes canonical trace-report.json + TSV mirror → {spec_source}/.living-docs/ (gitignored, regenerated bởi /sync hoặc /validate-traces)
→ Writes panel mirror → ./.trace của workspace hiện tại (non-empty khi mở single service)
→ Living Docs panel cập nhật ngay
```

> **Authoritative vs mirror:** per-service `.trace/*.tsv` vẫn được **commit** trong từng service (nguồn sự thật). `{spec_source}/.living-docs/` và `./.trace` chỉ là mirror gitignored, regenerated bởi `/sync` hoặc `/validate-traces`.

Thêm `.living-docs/` (spec module) và umbrella/workspace `.trace/` mirror vào `.gitignore`:
```
# .gitignore — spec module
.living-docs/
# .gitignore — workspace/umbrella root (mirror, không commit)
.trace/
```

---

← [Commands](commands.md) · Tiếp theo: [Workflow](workflow.md)
