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
/define-product           /review-context (đọc PRD + BDD)
/generate-prd        →    đọc BDD từ spec submodule
/refine-prd               /generate-tech-docs (từ BDD → Tech Docs)
/review-context           /generate-code     (từ BDD + Tech Docs → Code)
/generate-design-spec →   /generate-tests
/generate-bdd (web)       /review-code
/generate-bdd (app)       /run-tests
/generate-bdd (system)    /fix-bug / /debug
                          /validate-traces
```

**Dev chịu trách nhiệm:**
- Đọc và hiểu PRD + BDD từ spec submodule trước khi bắt đầu
- **KHÔNG tự generate BDD** — BDD đã được PO generate trong spec repo
- Đảm bảo code trace về đúng BDD scenario, BDD trace về đúng PRD
- Báo PO/BA khi PRD hoặc BDD có gì không rõ hoặc mâu thuẫn — không tự suy diễn

**Dev KHÔNG làm:**
- Viết/sửa PRD — đó là việc của PO/BA
- Viết/sửa Design Spec — đó là việc của PO/BA + Designer
- Approve PRD — chỉ PO mới có quyền này

---

## 2. Commands Dành Cho Dev

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/sync` `[spec-branch]` | **One-command setup hoặc update** — git pull + submodule sync + Living Docs refresh (nội dung dự án). Truyền branch để override branch spec submodule (vd `/sync develop`) | **Mỗi sáng trước khi bắt đầu work** |
| `/update-framework` | Nâng cấp **bản thân framework** (`.agent/commands/`, steps/, modules/) từ npm | Khi có version framework mới — không đụng project-context/CLAUDE.md |
| `/review-context {prd-file}` | Đọc + xác nhận PRD + BDD đủ rõ trước khi code | **Bước đầu tiên** khi nhận PRD mới |
| `/generate-tech-docs {prd-file}` | Sinh Tech Docs (API, DB schema, sequence diagram) | Sau khi đọc BDD từ spec submodule |
| `/generate-code {bdd-file}` | Sinh code — BE hoặc FE khi API đã sẵn sàng | Sau khi tech docs `approved` |
| `/generate-code {bdd-file} --phase=ui` | FE: gen UI + mock adapter (BE chưa ready) | Ngay sau khi đọc BDD |
| `/generate-code {bdd-file} --phase=integration` | FE: wire API thật thay mock | Sau khi sign-off gate `approved` |
| `/generate-tests {bdd-file}` | Sinh test cases từ BDD | Song song hoặc sau generate-code |
| `/review-code {file}` | Review code theo 4 lens (Security/Perf/Arch/Test) | Trước khi tạo PR |
| `/review-tech-docs {tech-doc-file}` | Review chất lượng Tech Docs | Sau generate-tech-docs |
| `/run-tests` | Chạy test suite hiện tại — *umbrella mode: tự `cd` vào service_root, dùng service's `test_command`* | Sau khi code + tests sẵn sàng |
| `/fix-bug {issue}` | Phân tích + fix bug có trace | Khi có bug report |
| `/debug {symptom}` | Debug vấn đề chưa rõ nguyên nhân | Khi cần trace root cause |
| `/smoke-test` | Kiểm tra nhanh các luồng chính | Sau deploy hoặc merge lớn |
| `/validate-traces` | Kiểm tra toàn bộ trace chain còn hợp lệ | Sau refactor hoặc khi PRD update |
| `/learn {text}` | Ghi lại lỗi AI hay lặp thành guardrail | Khi AI lặp lại lỗi mà bạn không muốn nó tái diễn |

### Project Lessons — dạy framework không lặp lỗi

AI đôi khi lặp đi lặp lại một lỗi trong dự án (vd: gọi repository thẳng từ controller, quên null-check). Thay vì sửa thủ công mỗi lần, **ghi lại thành "lesson"** — context-loader sẽ nạp nó vào đầu **mọi** lệnh như một ràng buộc cứng.

**2 cách ghi nhận:**

```bash
# Cách 1 — chủ động
/learn AI hay gọi repository thẳng từ controller, phải đi qua service layer

# Cách 2 — tự động: khi /review-code, /fix-bug, /debug phát hiện lỗi lặp lại
# → nó hỏi "Record as a project lesson? (Y/N)" → Y
```

**Lưu ở đâu:** `paths.lessons_file` (mặc định `specs/domain-knowledge/lessons-learned.md`; umbrella: `.agent/project-lessons.md` mỗi service). **Commit file này** để cả team cùng được bảo vệ.

