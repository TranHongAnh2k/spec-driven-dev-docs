# Hướng Dẫn PO/BA — Spec-Driven Dev

Tài liệu này dành riêng cho **Product Owner (PO)** và **Business Analyst (BA)**.  
Mô tả các tình huống thực tế và cách xử lý từng bước.

---

## Mục Lục

1. [Vai trò PO/BA trong framework](#1-vai-trò-poba-trong-framework)
2. [Commands dành cho PO/BA](#2-commands-dành-cho-poba)
3. [Tình huống thực tế](#3-tình-huống-thực-tế)
4. [Quy tắc viết PRD hiệu quả](#4-quy-tắc-viết-prd-hiệu-quả)
5. [Checklist handoff cho dev team](#5-checklist-handoff-cho-dev-team)

---

## 1. Vai Trò PO/BA Trong Framework

```
PO/BA là người duy nhất viết và approve tài liệu đầu vào + BDD:

  PO/BA                     Dev team
  ──────────────────────    ──────────────────────────────
  /define-product           đọc product-definition + BDD
  /generate-prd        →    /review-context (xác nhận PRD)
  /refine-prd (SA review)   /generate-tech-docs
  /review-context           /generate-code + tests
  /generate-design-spec →   /review-code / /run-tests
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

**3 lý do chính:**

**1. Tổng hợp tài liệu nghiệp vụ tại một nơi**

Khi BDD nằm trong spec repo cùng PRD, mọi spec nghiệp vụ đều ở một chỗ — dễ review, audit, và generate báo cáo. Dev team đọc spec từ submodule, không cần tự suy ra scenarios.

**2. BDD viết ở mức nghiệp vụ — PO hiểu được**

BDD trong spec repo mô tả **hành vi hệ thống** bằng ngôn ngữ nghiệp vụ, không phải implementation detail:

| Platform | BDD trong spec repo (PO viết) | Test code (Dev viết) |
|---|---|---|
| Web | `When user submits login → sees dashboard` | `await page.click('#login-btn')` |
| App | `When user taps Login → navigates to Home` | `element(by.id('login')).tap()` |
| System | `When login request received → returns auth token` | `POST /api/auth/login assertions` |

PO viết BDD ở tầng nghiệp vụ — dev đọc và tự map xuống tool/framework của mình.

**3. System BDD là nguồn sự thật cho BE**

"System BDD" được tổng hợp từ FE + App BDD:
- FE BDD nói: "User expects auth token để truy cập tiếp"
- App BDD nói: "App expects user profile sau khi đăng nhập"
- System BDD nói: "BE phải trả `{ token, user_profile }` để thỏa mãn cả 2 platform"

BE team đọc System BDD → implement API đúng contract, không cần đoán hay coordinate riêng với FE/App team.

---

## 2. Commands Dành Cho PO/BA

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/setup-ai-first` | Khởi tạo spec repo | 1 lần duy nhất khi bắt đầu project |
| `/define-product` | Tạo product definition | Trước khi viết PRD — capture ý tưởng tính năng |
| `/generate-prd` | Tạo PRD từ product definition | Sau khi product definition đủ rõ |
| `/refine-prd` | Review PRD qua 4 lens (QA/DEV/SA/PO) | Trước khi share với dev team |
| `/review-context` | Review chất lượng + consistency của PRD | Sau refine, trước khi approve |
| `/review-context --fix` | Auto-fix các lỗi nhỏ trong PRD | Khi có nhiều lỗi minor tự sửa được |
| `/review-context --resume` | Apply các fix sau Review Board | Sau khi review findings |
| `/generate-design-spec` | Tạo Design Spec cho FE/App | Sau khi PRD approved, trước khi generate BDD |
| `/generate-bdd` | Tạo BDD cho web / app / system | Sau khi PRD + Design Spec approved |
| `/learn {text}` | Ghi lại lỗi AI hay lặp khi gen PRD/BDD thành guardrail | Khi AI lặp lại cùng kiểu sai trong PRD/BDD |

> **Brownfield shortcut:** Nếu API đã tồn tại trên hệ thống cũ, thêm `API Source: existing` vào PRD Metadata + điền section "Existing API Contract". BDD generation sẽ dùng contract đó trực tiếp, T7 sign-off gate tự động được bỏ qua.

> **Project Lessons:** Nếu AI cứ lặp lại một kiểu sai khi gen PRD/BDD (vd: quên negative path, dùng sai thuật ngữ), chạy `/learn "AI hay X, đúng phải Y"` (category `prd`/`bdd`). Lesson được nạp vào đầu mọi lệnh → AI không lặp lại. Commit file lessons để cả team dùng chung.

> **Xử lý feedback từ tester:** Tester `/report-bug` và `/propose-scenario` commit vào `{spec_source}/feedback/` của spec repo. PO/Dev **thấy chúng khi chạy `/sync`** (dòng `📥 New tester feedback` liệt kê bug + proposal mới kéo về). PO review proposal:
> - **Map vào AC sẵn có** → coverage gap thật → để Dev thêm scenario vào `.feature` (hoặc regen).
> - Là **PRD change request** (behavior chưa có trong PRD) → PO thêm/sửa AC → bump version → `/generate-bdd` lại. Đây là cách edge case tester tìm ra quay về đúng quy trình spec.

---

## 3. Tình Huống Thực Tế

---

### Tình huống 1 — Bắt đầu tính năng mới từ đầu

**Bối cảnh:** PO nhận yêu cầu tính năng mới, cần viết tài liệu để dev team implement.

**Bước 1 — Capture ý tưởng:**
```
/define-product
```
Agent sẽ hỏi:
- Tên tính năng, domain
- Mô tả ngắn
- Vấn đề cần giải quyết
- Actors liên quan
- Kết quả mong muốn

Output: `specs/product-definition/FEAT-01-login.md`

**Bước 2 — Generate PRD:**
```
/generate-prd specs/product-definition/FEAT-01-login.md
```

Agent tự động:
- Đọc product definition
- Expand thành PRD đầy đủ với UC, AC, BR
- Nhắc nếu bạn viết UI details (để giữ platform-agnostic)
- Kiểm tra terminology với business-dictionary

Output: `specs/prd/auth/FEAT-01-login-prd.md`

**Bước 3 — Review nội dung:**
```
/refine-prd specs/prd/auth/FEAT-01-login-prd.md
```

Agent review qua 4 lens và tạo findings file. Mở findings file, xem xét từng finding:
- `accepted` → apply
- `modified` → viết note sửa theo ý bạn
- `rejected` → bỏ qua

```
/review-context --resume specs/prd/auth/FEAT-01-login-prd.md
```

**Bước 4 — Final check:**
```
/review-context specs/prd/auth/FEAT-01-login-prd.md
```

Kiểm tra: `@trace.status`, `@trace.domain`, completeness.

**Bước 5 — Approve PRD:**
```yaml
# Trong file PRD, cập nhật:
@trace.status: approved
@trace.version: 1.0
```

**Bước 6 — Tạo Design Spec (FE/App):**
```
/generate-design-spec specs/prd/auth/FEAT-01-login-prd.md
```
Agent hỏi platform (web / app). Sau khi Designer review + confirm Figma links → `@trace.status: approved`.

**Bước 7 — Generate BDD:**
```
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
```
Agent hỏi: **"Platform? (1) web  (2) app  (3) system"**

- Chọn `web` → Output: `specs/bdd/auth/web/FEAT-01-UC1-login-web.feature`
- Chạy lại, chọn `app` → Output: `specs/bdd/auth/app/FEAT-01-UC1-login-app.feature`
- Chạy lại, chọn `system` → Agent tổng hợp từ web+app BDDs → `specs/bdd/auth/system/FEAT-01-UC1-login-system.feature`

> Nếu project chỉ có web (không có app) → chỉ cần gen `web` rồi `system`.

**Bước 8 — Push và thông báo:**
```bash
git add specs/prd/ specs/design-spec/ specs/bdd/
git commit -m "feat(auth): add FEAT-01 PRD, design spec, and BDD"
git push
```

**Thông báo dev team:** "FEAT-01 PRD + BDD đã sẵn sàng, domain: `auth`. BDD tại `specs/bdd/auth/`"

---

### Tình huống 2 — Viết PRD khi dev team chưa setup

**Bối cảnh:** PO bắt đầu viết tài liệu, dev team chưa có umbrella repo.

**Không cần đợi dev team.** PO viết bình thường trong spec repo.

```
/define-product         → product definition
/generate-prd           → PRD với @trace.domain: auth
/review-context --fix   → auto-fix
/generate-design-spec   → design spec cho FE (nếu có designer)
/generate-bdd → web     → specs/bdd/auth/web/
/generate-bdd → system  → specs/bdd/auth/system/ (tổng hợp từ web)
```

**Điều duy nhất cần làm:** thống nhất domain names với dev team trước.

```bash
# Khi /setup-ai-first hỏi "List your business domains":
# → nhập: auth, payment, loyalty

# Communicate với dev team:
# "Domain names của project: auth, payment, loyalty
#  Khi setup umbrella, dùng đúng tên này làm services keys"
```

Khi dev team join sau → pull spec submodule → `/review-context` P0 check sẽ verify domain matching.

---

### Tình huống 3 — Tạo Design Spec và BDD sau khi PRD approved

**Bối cảnh:** PRD đã được approve. Cần tạo Design Spec + BDD trước khi handoff cho dev team.

**Điều kiện:** Có Figma link hoặc wireframe từ Designer.

**Bước 1 — Design Spec:**
```
/generate-design-spec specs/prd/auth/FEAT-01-login-prd.md
```

Agent hỏi platform: `web` hay `app`? → Output: `specs/design-spec/auth/FEAT-01-design-spec-web.md`

Sau khi Designer review → cập nhật Figma links → `@trace.status: approved`.

**Bước 2 — Generate BDD:**
```
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
```
Agent hỏi platform. Chạy 3 lần:

| Lần | Platform | Output |
|---|---|---|
| 1 | `web` | `specs/bdd/auth/web/FEAT-01-UC1-login-web.feature` |
| 2 | `app` | `specs/bdd/auth/app/FEAT-01-UC1-login-app.feature` |
| 3 | `system` | `specs/bdd/auth/system/FEAT-01-UC1-login-system.feature` |

Khi chọn `system`, agent đọc web+app BDDs → tổng hợp contract BE cần đáp ứng.

```bash
git add specs/design-spec/ specs/bdd/
git commit -m "feat(auth): add FEAT-01 design spec and BDD (web+app+system)"
git push
```

**Thông báo dev team:** "FEAT-01 BDD đã sẵn sàng tại `specs/bdd/auth/` — FE đọc `web/`, App đọc `app/`, BE đọc `system/`"

> **Lưu ý:** Nếu project chỉ có web (không có mobile app) → bỏ qua bước gen `app`, chỉ cần `web` và `system`.

---

### Tình huống 4 — PRD bị review trả lại (NEEDS_REVISION)

**Bối cảnh:** Dev team hoặc SA review PRD và trả lại với nhận xét.

**Bước 1 — Đọc findings:**
```bash
# Mở file trong .agent/review/
cat .agent/review/FEAT-01-login-prd-review-context-findings.yaml
```

Hoặc mở VS Code Extension → Review Board để xem visual.

**Bước 2 — Xử lý từng finding:**

Với mỗi finding, set status:
- `accepted` — đồng ý, apply theo suggestion
- `modified: "note của bạn"` — đồng ý nhưng muốn sửa khác đi
- `rejected` — không đồng ý, giữ nguyên

**Bước 3 — Apply:**
```
/review-context --resume specs/prd/auth/FEAT-01-login-prd.md
```

Agent apply tất cả finding `accepted`/`modified`, bump version, add changelog.

**Bước 4 — Re-review:**
```
/review-context specs/prd/auth/FEAT-01-login-prd.md
```

Lặp lại cho đến khi `recommendation: APPROVED`.

---

### Tình huống 5 — Requirements thay đổi sau khi PRD đã approved

**Bối cảnh:** Stakeholder yêu cầu thay đổi tính năng. PRD đã ở status `approved`, dev team có thể đã generate BDD.

**Bước 1 — Đổi status về draft:**
```yaml
# Trong PRD file:
@trace.status: draft    # ← đổi lại
@trace.version: 1.0     # sẽ được bump khi apply
```

**Bước 2 — Sửa nội dung:**

Có 2 cách:
- Tự sửa file trực tiếp
- Hoặc dùng `/refine-prd` với custom criteria:
  ```
  /refine-prd specs/prd/auth/FEAT-01-login-prd.md "Thêm yêu cầu: hỗ trợ đăng nhập bằng OTP"
  ```

**Bước 3 — Review lại:**
```
/review-context specs/prd/auth/FEAT-01-login-prd.md
/review-context --resume    (nếu cần fix)
```

**Bước 4 — Approve và thông báo:**
```yaml
@trace.status: approved
@trace.version: 1.1     # minor bump vì chỉ thêm AC, không đổi scope lớn
                         # major bump (2.0) nếu thay đổi cơ bản
```

```bash
git add specs/prd/auth/FEAT-01-login-prd.md
git commit -m "feat(auth): update FEAT-01 PRD v1.1 — add OTP login AC"
git push
```

**Thông báo dev team:** "FEAT-01 PRD updated v1.0 → v1.1, domain: `auth`. Cần check lại BDD coverage."

> **Dev team sẽ làm:** `/review-context {feature-file}` để kiểm tra B1 coverage gap
> → generate thêm scenarios nếu cần.

---

### Tình huống 6 — Thêm domain mới vào project

**Bối cảnh:** Project mở rộng, thêm domain `loyalty` chưa có trong config.

**Bước 1 — Cập nhật project-context.yaml:**
```yaml
domains:
  - auth
  - payment
  - loyalty    # ← thêm vào đây
```

**Bước 2 — Thông báo dev team cập nhật umbrella config:**

Dev team cần thêm vào `services` section trong umbrella:
```yaml
# my-project-be/.agent/project-context.yaml
services:
  loyalty:
    path: "loyalty-service"
    module: "java-spring"
    specs_dir: "loyalty-service/specs/bdd"
```

**Bước 3 — Viết PRD bình thường:**
```
/define-product
/generate-prd
# @trace.domain: loyalty
```

> **Lưu ý:** Nếu dev team chưa cập nhật services config → `/review-context` P0 check sẽ cảnh báo
> `@trace.domain: loyalty` không match. Dev team cần fix trước khi generate BDD.

---

### Tình huống 7 — Conflict giữa 2 PRDs (P3 check)

**Bối cảnh:** `/review-context` báo P3 — PRD mới mâu thuẫn với PRD cũ trong cùng domain.

Ví dụ: FEAT-01 quy định "session hết hạn sau 30 phút", nhưng FEAT-05 quy định "session hết hạn sau 2 giờ".

**Cách xử lý:**

1. Đọc findings file — P3 sẽ liệt kê cả 2 PRD và điểm mâu thuẫn
2. Quyết định PRD nào đúng:
   - Nếu FEAT-05 là yêu cầu mới nhất → sửa FEAT-01 cho consistent
   - Nếu FEAT-01 vẫn còn hiệu lực → sửa FEAT-05

3. Sửa file cần thiết:
   ```
   /review-context --resume specs/prd/auth/FEAT-01-login-prd.md
   # với finding P3: modified: "session timeout cập nhật thành 2 giờ theo FEAT-05"
   ```

4. Cả 2 PRD phải nhất quán trước khi dev team generate BDD.

---

### Tình huống 8 — Cập nhật Business Dictionary

**Bối cảnh:** Team phát hiện thuật ngữ mới cần chuẩn hóa, hoặc cần ban một số term.

**Cách 1 — Cập nhật thủ công:**
```bash
# Mở file trực tiếp
specs/domain-knowledge/business-dictionary.md

# Thêm vào Canonical Terms hoặc Banned Terms
```

**Cách 2 — Để agent phát hiện tự động:**

Khi chạy `/generate-prd` hoặc `/review-context`, nếu agent phát hiện term mới xuất hiện nhiều lần → sẽ hỏi:
> "Term `{term}` xuất hiện nhiều lần nhưng chưa có trong dictionary. Thêm vào không?"

Trả lời:
- Xác nhận định nghĩa
- Quyết định canonical name
- Agent tự cập nhật business-dictionary.md

**Sau khi cập nhật:**
```bash
git add specs/domain-knowledge/business-dictionary.md
git commit -m "docs: add/update business dictionary — {term}"
git push
```

> ⚠️ **Quan trọng:** Sau khi ban một term mới, chạy `/review-context --fix` trên các PRDs hiện tại
> để replace banned terms tự động.

---

### Tình huống 9 — Quản lý nhiều PRD song song

**Bối cảnh:** PO đang viết/update nhiều tính năng cùng lúc.

**Cấu trúc thư mục sẽ trông như sau:**
```
specs/prd/
├── auth/
│   ├── FEAT-01-login-prd.md          (approved)
│   └── FEAT-08-sso-prd.md            (draft)
├── payment/
│   ├── FEAT-03-checkout-prd.md       (approved)
│   └── FEAT-11-refund-prd.md         (draft)
└── loyalty/
    └── FEAT-06-points-prd.md         (in-review)
```

**Gợi ý làm việc:**
- Hoàn thiện 1 PRD đến `approved` trước khi bắt đầu PRD tiếp theo
- Chỉ có PRD `approved` mới được dev team sử dụng
- Dùng VS Code Extension → Living Documentation để track trạng thái

**Xem nhanh tất cả PRDs và status:**
```bash
grep -r "@trace.status" specs/prd/ --include="*.md"
```

---

### Tình huống 10 — Handoff PRD cho dev team

**Bối cảnh:** PRD hoàn tất, muốn thông báo dev team bắt đầu.

**Checklist trước khi thông báo:**

```yaml
# Kiểm tra trong PRD file:
@trace.id: FEAT-01          ✅ có
@trace.domain: auth         ✅ khớp với services keys của dev team
@trace.status: approved     ✅ đã approved (không phải draft)
@trace.version: 1.0         ✅ có version number
```

**Kiểm tra với framework:**
```
/review-context specs/prd/auth/FEAT-01-login-prd.md
```
→ Phải thấy `recommendation: APPROVED` và 0 critical findings.

**Template thông báo cho dev team:**
```
[FEAT-01] PRD Login đã approved — sẵn sàng implement

Domain: auth
File: my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
Version: 1.0
Design Spec (Web): my-project-specs/specs/design-spec/auth/FEAT-01-design-spec-web.md

Lệnh để bắt đầu:
  git submodule update --remote my-project-specs
  /review-context my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
  /generate-bdd my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
```

---

### Tình huống 11 — Brownfield: API đã tồn tại trên hệ thống cũ

**Bối cảnh:** Feature mới cần dùng/mô tả API đã có sẵn trên hệ thống cũ. Không muốn thiết kế lại từ đầu — chỉ cần ghi lại contract để dev team implement đúng.

**Cách làm:**

**Bước 1 — Khi viết PRD, thêm `API Source: existing` vào Metadata:**
```markdown
| **API Source** | existing |
```

**Bước 2 — Điền section "Existing API Contract" trong Appendix của PRD:**
```markdown
## Existing API Contract

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | /api/v1/orders | Bearer | `{ product_id, quantity }` | `{ id, status, total }` |
| GET  | /api/v1/orders/:id | Bearer | — | `{ id, items[], status, created_at }` |

**Error responses:**

| HTTP Status | Error Code | Khi nào xảy ra |
|-------------|------------|----------------|
| 404 | ORDER_NOT_FOUND | Order ID không tồn tại |
| 422 | INSUFFICIENT_STOCK | Số lượng vượt tồn kho |
```

**Bước 3 — Generate BDD như bình thường:**
```
/generate-bdd → system
```
→ Framework tự nhận ra `API Source: existing`, dùng contract trong PRD thay vì synthesize từ web + app BDD.

**Bước 4 — Generate Tech Docs:**
```
/generate-tech-docs
```
→ Chạy ở mode **Reverse-document**: mô tả lại API đã tồn tại, ghi chú gaps.

**Điều không cần làm:**
- Không cần sign-off gate T7 (framework tự skip)
- Không cần `--phase=ui` (API đã live)
- Không cần conflict resolution

---

## 4. Quy Tắc Viết PRD Hiệu Quả

### PRD là platform-agnostic

| ✅ Viết trong PRD | ❌ Không viết trong PRD |
|---|---|
| "Đăng nhập thành công → truy cập tính năng" | "Click button Login → hiển thị spinner" |
| "Sai password 5 lần → khoá 30 phút" | "API POST /auth/login trả về JWT" |
| "Session hết hạn → yêu cầu xác thực lại" | "Toast message màu đỏ ở góc phải" |
| "User có thể reset password qua email" | "Gọi sendgrid API để gửi email" |

UI details và API specs thuộc về:
- **Design Spec** → UI/UX cho FE/App
- **Tech Docs** → API contract, technical design (dev team tự generate)

### UC và AC phải testable

Mỗi AC phải trả lời được câu hỏi: **"Làm sao biết tính năng này hoạt động đúng?"**

❌ Không testable:
```
AC: "Hệ thống xử lý nhanh"
AC: "UI thân thiện với người dùng"
```

✅ Testable:
```
AC: "Trang login load trong vòng 3 giây"
AC: "Error message hiển thị trong vòng 1 giây sau khi submit"
```

### BR phải có ID liên tục

```
UC1 → BR1, BR2, BR3
UC2 → BR4, BR5          ← tiếp tục từ BR3, không reset về BR1
UC3 → BR6
```

### Luôn có negative path

Với mỗi AC happy path, cần có ít nhất 1 AC cho error case:

```
AC1 (happy): Đăng nhập thành công với email + password hợp lệ
AC2 (error): Đăng nhập thất bại khi password sai → hiển thị lỗi
AC3 (error): Tài khoản bị khoá → hiển thị thông báo + thời gian còn lại
```

---

## 5. Checklist Handoff Cho Dev Team

Trước khi thông báo dev team bắt đầu, kiểm tra:

**Metadata:**
- [ ] `@trace.id` có và đúng format (vd: FEAT-01)
- [ ] `@trace.domain` có và khớp với domain names đã thống nhất với dev team
- [ ] `@trace.status: approved` (không phải draft/in-review)
- [ ] `@trace.version` có (vd: 1.0)

**Nội dung PRD:**
- [ ] Tất cả `{{PLACEHOLDER}}` đã được điền
- [ ] Ít nhất 1 UC với format `{TICKET-ID}-UC1`
- [ ] Mỗi UC có: Description, Actors, Preconditions, AC, BR
- [ ] Có cả happy path và error cases trong AC
- [ ] Không có UI details (button colors, API paths) trong PRD
- [ ] Changelog section có ít nhất 1 entry

**Framework check:**
- [ ] `/review-context` cho kết quả `APPROVED` (0 critical findings)
- [ ] Không có P3 conflict với PRD khác trong cùng domain

**Cho FE/App thêm:**
- [ ] Design Spec đã tạo và `@trace.status: approved`
- [ ] Figma links đã được điền trong Design Spec (nếu có)
- [ ] Designer đã sign-off

**BDD (bắt buộc trước khi handoff):**
- [ ] BDD web đã gen: `specs/bdd/{domain}/web/{TICKET-ID}-UC*.feature`
- [ ] BDD app đã gen: `specs/bdd/{domain}/app/{TICKET-ID}-UC*.feature` (nếu project có app)
- [ ] System BDD đã gen: `specs/bdd/{domain}/system/{TICKET-ID}-UC*.feature`
- [ ] Tất cả BDD files đã được review và không có `MISSING` trong Coverage Matrix

**Git:**
- [ ] Đã commit và push toàn bộ (`specs/prd/`, `specs/design-spec/`, `specs/bdd/`) lên spec repo
- [ ] Thông báo dev team domain name và BDD path để họ cập nhật `git submodule update --remote`

---

*Xem thêm:* [OPERATIONS.md](OPERATIONS.md) — hướng dẫn vận hành cho toàn team  
*Xem thêm:* [SETUP_GUIDE.md](SETUP_GUIDE.md) — hướng dẫn cài đặt framework
