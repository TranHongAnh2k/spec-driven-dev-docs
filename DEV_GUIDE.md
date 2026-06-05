# Hướng Dẫn Developer — Spec-Driven Dev

Tài liệu này dành riêng cho **Developer (FE / BE / App)**.  
Mô tả vai trò, workflow, và các tình huống thực tế khi làm việc trong framework spec-driven-dev.

---

## Mục Lục

1. [Vai trò Dev trong framework](#1-vai-trò-dev-trong-framework)
2. [Commands dành cho Dev](#2-commands-dành-cho-dev)
3. [Tại sao BDD quan trọng với Dev](#3-tại-sao-bdd-quan-trọng-với-dev)
4. [Hiểu Trace System](#4-hiểu-trace-system)
5. [Workflow cơ bản](#5-workflow-cơ-bản)
6. [Tình huống thực tế](#6-tình-huống-thực-tế)

---

## 1. Vai Trò Dev Trong Framework

```
PO/BA                     Dev
──────────────────────    ──────────────────────────────────────
/define-product           /review-context (đọc PRD trước khi bắt đầu)
/generate-prd        →    /generate-bdd      (từ PRD → BDD)
/refine-prd               /generate-tech-docs (từ PRD + BDD → Tech Docs)
/review-context           /generate-code     (từ BDD + Tech Docs → Code)
/generate-design-spec →   /generate-bdd (FE/App: đọc cả Design Spec)
                          /review-code
                          /generate-tests
                          /run-tests
                          /fix-bug / /debug
                          /validate-traces
```

**Dev chịu trách nhiệm:**
- Đọc và hiểu PRD trước khi generate bất kỳ thứ gì
- Generate BDD đúng với platform của mình (BE/FE/App có BDD khác nhau)
- Đảm bảo code trace về đúng BDD, BDD trace về đúng PRD
- Báo PO/BA khi PRD có gì không rõ hoặc mâu thuẫn — không tự suy diễn

**Dev KHÔNG làm:**
- Viết/sửa PRD — đó là việc của PO/BA
- Viết/sửa Design Spec — đó là việc của PO/BA + Designer
- Approve PRD — chỉ PO mới có quyền này

---

## 2. Commands Dành Cho Dev

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/review-context {prd-file}` | Đọc + xác nhận PRD đủ rõ trước khi code | **Bước đầu tiên** khi nhận PRD mới |
| `/generate-bdd {prd-file}` | Sinh BDD Gherkin từ PRD | Sau khi review-context pass |
| `/generate-tech-docs {prd-file}` | Sinh Tech Docs (API, DB schema, sequence diagram) | Sau khi BDD được review |
| `/generate-code {bdd-file}` | Sinh code skeleton từ BDD + Tech Docs | Sau khi tech docs sẵn sàng |
| `/generate-tests {bdd-file}` | Sinh test cases từ BDD | Song song hoặc sau generate-code |
| `/review-code {file}` | Review code theo 4 lens (Security/Perf/Arch/Test) | Trước khi tạo PR |
| `/review-tech-docs {tech-doc-file}` | Review chất lượng Tech Docs | Sau generate-tech-docs |
| `/run-tests` | Chạy test suite hiện tại | Sau khi code + tests sẵn sàng |
| `/fix-bug {issue}` | Phân tích + fix bug có trace | Khi có bug report |
| `/debug {symptom}` | Debug vấn đề chưa rõ nguyên nhân | Khi cần trace root cause |
| `/smoke-test` | Kiểm tra nhanh các luồng chính | Sau deploy hoặc merge lớn |
| `/validate-traces` | Kiểm tra toàn bộ trace chain còn hợp lệ | Sau refactor hoặc khi PRD update |

---

## 3. Tại Sao BDD Quan Trọng Với Dev

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

### BDD khác nhau theo platform từ cùng 1 PRD

| Platform | BDD focus | Ví dụ scenario |
|---|---|---|
| **BE** | API contracts, business logic, data integrity | `POST /auth/login → 200 OK + JWT` |
| **FE/Web** | User interactions, UI state, Playwright flows | `Click Login → see dashboard` |
| **App** | Mobile gestures, screen transitions, offline behavior | `Tap Login → navigate to HomeScreen` |

Cả 3 đều trace về **cùng 1 PRD**. BE không cần đọc BDD của FE và ngược lại.

---

## 4. Hiểu Trace System

Framework dùng metadata `@trace.*` để liên kết PRD → BDD → Code.

### Các trace fields quan trọng

| Field | Vị trí | Ý nghĩa |
|---|---|---|
| `@trace.domain` | PRD frontmatter | Domain của feature (auth, payment, ...) — dùng để route vào đúng service submodule |
| `@trace.module` | BDD / Tech Doc header | Module trong codebase sẽ implement |
| `@trace.prd` | BDD / Tech Doc header | Link về PRD gốc |
| `@trace.bdd` | Code comment / test | Link về BDD scenario |
| `@trace.status` | PRD frontmatter | `draft` / `approved` — chỉ code khi `approved` |

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

```
/validate-traces
→ Sẽ report: broken links, orphan BDD (không có PRD), dead code traces
```

---

## 5. Workflow Cơ Bản

```
Nhận thông báo PRD mới từ PO
        │
        ▼
/review-context {prd-file}
→ Kiểm tra @trace.domain, @trace.status = approved
→ Đọc hiểu AC, UC, BR
→ Nếu có gì không rõ: hỏi PO TRƯỚC khi tiếp tục
        │
        ▼
/generate-bdd {prd-file}
→ Gen Gherkin scenarios cho platform của mình
→ Review: đủ happy path? edge case? error path?
        │
        ▼
/generate-tech-docs {prd-file}
→ Gen API spec, DB schema, sequence diagram
→ /review-tech-docs để verify chất lượng
        │
        ▼
/generate-code {bdd-file}
→ Gen code skeleton theo BDD
→ Implement logic
→ Đảm bảo @trace.bdd comment trong code
        │
        ▼
/generate-tests {bdd-file}
→ Gen unit test + integration test
→ /run-tests để verify
        │
        ▼
/review-code {files}
→ 4 lens: Security / Performance / Architecture / Test Coverage
→ Fix issues trước khi tạo PR
        │
        ▼
Tạo PR → notify PO/SA review
```

---

## 6. Tình Huống Thực Tế

---

### Tình huống 1: Nhận PRD mới và bắt đầu work

**Bối cảnh:** PO thông báo PRD `FT-042-checkout.md` đã approved.

**Các bước:**

```
1. /review-context specs/prd/payment/FT-042-checkout.md
   → Kiểm tra @trace.status = approved (không code khi còn draft)
   → Đọc kỹ AC, UC, BR — ghi chú gì chưa rõ

2. Nếu có thắc mắc → hỏi PO ngay, không tự suy diễn
   Ví dụ: "BR5 nói 'kiểm tra giới hạn thanh toán' — giới hạn này config ở đâu?
           Trong DB hay hardcode? Có khác nhau theo tier user không?"

3. Sau khi rõ ràng:
   /generate-bdd specs/prd/payment/FT-042-checkout.md

4. Review BDD output:
   - Đủ happy path?
   - Edge cases (giỏ hàng rỗng, hết hàng, timeout payment)?
   - Error paths (card declined, network error)?
```

**Lưu ý:** Nếu `/review-context` báo P0 warning (domain không match config) → **dừng lại**, báo PO/DevOps fix config trước.

---

### Tình huống 2: Generate BDD cho BE (API-centric)

**Bối cảnh:** BE dev nhận PRD về tính năng đăng nhập.

**BDD BE tập trung vào:**
- HTTP request/response contracts
- Business rule enforcement tại API layer
- Data validation, error codes
- Database state changes

```
/generate-bdd specs/prd/auth/FT-001-login.md
→ Agent sẽ hỏi: "Platform? (BE/FE/App)"
→ Chọn BE

Output: specs/bdd/auth/FT-001-login.feature
```

```gherkin
# Ví dụ BDD BE được gen ra
Feature: User Authentication
  @trace.prd: FT-001
  @trace.module: AuthService

  Scenario: Successful login returns JWT token
    Given user "alice@example.com" exists with valid credentials
    When POST /api/v1/auth/login with {"email": "...", "password": "..."}
    Then response status is 200
    And response body contains "access_token" and "refresh_token"
    And token expires_in is 3600

  Scenario: Failed login increments attempt counter
    Given user has 0 failed attempts
    When POST /api/v1/auth/login with wrong password
    Then response status is 401
    And failed_attempts increases by 1
```

---

### Tình huống 3: Generate BDD cho FE/App (UI-centric)

**Bối cảnh:** FE dev nhận PRD + Design Spec về cùng tính năng đăng nhập.

**FE/App cần đọc cả 2:**
- PRD → business rules, AC
- Design Spec → screen names, component names, user flow

```
/generate-bdd specs/prd/auth/FT-001-login.md
→ Platform: FE (Web)
→ Agent tự đọc Design Spec nếu có link trong PRD

Output: specs/bdd/auth/FT-001-login-web.feature
```

```gherkin
# BDD FE/Web — khác hoàn toàn với BE
Scenario: User sees error message after wrong password
  Given user is on LoginScreen
  When user enters wrong password and taps "Đăng nhập"
  Then error toast "Sai mật khẩu" appears
  And password field is cleared
  And "Quên mật khẩu?" link is highlighted

Scenario: Account locked — show countdown
  Given user has entered wrong password 5 times
  Then LoginScreen shows "Tài khoản bị khoá. Thử lại sau 29:45"
  And countdown timer decrements every second
```

---

### Tình huống 4: PRD thay đổi mid-sprint

**Bối cảnh:** PO update PRD `FT-042` từ v1.0 → v1.1 khi dev đang code.

**Quy trình:**

```
PO notify: "FT-042 updated — BR7 thay đổi giới hạn từ 5tr → 10tr"
        │
        ▼
Dev chạy:
/review-context specs/prd/payment/FT-042-checkout.md
→ Xem diff từ v1.0 sang v1.1 (agent highlight thay đổi)

        │
        ▼
Đánh giá impact:
- BDD bị ảnh hưởng? → update BDD scenarios liên quan
- Tech Docs bị ảnh hưởng? → update API spec
- Code bị ảnh hưởng? → update logic + tests

        │
        ▼
/validate-traces
→ Đảm bảo không có trace nào còn trỏ về spec cũ
```

**Nguyên tắc:** Không merge code khi traces broken. Fix traces trước.

---

### Tình huống 5: Nhận bug report từ Tester

**Bối cảnh:** Tester gửi bug report theo đúng format spec-driven, có đầy đủ spec context.

```
Bug ID   : BUG-20260605-003
Feature  : FT-001 — User Login
Service  : BE
Severity : Major

Spec context:
  PRD      : specs/prd/auth/FT-001-login.md (v1.0)
  BDD      : free-trial-be/specs/bdd/auth/FT-001-login.feature
             → Scenario: "Lock account after 5 failed attempts"
  Tech Doc : free-trial-be/specs/tech-docs/auth/FT-001-auth-api.md

AC bị vi phạm:
  AC3: "Sai password 5 lần liên tiếp → khoá tài khoản 30 phút"

BDD Scenario bị fail:
  Given : user has 0 failed attempts
  When  : login with wrong password 5 times
  Then  : 5th attempt returns 423 Locked AND retry_after = 1800

Actual: 5th attempt trả 401, không có retry_after, tài khoản không bị khoá
```

**Bước 1 — Xác định bug thuộc layer nào:**

```
Đọc theo thứ tự: PRD → BDD → Code

PRD AC3: "5 lần sai → khoá 30 phút"          → rõ ràng ✅
BDD SC3: "Then 423 Locked, retry_after=1800"  → đúng theo PRD ✅
Code: ???                                      → kiểm tra tiếp
```

Chạy:
```
/fix-bug "BUG-20260605-003: FT-001-UC2-BR3 — account not locked after 5 failures
  PRD: specs/prd/auth/FT-001-login.md
  BDD: free-trial-be/specs/bdd/auth/FT-001-login.feature"
```

Agent sẽ:
1. Đọc BDD scenario được chỉ định
2. Tìm code implement scenario đó (theo `@trace.bdd`)
3. So sánh logic thực tế vs spec
4. Propose fix có giải thích

**Bước 2 — Xác định bug thuộc layer nào:**

Đọc theo thứ tự PRD → BDD → Code để tìm chỗ lệch. Có 3 khả năng:

| PRD | BDD | Code | → Fix ở đâu |
|---|---|---|---|
| ✅ rõ | ✅ đúng | ❌ sai | Fix code |
| ✅ rõ | ❌ sai | ❌ sai | Fix BDD + code |
| ❌ mơ hồ | bất kỳ | bất kỳ | Hỏi PO trước, không tự fix |

> Flow đầy đủ cho cả 6 cases (bao gồm PRD change, Design Spec bug, env bug) và cách phối hợp với PO/Tester: xem **[BUG_FLOW.md](BUG_FLOW.md)**.

**Bước 3 — Sau khi fix:**

```
/validate-traces
→ Đảm bảo @trace.bdd trong code vẫn trỏ đúng BDD scenario
→ Không có trace broken

/run-tests
→ BDD pass = fix đúng theo spec

Notify tester:
  "BUG-20260605-003 fixed — deploy to staging [link commit/PR]
   Root cause: Case A — code dùng > thay vì >=
   BDD: không đổi (spec đã đúng)
   Re-test: FT-001-UC2-SC3"
```

---

### Tình huống 6: Nhận Design Spec từ PO (FE/App)

**Bối cảnh:** PO tạo Design Spec cho tính năng checkout — FE cần implement.

```
PO thông báo: "Design Spec FT-042-checkout-design.md đã sẵn sàng"
        │
        ▼
FE dev đọc:
- specs/prd/payment/FT-042-checkout.md (business rules)
- specs/design/payment/FT-042-checkout-design.md (screens, components, flow)

        │
        ▼
/generate-bdd specs/prd/payment/FT-042-checkout.md
→ Platform: FE
→ Agent đọc cả Design Spec → gen BDD với đúng screen names + component names

        │
        ▼
/sync-figma-components (nếu dùng Figma)
→ Đồng bộ component tokens từ Figma vào codebase
```

**Lưu ý:** BE không cần đọc Design Spec — chỉ đọc PRD.

---

### Tình huống 7: Setup service submodule (Umbrella mode)

**Bối cảnh:** Project dùng umbrella repo. Dev được assign vào service `mass-product-be`.

**Setup lần đầu:**

```bash
# 1. Clone umbrella repo
git clone {umbrella-repo-url} mass-product
cd mass-product

# 2. Init submodules
git submodule update --init --recursive

# 3. Mở Claude Code TẠI umbrella root (QUAN TRỌNG)
cd mass-product   ← umbrella root, không phải service dir
code .            ← hoặc claude .

# 4. Framework tự detect umbrella mode từ project-context.yaml
# Khi chạy /review-context với PRD có @trace.domain: be
# → tự động route tới mass-product-be/specs/bdd/
```

**project-context.yaml của umbrella:**
```yaml
setup:
  mode: umbrella
  spec_source: "mass-product-spec"
services:
  be:
    path: "mass-product-be"
    module: "NestJS"
    specs_dir: "mass-product-be/specs/bdd"
    tech_docs_dir: "mass-product-be/specs/tech-docs"
  web:
    path: "mass-product-web"
    module: "NextJS"
    specs_dir: "mass-product-web/specs/bdd"
    tech_docs_dir: "mass-product-web/specs/tech-docs"
```

**Commit 2 lớp (bắt buộc):**

```bash
# Lớp 1: Commit trong service submodule
cd mass-product-be
git add specs/bdd/auth/FT-001-login.feature
git commit -m "feat(bdd): add login BDD scenarios — FT-001"
git push origin feature/ft-001-login

# Lớp 2: Update pointer tại umbrella
cd ..   ← về umbrella root
git add mass-product-be
git commit -m "chore: update mass-product-be submodule pointer — FT-001 BDD"
git push
```

**Không commit lớp 2 → umbrella repo vẫn trỏ về commit cũ của service.**

---

### Tình huống 8: Validate traces trước khi tạo PR lớn

**Bối cảnh:** Dev refactor module Auth — đổi tên `AuthService` → `IdentityService`.

```
Sau khi refactor xong:

/validate-traces
        │
        ▼
Agent kiểm tra:
- BDD có @trace.module: AuthService → BROKEN (class không còn tồn tại)
- Code comments @trace.bdd: FT-001-UC1-SC1 → còn hợp lệ không?
- Tech Docs mention "AuthService" → stale reference

        │
        ▼
Report:
  ❌ BROKEN  specs/bdd/auth/FT-001-login.feature  @trace.module: AuthService (not found)
  ❌ BROKEN  specs/tech-docs/auth/FT-001-auth-api.md  "AuthService" referenced 7 times
  ✅ OK      src/identity/identity.service.ts  @trace.bdd: FT-001-UC1-SC1

        │
        ▼
Fix: Update @trace.module và references → re-run /validate-traces → all green → tạo PR
```

**Quy tắc:** PR không được merge khi còn broken traces.

---

## Checklist trước khi tạo PR

- [ ] `/validate-traces` → all green (không broken trace)
- [ ] `/run-tests` → all pass
- [ ] `/review-code` → không có issue Critical hoặc Major chưa xử lý
- [ ] BDD scenarios đã cover đủ: happy path + edge cases + error paths
- [ ] Code có `@trace.bdd` comment cho các function implement BDD scenario
- [ ] Tech Docs đã được update nếu có thay đổi API/DB schema
- [ ] Không tự sửa PRD hay BDD mà không có lý do — log lại nếu có divergence