> Đây là **bộ nhớ dự án**, không phải fine-tune model — lesson được nạp vào context mỗi lần chạy, nên AI "nhớ" và không lặp lại. Xem `[CTX LOADED]` có dòng `Lessons: loaded — N guardrails`.

### Xử lý feedback từ tester

Tester gửi bug report (`/report-bug`) và đề xuất scenario (`/propose-scenario`) vào `feedback/` của **spec repo**. Khi dev chạy `/sync`, nó liệt kê:

```
📥 New tester feedback (pulled this sync):
   Bug reports:    BUG-20260608-01  FT-001 — ...  [layer: Code]
   Scenario proposals: FT-001-trailing-spaces → AC2 (pending review)
```

Dev hành động theo phân loại:
- **Bug report** → `/fix-bug {BUG-ID}` (report đã có sẵn spec-context + AC bị vi phạm + layer)
- **Scenario proposal map vào AC sẵn có** → thêm scenario vào `.feature` (hoặc `/generate-bdd` lại), rồi `/generate-code` + `/generate-tests`
- **Proposal là yêu cầu mới (PRD change request)** → chuyển PO sửa PRD trước

> Tester chỉ *đề xuất* trong `feedback/` — dev/PO mới đưa vào BDD chính thức. Giữ đúng ownership.

### Khi nào dùng `--phase` cho FE/App?

| Tình huống | Command |
|---|---|
| API **đã có sẵn** và đang hoạt động | `/generate-code {file}` — không flag, gen real API ngay |
| BE **chưa ready**, FE muốn bắt đầu ngay | `/generate-code {file} --phase=ui` — UI + mock adapter |
| Sign-off gate xong, cần wire API thật | `/generate-code {file} --phase=integration` |
| BE implement (system BDD) | `/generate-code {file}` — không flag |

> `--phase` chỉ có giá trị khi BE chưa sẵn sàng. Nếu API đã live → bỏ qua `--phase`, chạy thẳng default.

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

### BDD đến từ spec repo — Dev đọc, không tự gen

BDD được PO generate trong spec repo, nằm tại `specs/bdd/{domain}/`:

| Subfolder | Platform | Dev team đọc |
|---|---|---|
| `web/` | FE/Web (clicks, sees, navigates) | FE/Web dev |
| `app/` | Mobile (taps, sees screen, navigates) | App dev |
| `system/` | System/BE (request, response, business rules) | BE dev |

Cả 3 subfolder đều trace về **cùng 1 PRD**. BE không cần đọc BDD của FE và ngược lại.

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
- **Sau mỗi codegen session trong umbrella mode** — để sync Living Docs panel

```
/validate-traces
→ Sẽ report: broken links, orphan BDD (không có PRD), dead code traces
```

**Lưu ý khi dùng umbrella (submodule):**

```
Vấn đề: Living Docs panel mở ở umbrella root → đọc .trace/ ở umbrella root → TRỐNG.
         TSV thực tế nằm trong từng service submodule: user-service/.trace/, order-service/.trace/

Giải pháp: /validate-traces sẽ tự gom từ tất cả service submodule và sync về umbrella .trace/

Lệnh chạy sau mỗi session:
/validate-traces
→ Reads từ user-service/.trace/, order-service/.trace/, ...
→ Copies TSVs → .trace/{service}/{UC-ID}.tsv (umbrella root)
→ Writes merged trace-report.json → .trace/trace-report.json
→ Living Docs panel cập nhật ngay
```

Thêm umbrella `.trace/` vào `.gitignore` (chỉ là mirror, không commit):
```
# .gitignore tại umbrella root
.trace/
```

---

## 5. Workflow Cơ Bản

```
Nhận thông báo PRD + BDD mới từ PO
        │
        ▼
git submodule update --remote my-project-specs
(lấy spec mới nhất, bao gồm cả BDD đã được PO gen)
        │
        ▼
/review-context {prd-file}
→ Kiểm tra @trace.domain, @trace.status = approved
→ Đọc hiểu AC, UC, BR trong PRD
→ Đọc BDD tương ứng trong specs/bdd/{domain}/{platform}/
→ Nếu có gì không rõ: hỏi PO TRƯỚC khi tiếp tục
        │
        ▼
(Đọc BDD từ spec submodule — KHÔNG tự generate BDD)
FE/Web:  my-project-specs/specs/bdd/{domain}/web/{TICKET-ID}-UC*.feature
App:     my-project-specs/specs/bdd/{domain}/app/{TICKET-ID}-UC*.feature
BE:      my-project-specs/specs/bdd/{domain}/system/{TICKET-ID}-UC*.feature
        │
        ▼
/generate-tech-docs {prd-file}
→ Gen API spec, DB schema, sequence diagram dựa trên BDD
→ /review-tech-docs để verify chất lượng
        │
        ▼
# BE (hoặc full-stack không cần mock split):
/generate-code {bdd-file}
→ Gen code skeleton theo BDD + tech docs
→ Đảm bảo @trace.bdd comment trong code

# FE/App (2 phases — không cần chờ BE):
/generate-code {bdd-file} --phase=ui          # Phase 1: UI + mock adapter
→ Tester test FE ngay (không cần BE ready)
/generate-code {bdd-file} --phase=integration # Phase 2: sau khi sign-off done
→ Wire real API thay thế mock
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

### Tình huống 1: Nhận PRD + BDD mới và bắt đầu work

**Bối cảnh:** PO thông báo PRD `FT-042-checkout.md` và BDD đã approved, sẵn sàng implement.

**Các bước:**

```
1. git submodule update --remote my-project-specs
   (lấy PRD + BDD mới nhất từ PO)

