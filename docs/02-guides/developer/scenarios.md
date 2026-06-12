[📚 Docs](../../README.md) › [Guides](../README.md) › [Developer](README.md) › Tình huống thực tế

# Tình Huống Thực Tế

- [1. Nhận PRD + BDD mới và bắt đầu work](#tình-huống-1-nhận-prd--bdd-mới-và-bắt-đầu-work)
- [2. Đọc và hiểu System BDD (BE dev)](#tình-huống-2-đọc-và-hiểu-system-bdd-be-dev)
- [3. Đọc Web/App BDD (FE/App dev)](#tình-huống-3-đọc-webapp-bdd-feapp-dev)
- [4. PRD thay đổi mid-sprint](#tình-huống-4-prd-thay-đổi-mid-sprint)
- [4b. Chờ API design — BE + FE/App đồng thuận](#tình-huống-4b-chờ-api-design--be--feapp-đồng-thuận)
- [5. Nhận bug report từ Tester](#tình-huống-5-nhận-bug-report-từ-tester)
- [6. Nhận Design Spec + BDD từ PO (FE/App)](#tình-huống-6-nhận-design-spec--bdd-từ-po-feapp)
- [7b. Brownfield — API đã tồn tại](#tình-huống-7b-brownfield--api-đã-tồn-tại-trên-hệ-thống-cũ)
- [7. Setup service submodule (Umbrella)](#tình-huống-7-setup-service-submodule-umbrella-mode)
- [8. Validate traces trước khi tạo PR lớn](#tình-huống-8-validate-traces-trước-khi-tạo-pr-lớn)

## Tình huống 1: Nhận PRD + BDD mới và bắt đầu work

**Bối cảnh:** PO thông báo PRD `FT-042-checkout.md` và BDD đã approved, sẵn sàng implement.
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

## Tình huống 2: Đọc và hiểu System BDD (BE dev)

**Bối cảnh:** BE dev nhận thông báo BDD đã sẵn sàng tại `specs/bdd/auth/system/`.

**System BDD tập trung vào:** API contracts được tổng hợp từ FE + App BDD · Business rule enforcement tại system level · Data contracts (request/response shape) · Cross-platform consistency.
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
BE dev dùng System BDD để: thiết kế API endpoint + response schema (`/generate-tech-docs`) · generate code skeleton (`/generate-code`) · viết integration tests (`/dev-gen-test`).

## Tình huống 3: Đọc Web/App BDD (FE/App dev)

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
2. Gen UI + mock adapter từ System BDD contract (`/generate-code --phase=ui`) → Mock adapter trả về fixture data đúng với BDD `Then` clauses → Tester test toàn bộ FE flow ngay, không cần chờ BE
3. [Trong khi đó — tham gia review API contract, sign-off T7 gate]
4. Khi sign-off done → wire real API (`/generate-code --phase=integration`)
5. Viết E2E tests với Playwright/Cypress (`/dev-gen-test`)

## Tình huống 4: PRD thay đổi mid-sprint

**Bối cảnh:** PO update PRD `FT-042` từ v1.0 → v1.1 khi dev đang code.
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

## Tình huống 4b: Chờ API design — BE + FE/App đồng thuận

**Bối cảnh:** System BDD đã gen, BE dev bắt đầu `/generate-tech-docs` nhưng FE/App chưa confirm API contract. **Trạng thái tech docs trong thời gian này:** `@trace.status: in-review`.
```
BE dev: /generate-tech-docs auth/FT-001-UC1
# Umbrella mode (có spec_source): output nằm trong SPEC REPO chung
→ Output: free-trial-specs/specs/tech-docs/auth/FT-001-UC1-tech-design.md
# Single-service (không có spec_source): output nằm tại project root
→ Output: specs/tech-docs/auth/FT-001-UC1-tech-design.md
→ @trace.status: draft
→ @trace.sign_off: { be_team: done, fe_team: pending, app_team: pending, sa: pending }
→ Publish: commit + push file lên spec repo (2-layer) để FE/App `/sync` đọc được

BE dev: /review-tech-docs free-trial-specs/specs/tech-docs/auth/FT-001-UC1-tech-design.md
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

## Tình huống 5: Nhận bug report từ Tester

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
  Tech Doc : free-trial-specs/specs/tech-docs/auth/FT-001-auth-api.md

AC bị vi phạm:
  AC3: "Sai password 5 lần liên tiếp → khoá tài khoản 30 phút"

BDD Scenario bị fail:
  Given : user has 0 failed attempts
  When  : login with wrong password 5 times
  Then  : 5th attempt returns 423 Locked AND retry_after = 1800

Actual: 5th attempt trả 401, không có retry_after, tài khoản không bị khoá
```

**Bước 1 — Tìm code implement scenario bị fail:**
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
Agent sẽ: đọc BDD scenario được chỉ định · tìm code implement scenario đó (theo `@trace.bdd`) · so sánh logic thực tế vs spec · propose fix có giải thích.

**Bước 2 — Xác định bug thuộc layer nào:** Đọc theo thứ tự PRD → BDD → Code để tìm chỗ lệch. Có 3 khả năng:

| PRD | BDD | Code | → Fix ở đâu |
|---|---|---|---|
| ✅ rõ | ✅ đúng | ❌ sai | Fix code |
| ✅ rõ | ❌ sai | ❌ sai | Fix BDD + code |
| ❌ mơ hồ | bất kỳ | bất kỳ | Hỏi PO trước, không tự fix |

> Flow đầy đủ cho cả 6 cases (bao gồm PRD change, Design Spec bug, env bug) và cách phối hợp với PO/Tester: xem [Operations › Bug Flow](../../04-operations/bug-flow.md).

**Bước 3 — Sau khi fix:**
```
/validate-traces
→ Đảm bảo @trace.bdd trong code vẫn trỏ đúng BDD scenario
→ Không có trace broken

/dev-run-test
→ BDD pass = fix đúng theo spec

Notify tester:
  "BUG-20260605-003 fixed — deploy to staging [link commit/PR]
   Root cause: Case A — code dùng > thay vì >=
   BDD: không đổi (spec đã đúng)
   Re-test: FT-001-UC2-SC3"
```

## Tình huống 6: Nhận Design Spec + BDD từ PO (FE/App)

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
Bật Figma Dev Mode MCP (nếu dùng Figma — để FE codegen chính xác):
→ Mở Figma DESKTOP app + enable Dev Mode MCP server (local, http://127.0.0.1:3845)
→ `/generate-code` (FE/App UI) tự detect server local + prompt nếu chưa bật
→ Dùng real tokens, components, Code Connect thay vì web link → codegen sát design hơn
        │
        ▼
/generate-tech-docs payment/FT-042-UC1
→ Gen component spec, API integration plan dựa trên Design Spec + BDD
        │
        ▼
/generate-code payment/FT-042-UC1 --phase=ui
→ Gen UI components + mock API adapter (fixture từ System BDD Then clauses)
→ FE codegen đọc Figma Dev Mode MCP local nếu đang bật (tokens/components/Code Connect)
→ Tester có thể test FE ngay
        │  [trong khi đó: tham gia review API contract — T7 sign-off gate]
        ▼
[Nhận thông báo: sign-off gate approved]
        │
        ▼
/generate-code payment/FT-042-UC1 --phase=integration
→ Wire real API adapter thay thế mock
→ /dev-gen-test payment/FT-042-UC1
→ /review-code {files-changed}
→ /dev-run-test
```
**Lưu ý:** BE không cần đọc Design Spec — chỉ đọc System BDD tại `specs/bdd/{domain}/system/`.

## Tình huống 7b: Brownfield — API đã tồn tại trên hệ thống cũ

**Bối cảnh:** PO viết PRD cho feature mới nhưng BE API đã có sẵn trên hệ thống cũ, chưa có tài liệu. PO khai báo luôn trong PRD thay vì thiết kế lại.

**Dấu hiệu nhận ra:** PRD Metadata có `| **API Source** | existing |` · PRD có section "Existing API Contract" với bảng endpoint + response.

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

## Tình huống 7: Setup service submodule (Umbrella mode)

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
  web:
    path: "mass-product-web"
    module: "NextJS"
    specs_dir: "mass-product-web/specs/bdd"
```

> **Tech-docs (API contract):** không khai trong `services`. Khi `setup.spec_source` được set, BE tech-design (API contract) **LUÔN** nằm tại `{spec_source}/specs/tech-docs` (vd `mass-product-spec/specs/tech-docs`) — context-loader tự route `tech_docs_dir` về đó. BE generate xong → commit + push lên spec repo; FE/App đọc qua `/sync` + `/generate-code --phase=integration`. Per-service tech-docs **chỉ** khi KHÔNG có `spec_source`.
>
> **Bắt buộc:** Mỗi service submodule cũng cần file `.agent/project-context.yaml` riêng. Framework đọc file này (context-loader Step 1.6) để lấy đúng `test_command` và `build_command` khi `/dev-run-test` hoặc `/dev-gen-test` chạy từ umbrella root.

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

Khi `/dev-run-test` chạy từ umbrella root cho một UC thuộc domain `be`:
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

## Tình huống 8: Validate traces trước khi tạo PR lớn

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

← [Workflow](workflow.md) · Tiếp theo: [Checklist trước khi tạo PR](pr-checklist.md)
