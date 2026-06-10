# Hướng Dẫn Tester — Spec-Driven Dev

Tài liệu này dành riêng cho **QA / Tester**.  
Mô tả cách tester's agent kết nối với spec framework, workflow kiểm thử, và các tình huống thực tế.

---

## Mục Lục

1. [Vai trò Tester trong framework](#1-vai-trò-tester-trong-framework)
2. [Spec-manifest là gì và tại sao cần](#2-spec-manifest-là-gì-và-tại-sao-cần)
3. [Setup tester agent](#3-setup-tester-agent)
4. [Workflow cơ bản](#4-workflow-cơ-bản)
5. [Đọc và hiểu spec chain](#5-đọc-và-hiểu-spec-chain)
6. [Tình huống thực tế](#6-tình-huống-thực-tế)
7. [Khi tìm thấy bug — quy trình trace](#7-khi-tìm-thấy-bug--quy-trình-trace)

---

## 1. Vai Trò Tester Trong Framework

```
PO/BA          Dev                    Tester
──────────     ──────────────────     ──────────────────────────
PRD            BDD (từ PRD)           /sync + /generate-spec-manifest
               Tech Docs              Đọc PRD + BDD + Tech Docs
               Code                   Viết test cases · Chạy test
   ▲              ▲                    /report-bug      (bug → spec-traced)
   │              │                    /propose-scenario (edge case → BDD draft)
   └──────────────┴───── feedback/ trong spec repo ◄────┘
        PO/Dev thấy qua /sync (📥) → fix / promote / update PRD
```

**Tester chịu trách nhiệm:**
- Đọc PRD để hiểu business requirement trước khi test
- Đọc BDD để biết chính xác scenarios cần cover
- Đọc Tech Docs để hiểu API contracts và data flow
- Viết test cases dựa trên BDD scenarios
- Khi bug xảy ra: `/report-bug` → tạo bug report có spec-context đầy đủ, phân loại layer
- Khi phát hiện edge case chưa có trong BDD: `/propose-scenario` → đề xuất scenario cho PO/Dev duyệt

**Tester KHÔNG làm:**
- Sửa PRD / BDD / Tech Docs trực tiếp — đó là việc của PO và Dev
- Approve PRD — chỉ PO
- Generate code — chỉ Dev

> **Lưu ý về `/propose-scenario`:** lệnh này **không** sửa BDD. Nó chỉ ghi một bản *đề xuất* (draft Gherkin) vào vùng `feedback/bdd-proposals/` của spec repo dùng chung. PO/Dev review và đưa vào `.feature` chính thức. Vậy nên không vi phạm nguyên tắc "tester không sửa BDD".

### Commands dành cho Tester

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/generate-spec-manifest` | Tạo index TICKET-ID → PRD/BDD/tech-doc | Trước mỗi sprint, sau khi pull specs mới |
| `/report-bug {UC-ID} {mô tả}` | Tạo bug report có spec-context + phân loại layer (read-only) | Khi test fail / phát hiện bug |
| `/propose-scenario {UC-ID} {mô tả}` | Đề xuất scenario BDD mới cho edge case chưa cover | Khi tìm thấy case ngoài BDD hiện có |

**Tester KHÔNG cần:**
- Hiểu codebase của từng service
- Biết framework đang được install như thế nào
- Đọc `project-context.yaml` trực tiếp

---

## 2. Spec-Manifest Là Gì Và Tại Sao Cần

Trong umbrella repo, spec files nằm rải rác ở nhiều submodule:

```
free-trial-spec/
├── specs/prd/auth/FT-001-login.md         ← PRD (spec submodule)
├── specs/product-definition/auth/...      ← PDD
├── free-trial-be/specs/bdd/...            ← BDD của BE
├── free-trial-web/free-trial-specs/...    ← BDD của Web
└── free-trial-app/specs/bdd/...           ← BDD của App
```

Tester's agent không biết layout này → cần một **index file** để tra cứu.

`spec-manifest.yaml` là file được gen tự động, ánh xạ TICKET-ID → tất cả file liên quan:

```yaml
features:
  FT-001:
    domain: auth
    status: approved
    prd: "specs/prd/auth/FT-001-login.md"
    pdd: "specs/product-definition/auth/FT-001-def.md"
    tech_docs:
      be:  "free-trial-specs/specs/tech-docs/auth/FT-001-auth-api.md"
      web: "free-trial-specs/specs/tech-docs/auth/FT-001-login-web.md"
    bdd:
      be:  "free-trial-be/specs/bdd/auth/FT-001-login.feature"
      web: "free-trial-web/free-trial-specs/free-trial-web/bdd/auth/FT-001-login-web.feature"
```

**Quan trọng:**
- File này **không commit vào git** (gitignored) — gen local mỗi khi cần
- Luôn `git pull` trước khi gen để có submodule mới nhất
- Chỉ test các feature có `status: approved` — draft PRD chưa được approve

---

## 3. Setup Tester Agent

### Lần đầu tiên

```bash
# 1. Clone umbrella repo (nếu chưa có)
git clone {umbrella-repo-url} free-trial-spec
cd free-trial-spec

# 2. Init tất cả submodules
git submodule update --init --recursive

# 3. Mở Claude Code tại umbrella root
#    QUAN TRỌNG: mở tại free-trial-spec/, không phải free-trial-tests/
```

### Trước mỗi test sprint

```bash
# Cập nhật tất cả submodules về commit mới nhất
git pull
git submodule update --remote --recursive

# Gen manifest
/generate-spec-manifest
# → viết spec-manifest.yaml tại free-trial-spec/ root
```

### Cấu hình tester agent đọc manifest

Trong tester's agent config (hoặc system prompt), thêm:

```
Trước khi test bất kỳ feature nào:
1. Đọc spec-manifest.yaml tại project root
2. Lookup TICKET-ID → lấy paths của prd, bdd, tech_docs
3. Chỉ test features có status: approved
4. Đọc PRD trước để hiểu business context
5. Đọc BDD để biết scenarios cần cover
```

### Living Docs Panel (umbrella mode)

Khi làm việc với umbrella repo (nhiều service submodule), VS Code Living Docs panel cần được đồng bộ thủ công trước khi xem.

**Panel đọc từ:** `umbrella-root/.trace/{service}/{UC-ID}.tsv`  
**Sync bằng:** `/validate-traces` chạy tại umbrella root

```bash
# Chạy sau mỗi codegen session — hoặc khi cần refresh trạng thái coverage
/validate-traces

# → Đọc TSVs từ: user-service/.trace/, order-service/.trace/, ...
# → Copy về:     .trace/{service}/{UC-ID}.tsv  (umbrella root)
# → Ghi:         .trace/trace-report.json      (aggregated toàn hệ thống)
# → Panel cập nhật ngay sau khi lệnh hoàn tất
```

Panel hiển thị trạng thái cross-service:

| Cột | Ý nghĩa |
|-----|---------|
| PRDs | Số PRD đã approve / tổng |
| UCs | Số Use Cases đã implement |
| Code Cov. | % scenarios có code |
| Test Cov. | % scenarios có test |
| DRIFT | Spec thay đổi sau khi gen code |
| GAP | Code có nhưng chưa có test |

> **Lưu ý:** `.trace/` tại umbrella root là **mirror read-only** — không commit vào git. Authoritative trace state ở từng service submodule.
>
> **Prerequisite cho data chính xác:** Mỗi service submodule cần file `.agent/project-context.yaml` với `paths.trace_dir` được configure đúng. Nếu thiếu, framework không biết ghi trace TSV về đâu → panel thiếu data. Báo Dev team kiểm tra nếu panel trống sau khi chạy `/validate-traces`.

---

## 4. Workflow Cơ Bản

```
Nhận task: "Test FT-042 — Checkout flow"
        │
        ▼
git pull && git submodule update --remote --recursive
        │
        ▼
/generate-spec-manifest
→ spec-manifest.yaml được refresh
        │
        ▼
Lookup FT-042 trong spec-manifest.yaml
→ status: approved? ✅ (nếu draft → dừng, báo PO)
→ lấy paths: prd, bdd.be, bdd.web, tech_docs.be
        │
        ▼
Đọc PRD (business understanding)
→ hiểu AC, UC, BR — đây là "ground truth"
        │
        ▼
Đọc BDD (test scenarios)
→ mỗi Scenario = 1 test case cần cover
→ lưu ý: BE có BDD riêng, Web có BDD riêng
        │
        ▼
Đọc Tech Docs (implementation details)
→ BE: API endpoints, request/response schema, error codes
→ Web: screen flow, component states
        │
        ▼
Viết và chạy test cases
→ happy path → edge cases → error paths
→ verify từng AC trong PRD được satisfy
        │
        ▼
Kết quả:
  PASS → done
  FAIL → trace về spec, báo cáo với đầy đủ context
```

---

## 5. Đọc Và Hiểu Spec Chain

### Thứ tự đọc và mục đích

| Tài liệu | Đọc để | Ví dụ thực tế |
|---|---|---|
| **PRD** | Hiểu WHAT — business requirement | "Sai password 5 lần → khoá 30 phút" |
| **BDD** | Biết HOW VERIFIED — exact scenarios | `Given 5 failed logins, Then 423 Locked` |
| **Tech Docs BE** | Biết API contract khi test BE | `POST /api/v1/auth/login`, error codes |
| **Tech Docs Web** | Biết UI states khi test Web | Screen names, component behaviors |

### Ví dụ đọc spec chain — FT-001 Login

**Bước 1: Đọc PRD** → hiểu business rules

```markdown
# AC trong PRD FT-001:
AC1: Đăng nhập thành công → truy cập được hệ thống
AC2: Sai password → hiển thị lỗi, không tiết lộ thông tin user
AC3: Sai password 5 lần liên tiếp → khoá tài khoản 30 phút
AC4: Tài khoản bị khoá → thông báo thời gian mở khoá
```

**Bước 2: Đọc BDD BE** → exact test scenarios cho API

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

**Bước 3: Đọc Tech Docs BE** → biết thêm chi tiết để test edge cases

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
- Email không tồn tại → 404 (không khác gì với 401 về mặt message — security by design)
- Sau 30 phút → có thể login lại

---

## 6. Tình Huống Thực Tế

---

### Tình huống 1: Test feature mới được assign

**Bối cảnh:** PO thông báo FT-042 Checkout đã approved, cần test trước khi release.

```bash
# 1. Cập nhật repo
git pull && git submodule update --remote --recursive

# 2. Refresh manifest
/generate-spec-manifest

# 3. Kiểm tra status
# Trong spec-manifest.yaml:
# FT-042:
#   status: approved  ← ✅ có thể test
#   prd: "specs/prd/payment/FT-042-checkout.md"
#   bdd:
#     be:  "free-trial-be/specs/bdd/payment/FT-042-checkout.feature"
#     web: "free-trial-web/.../bdd/payment/FT-042-checkout-web.feature"
```

**Test plan từ BDD BE (7 scenarios):**

| # | Scenario | Input | Expected |
|---|---|---|---|
| 1 | Checkout thành công | Cart đủ hàng, card valid | 201 + order_id |
| 2 | Cart rỗng | Không có items | 400 EMPTY_CART |
| 3 | Hết hàng | item_id đã sold out | 409 OUT_OF_STOCK |
| 4 | Card bị từ chối | card number test declined | 402 PAYMENT_DECLINED |
| 5 | Timeout payment gateway | mock timeout | 504 + order status PENDING |
| 6 | Giới hạn thanh toán | amount > 10,000,000 VND | 400 LIMIT_EXCEEDED |
| 7 | Session hết hạn | expired token | 401 UNAUTHORIZED |

---

### Tình huống 2: PRD thay đổi giữa sprint

**Bối cảnh:** Đang test FT-042 thì PO update PRD v1.0 → v1.1 (giới hạn từ 5tr → 10tr).

```bash
# Nhận thông báo từ PO/Dev: "FT-042 updated to v1.1"

# 1. Cập nhật
git pull && git submodule update --remote --recursive

# 2. Refresh manifest
/generate-spec-manifest
# spec-manifest.yaml: FT-042 prd_version: "1.1"

# 3. Đọc lại PRD — xem changelog
# FT-042 Changelog:
# v1.1 | 2026-06-05 | BR7: payment limit thay đổi 5,000,000 → 10,000,000
```

**Cập nhật test case:**

```
Test case 6 cũ:  amount > 5,000,000 → 400 LIMIT_EXCEEDED  ← STALE
Test case 6 mới: amount > 10,000,000 → 400 LIMIT_EXCEEDED ← theo v1.1

Thêm test case mới:
Test case 6b: amount = 9,999,999 → 201 OK (boundary case — giờ pass)
Test case 6c: amount = 10,000,001 → 400 LIMIT_EXCEEDED
```

**Lưu ý:** Nếu Dev chưa update code theo v1.1 → test sẽ fail vì code vẫn dùng limit 5tr → báo Dev, không phải bug.

---

### Tình huống 3: Test BE và Web cùng 1 feature

**Bối cảnh:** FT-001 Login có BDD cho cả BE (API) và Web (UI). Cần test đồng thời.

**BE test** — đọc `bdd.be` + `tech_docs.be`:

```
Endpoint: POST /api/v1/auth/login
Test tool: Postman / k6 / Jest supertest

Test cases từ BDD BE:
✅ SC1: Valid credentials → 200 + JWT
✅ SC2: Wrong password → 401 (message generic)
✅ SC3: 5 failures → 423 + retry_after
✅ SC4: Locked account → 423 (không cần thử lại)
✅ SC5: Email không tồn tại → 404 (message = 401 message)
```

**Web test** — đọc `bdd.web` + `tech_docs.web`:

```
Tool: Playwright

Test cases từ BDD Web:
✅ SC1: Điền đúng → redirect dashboard
✅ SC2: Sai password → toast error, password cleared
✅ SC3: 5 lần sai → form disabled + countdown timer
✅ SC4: "Quên mật khẩu?" link visible sau 3 lần sai
✅ SC5: Session expired → redirect về login với message
```

**Cross-check quan trọng:** BE trả `retry_after: 1800` → Web hiển thị countdown `29:59`. Nếu Web không đọc đúng header → bug.

---

### Tình huống 4: Feature chưa có BDD

**Bối cảnh:** Manifest báo FT-055 không có BDD:

```
⚠️  1 PRDs with no matching BDD:
    - FT-055 (notification) — run /generate-bdd first
```

**Xử lý:**

```
Không tự gen BDD — đó là việc của Dev.

1. Kiểm tra: PRD có status: approved không?
   → approved nhưng chưa có BDD → báo Dev lead
   → draft → chờ PO approve trước

2. Nếu bị yêu cầu test mà chưa có BDD:
   → Test dựa trên PRD AC trực tiếp
   → Ghi chú rõ trong test report: "Tested against PRD AC, no BDD available"
   → Không thể trace chi tiết → coverage thấp hơn

3. Sau khi Dev gen BDD → refresh manifest → test lại đầy đủ
```

---

### Tình huống 5: Regression testing sau merge lớn

**Bối cảnh:** Dev merge refactor module Auth (đổi tên `AuthService` → `IdentityService`). Cần chạy regression.

```bash
# 1. Cập nhật submodules (có thể có BDD/tech-docs mới)
git pull && git submodule update --remote --recursive

# 2. Refresh manifest
/generate-spec-manifest
```

**Scope regression:**

```
Cách xác định scope từ spec-manifest:

1. Domain bị ảnh hưởng: auth
2. Tất cả features có domain: auth trong manifest:
   - FT-001 (login)
   - FT-003 (register)
   - FT-007 (password reset)
   - FT-012 (2FA)

3. Features khác có @trace.module: IdentityService:
   → Grep trong tech_docs: "IdentityService"
   → FT-019 (profile) cũng phụ thuộc vào IdentityService

Regression scope: FT-001, FT-003, FT-007, FT-012, FT-019
```

---

### Tình huống 6: Test feature multi-service (BE + Web + App)

**Bối cảnh:** FT-042 Checkout có cả BE API, Web UI, và App UI. PO yêu cầu end-to-end test.

**Đọc manifest:**

```yaml
FT-042:
  domain: payment
  status: approved
  prd: "specs/prd/payment/FT-042-checkout.md"
  tech_docs:
    be:  "free-trial-specs/specs/tech-docs/payment/FT-042-checkout-api.md"
    web: "free-trial-specs/specs/tech-docs/payment/FT-042-checkout-web.md"
    app: "free-trial-specs/specs/tech-docs/payment/FT-042-checkout-app.md"
  bdd:
    be:  "free-trial-be/specs/bdd/payment/FT-042-checkout.feature"
    web: "free-trial-web/.../bdd/payment/FT-042-checkout-web.feature"
    app: "free-trial-app/specs/bdd/payment/FT-042-checkout-app.feature"
```

**Test strategy:**

```
Layer 1 — BE API (unit boundary):
→ Test /api/v1/checkout endpoint isolated
→ Tool: Postman hoặc Jest supertest
→ Verify: order created, payment processed, inventory updated

Layer 2 — Web E2E:
→ Tool: Playwright
→ Flow: Cart → Checkout page → Payment form → Order confirmation
→ Verify: UI states match Design Spec + BDD Web scenarios

Layer 3 — App E2E:
→ Tool: Maestro / Detox
→ Flow: Cart screen → Checkout → Payment → Success screen
→ Verify: mobile-specific behaviors (back button, network loss)

Layer 4 — Cross-layer:
→ Web checkout → kiểm tra order trong BE database
→ App checkout → cùng order visible trong Web dashboard
```

---

## 7. Khi Tìm Thấy Bug — Quy Trình Trace

Khi test fail, báo cáo phải có đủ **spec context** để Dev fix đúng chỗ.

### Cách nhanh nhất: `/report-bug`

```bash
/report-bug FT-001 tài khoản khoá sau 6 lần sai, spec ghi 5
```

Lệnh tự động:
- Resolve spec-context từ `spec-manifest.yaml`: PRD path + version, BDD scenario fail, tech-doc
- Tìm **AC bị vi phạm** trong PRD
- **Phân loại layer** theo BUG_FLOW (Code / BDD / PRD / Design Spec / Env) → route đúng người
- Phát hiện **coverage gap** (behavior không có scenario nào cover) → gợi ý `/propose-scenario`
- Ghi report vào **spec repo** (`{spec_source}/feedback/bug-reports/{BUG-ID}.md`) → **commit + push**

Hoàn toàn **read-only** trên spec/code chính thức — chỉ ghi vào vùng `feedback/`. Sửa là việc của Dev (`/fix-bug {BUG-ID}`).

> **Quan trọng — làm sao PO/Dev biết?** Report được commit+push vào **spec repo dùng chung**. PO/Dev chạy `/sync` hàng ngày sẽ thấy dòng `📥 New tester feedback` liệt kê bug mới kéo về. File nằm local một mình thì không ai biết — chính bước push + `/sync` mới khép vòng. (Không có quyền push spec repo → tạo PR/MR.)

### Khi bug là "thiếu scenario" — `/propose-scenario`

Nếu `/report-bug` báo *coverage gap* (behavior đúng nhưng chưa có scenario nào kiểm), đừng chỉ báo miệng — đề xuất luôn:

```bash
/propose-scenario FT-001 login với email có khoảng trắng ở cuối vẫn phải thành công
```

- Nếu behavior **đã nằm trong một AC của PRD** → lệnh draft Gherkin (tag `@proposed @from-test`), ghi vào `{spec_source}/feedback/bdd-proposals/` → commit + push. PO/Dev review rồi đưa vào `.feature`.
- Nếu behavior **chưa có trong PRD** → lệnh dừng và xuất *PRD change request* cho PO (vì scenario phải trace về AC). PO thêm AC trước, rồi `/generate-bdd`.

> Tester không tự ghi vào BDD — chỉ đề xuất. Giữ đúng ownership.
> Proposal cũng nằm trong spec repo → PO/Dev thấy qua `/sync` (`📥 New tester feedback`).

### Template báo cáo bug

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

### Ví dụ báo cáo thực tế

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

### Sau khi gửi bug report

Dev sẽ xác định bug thuộc layer nào (code / BDD / PRD / env) và phản hồi với root cause + scenario cần re-test.

> Xem flow phối hợp đầy đủ giữa Tester ↔ Dev ↔ PO trong **[BUG_FLOW.md](BUG_FLOW.md)** — bao gồm 6 cases, communication templates, và close checklist cho từng role.

> **Nếu bạn thấy AI lặp lại cùng một kiểu lỗi qua nhiều feature** (vd: luôn quên một validation, luôn sai một boundary): ghi chú điều này trong bug report. Dev có thể chạy `/learn` để biến nó thành guardrail — AI sẽ không sinh lại lỗi đó ở các UC sau. Đây là cách framework "học" từ bug bạn tìm ra.

---

## Checklist trước khi báo "test pass"

- [ ] Tất cả BDD scenarios đã được cover (happy path + edge + error)
- [ ] Mọi AC trong PRD đã được verify
- [ ] Test đã chạy trên đúng môi trường (staging, không phải local dev)
- [ ] Cross-service flows đã được verify (nếu feature span nhiều service)
- [ ] Manifest version khớp với PRD version đang deploy
- [ ] Không có scenarios bị skip mà không có lý do ghi chú