2. /review-context my-project-specs/specs/prd/payment/FT-042-checkout.md
   → Kiểm tra @trace.status = approved (không code khi còn draft)
   → Đọc kỹ AC, UC, BR

3. Đọc BDD tương ứng theo platform của mình:
   FE/Web:  my-project-specs/specs/bdd/payment/web/FT-042-UC*.feature
   App:     my-project-specs/specs/bdd/payment/app/FT-042-UC*.feature
   BE:      my-project-specs/specs/bdd/payment/system/FT-042-UC*.feature

4. Nếu có thắc mắc về PRD hoặc BDD → hỏi PO ngay, không tự suy diễn
   Ví dụ: "BR5 trong System BDD nói 'kiểm tra giới hạn thanh toán' — limit này
           có khác nhau theo tier user không? BDD không chỉ rõ."

5. Bắt đầu: /generate-tech-docs dựa trên BDD
```

**Lưu ý:** Nếu `/review-context` báo P0 warning (domain không match config) → **dừng lại**, báo PO/DevOps fix config trước.

---

### Tình huống 2: Đọc và hiểu System BDD (BE dev)

**Bối cảnh:** BE dev nhận thông báo BDD đã sẵn sàng tại `specs/bdd/auth/system/`.

**System BDD tập trung vào:**
- API contracts được tổng hợp từ FE + App BDD
- Business rule enforcement tại system level
- Data contracts (request/response shape)
- Cross-platform consistency

```
# Đọc file BDD (không generate):
my-project-specs/specs/bdd/auth/system/FT-001-UC1-login-system.feature
```

```gherkin
# Ví dụ System BDD do PO gen (tổng hợp từ web + app BDD)
Feature: User Authentication — System Contract
  # @trace.prd: FT-001
  # @trace.platform: system

  Scenario: Successful login returns token and profile
    Given a registered user with valid credentials
    When the system receives a login request
    Then the system returns an auth token
    And the system returns the user profile
    And the session is valid for 3600 seconds

  Scenario: Account locked after 5 failed attempts
    Given a user with 4 failed login attempts
    When the system receives a 5th failed login
    Then the system locks the account for 30 minutes
    And the system signals the locked state with remaining time
```

BE dev dùng System BDD để:
1. Thiết kế API endpoint + response schema (`/generate-tech-docs`)
2. Generate code skeleton (`/generate-code`)
3. Viết integration tests (`/generate-tests`)

---

### Tình huống 3: Đọc Web/App BDD (FE/App dev)

**Bối cảnh:** FE dev nhận thông báo BDD web đã sẵn sàng tại `specs/bdd/auth/web/`.

```
# Đọc file BDD (không generate):
my-project-specs/specs/bdd/auth/web/FT-001-UC1-login-web.feature
```

```gherkin
# Ví dụ Web BDD do PO gen (vocabulary: clicks, sees, navigates)
Scenario: User sees error after wrong password
  Given user is on the Login screen
  When user submits login with wrong password
  Then user sees "Sai mật khẩu" error
  And the password field is cleared

Scenario: Account locked — countdown shown
  Given user has submitted wrong password 5 times
  Then user sees "Tài khoản bị khoá. Thử lại sau 29:45"
  And the countdown decrements every second
```

FE dev dùng Web BDD để:
1. Thiết kế component spec + API integration plan (`/generate-tech-docs`)
2. Gen UI + mock adapter từ System BDD contract (`/generate-code --phase=ui`)
   → Mock adapter trả về fixture data đúng với BDD `Then` clauses
   → Tester test toàn bộ FE flow ngay, không cần chờ BE
3. [Trong khi đó — tham gia review API contract, sign-off T7 gate]
4. Khi sign-off done → wire real API (`/generate-code --phase=integration`)
5. Viết E2E tests với Playwright/Cypress (`/generate-tests`)

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
- BDD bị ảnh hưởng? → thông báo PO để PO update BDD trong spec repo, rồi pull lại
- Tech Docs bị ảnh hưởng? → update API spec
- Code bị ảnh hưởng? → update logic + tests

        │
        ▼
/validate-traces
→ Đảm bảo không có trace nào còn trỏ về spec cũ
```

**Nguyên tắc:** Không merge code khi traces broken. Fix traces trước.

---

### Tình huống 4b: Chờ API design — BE + FE/App đồng thuận

**Bối cảnh:** System BDD đã gen, BE dev bắt đầu `/generate-tech-docs` nhưng FE/App chưa confirm API contract.

**Trạng thái tech docs trong thời gian này:** `@trace.status: in-review`

```
BE dev: /generate-tech-docs auth/FT-001-UC1
→ Output: user-service/specs/tech-docs/auth/FT-001-UC1-tech-design.md
→ @trace.status: draft
→ @trace.sign_off: { be_team: done, fe_team: pending, app_team: pending, sa: pending }

BE dev: /review-tech-docs user-service/specs/tech-docs/auth/FT-001-UC1-tech-design.md
→ Chạy T1–T7 (bao gồm T7 cross-team contract check)
→ Report: "Sign-off gate: 🔒 BLOCKED — pending: fe_team, app_team, sa"
```

**FE dev review API contract:**
```
# FE dev mở tech-design file → xem API contract section
# Xác nhận: response fields có đủ cho web BDD expectations không?
# Nếu ok → thêm comment hoặc báo BE dev cập nhật sign_off
```

Khi FE/App confirm xong → BE dev update header:
```yaml
# @trace.sign_off:
#   be_team:  done
#   fe_team:  done    ← FE đã confirm
#   app_team: done    ← App đã confirm
#   sa:       done    ← SA đã approve
```

```
BE dev: /review-tech-docs --resume {tech-design-file}
→ Sign-off gate: ✅ READY
→ @trace.status: approved
→ BE có thể chạy /generate-code
→ FE chạy /generate-code --phase=integration để wire API thật
```

```
# FE — sau khi sign-off gate approved:
/generate-code --phase=integration auth/FT-001-UC1
→ Reads existing mock adapter interface ({UC-ID}ApiPort)
→ Generates real API adapter với calls đến endpoints trong tech-doc
→ Flips DI/env flag: service dùng real adapter thay mock
→ Mock adapter giữ lại cho unit test
```

**Nguyên tắc:** 
- `/generate-code` (không phase flag) cho BE trả về warning nếu tech docs status là `in-review` hoặc `draft`.
- FE dùng `--phase=ui` được ngay sau khi đọc BDD — không cần chờ.
- FE dùng `--phase=integration` chỉ sau khi sign-off gate `approved`.

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

### Tình huống 6: Nhận Design Spec + BDD từ PO (FE/App)

**Bối cảnh:** PO tạo Design Spec + BDD web cho tính năng checkout — FE cần implement.

```
PO thông báo: "FT-042 Design Spec + BDD đã sẵn sàng"
        │
        ▼
git submodule update --remote my-project-specs

FE dev đọc:
- my-project-specs/specs/prd/payment/FT-042-checkout.md      (business rules)
- my-project-specs/specs/design-spec/payment/FT-042-*.md     (screens, components)
- my-project-specs/specs/bdd/payment/web/FT-042-UC*.feature  (BDD đã gen sẵn)

        │
        ▼
/sync-figma-components (nếu dùng Figma)
→ Đồng bộ component tokens từ Figma vào codebase

        │
        ▼
/generate-tech-docs payment/FT-042-UC1
→ Gen component spec, API integration plan dựa trên Design Spec + BDD

        │
        ▼
/generate-code payment/FT-042-UC1 --phase=ui
→ Gen UI components + mock API adapter (fixture từ System BDD Then clauses)
→ Tester có thể test FE ngay

        │  [trong khi đó: tham gia review API contract — T7 sign-off gate]
        ▼
[Nhận thông báo: sign-off gate approved]

        │
        ▼
/generate-code payment/FT-042-UC1 --phase=integration
→ Wire real API adapter thay thế mock
→ /generate-tests payment/FT-042-UC1
→ /review-code {files-changed}
→ /run-tests
```

**Lưu ý:** BE không cần đọc Design Spec — chỉ đọc System BDD tại `specs/bdd/{domain}/system/`.

---

### Tình huống 7b: Brownfield — API đã tồn tại trên hệ thống cũ

**Bối cảnh:** PO viết PRD cho feature mới nhưng BE API đã có sẵn trên hệ thống cũ, chưa có tài liệu. PO khai báo luôn trong PRD thay vì thiết kế lại.

**Dấu hiệu nhận ra:**
- PRD Metadata có `| **API Source** | existing |`
- PRD có section "Existing API Contract" với bảng endpoint + response

**Dev workflow (đơn giản hơn greenfield):**

```
git submodule update --remote my-project-specs

1. /review-context → đọc PRD + BDD
   → BDD system đã dùng contract sẵn có từ PRD (không synthesis)
   → @trace.api_source: existing trong BDD header

2. /generate-tech-docs {feature-file}
   → Mode: Reverse-document
   → §2 API Endpoints: mô tả lại API đã tồn tại từ bảng PRD
   → Ghi chú gaps nếu contract thực tế khác BDD expectations

3. /review-tech-docs {tech-design-file}
   → T7 sign-off gate: tự động SKIP (không có API design mới)
   → Chỉ review T1–T6 (architecture, entity, BDD traceability, ...)
   → Approved nhanh hơn

4. /generate-code {feature-file}   ← không cần --phase
   → API đã live, gen real adapter trực tiếp
```

**Điểm khác biệt so với greenfield:**

| | Greenfield | Brownfield (API existing) |
|---|---|---|
| System BDD | Synthesize từ FE + App BDD | Dùng PRD contract trực tiếp |
| T7 gate | Bắt buộc | Tự động skip |
| `--phase=ui` | Cần nếu BE chưa ready | Không cần |
| `generate-tech-docs` | Design mới | Reverse-document |

---

### Tình huống 7: Setup service submodule (Umbrella mode)

**Bối cảnh:** Project dùng umbrella repo. Dev được assign vào service `mass-product-be`.

**Setup lần đầu:**

```bash
# 1. Clone umbrella repo
git clone {umbrella-repo-url} mass-product
cd mass-product

# 2. Mở Claude Code TẠI umbrella root (QUAN TRỌNG)
code .   ← hoặc claude .

# 3. Chạy một lệnh duy nhất — setup toàn bộ
/sync
# → tự detect setup mode (submodule chưa init)
# → git pull + git submodule update --init --recursive --remote
# → validate service configs (cảnh báo nếu thiếu .agent/project-context.yaml)
# → sync Living Docs panel

# 4. Framework tự detect umbrella mode từ project-context.yaml
# Khi chạy /review-context với PRD có @trace.domain: be
# → tự động route tới mass-product-be/specs/bdd/
```

**Update hằng ngày — cũng chỉ 1 lệnh:**

```bash
/sync
# → git pull + submodule update --remote
# → refresh Living Docs
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

> **Bắt buộc:** Mỗi service submodule cũng cần file `.agent/project-context.yaml` riêng. Framework đọc file này (context-loader Step 1.6) để lấy đúng `test_command` và `build_command` khi `/run-tests` hoặc `/generate-tests` chạy từ umbrella root.

**project-context.yaml của từng service submodule:**
```yaml
# mass-product-be/.agent/project-context.yaml
tech_stack:
  language: "TypeScript"
  framework: "NestJS"
  module: "nestjs"

conventions:
  test_command: "npm test"
  build_command: "npm run build"

paths:
  trace_dir: ".trace"
```

```yaml
# mass-product-web/.agent/project-context.yaml
tech_stack:
  language: "TypeScript"
  framework: "Next.js 14"
  module: "nextjs"

conventions:
  test_command: "npx vitest run"
  build_command: "npm run build"

paths:
  trace_dir: ".trace"
```

Khi `/run-tests` chạy từ umbrella root cho một UC thuộc domain `be`:
1. Step 1.5 detect `service_root = "mass-product-be"`
2. Step 1.6 load `mass-product-be/.agent/project-context.yaml` → `test_command = "npm test"`
3. Lệnh test chạy: `cd mass-product-be && npm test`

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
- [ ] `/run-tests` → all pass *(umbrella: đảm bảo service có `.agent/project-context.yaml` với `test_command` trước khi chạy)*
- [ ] `/review-code` → không có issue Critical hoặc Major chưa xử lý
- [ ] Code trace về đúng BDD scenarios trong `my-project-specs/specs/bdd/`
- [ ] Code có `@trace.bdd` comment cho các function implement BDD scenario
- [ ] Tech Docs đã được update nếu có thay đổi API/DB schema
- [ ] **Không tự sửa BDD** — BDD là của PO, nếu cần update thì báo PO rồi pull lại
