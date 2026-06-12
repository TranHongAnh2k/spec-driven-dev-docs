[📚 Docs](../../README.md) › [Guides](../README.md) › [Product Owner](README.md) › Tình huống thực tế

# Tình Huống Thực Tế

## Tình huống 1 — Bắt đầu tính năng mới từ đầu

**Bối cảnh:** PO nhận yêu cầu tính năng mới, cần viết tài liệu để dev team implement.

**Bước 1 — Capture ý tưởng:**
```
/define-product
```
Agent sẽ hỏi: Tên tính năng, domain · Mô tả ngắn · Vấn đề cần giải quyết · Actors liên quan · Kết quả mong muốn.

Output: `specs/product-definition/FEAT-01-login.md`

**Bước 2 — Generate PRD:**
```
/generate-prd specs/product-definition/FEAT-01-login.md
```
Agent tự động: đọc product definition · expand thành PRD đầy đủ với UC, AC, BR · nhắc nếu bạn viết UI details · kiểm tra terminology với business-dictionary.

Output: `specs/prd/auth/FEAT-01-login-prd.md`

**Bước 3 — Review nội dung:**
```
/refine-prd specs/prd/auth/FEAT-01-login-prd.md
```
Agent fan-out 4 lens (QA/DEV/SA/PO) chạy song song, rồi chạy completeness-critic loop cho đến khi một vòng không tìm ra finding mới, cuối cùng dedup + resolve conflict. Findings đầy đủ trong 1 lần chạy.

Mở findings file, xem xét từng finding: `accepted` → apply · `modified` → viết note · `rejected` → bỏ qua.
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
Agent hỏi platform (web / app). PO phải cung cấp **Figma link node-level** (URL chứa `?node-id=`, lấy bằng right-click vào frame → "Copy link to selection") cho **mỗi screen**. Screen nào thiếu link → bị flag ❌ Missing, Status giữ `draft`, `/generate-bdd` bị BLOCKED cho đến khi đủ link.

Sau khi Designer review + confirm đủ Figma node-id links → `@trace.status: approved`.

**Bước 7 — Generate BDD:**
```
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
```
Agent hỏi: **"Platform? (1) web  (2) app  (3) system"**
- Chọn `web` → `specs/bdd/auth/web/FEAT-01-UC1-login-web.feature`
- Chạy lại, chọn `app` → `specs/bdd/auth/app/FEAT-01-UC1-login-app.feature`
- Chạy lại, chọn `system` → Agent tổng hợp từ web+app BDDs → `specs/bdd/auth/system/FEAT-01-UC1-login-system.feature`

> Nếu project chỉ có web → chỉ cần gen `web` rồi `system`.

**Bước 8 — Push và thông báo:**
```bash
git add specs/prd/ specs/design-spec/ specs/bdd/
git commit -m "feat(auth): add FEAT-01 PRD, design spec, and BDD"
git push
```
**Thông báo dev team:** "FEAT-01 PRD + BDD đã sẵn sàng, domain: `auth`. BDD tại `specs/bdd/auth/`"

---

## Tình huống 2 — Viết PRD khi dev team chưa setup

**Bối cảnh:** PO bắt đầu viết tài liệu, dev team chưa có umbrella repo. **Không cần đợi dev team.**

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

---

## Tình huống 3 — Tạo Design Spec và BDD sau khi PRD approved

**Bối cảnh:** PRD đã được approve. Cần tạo Design Spec + BDD trước khi handoff.

**Điều kiện:** Có Figma link **node-level** (URL chứa `?node-id=`, right-click frame → "Copy link to selection") cho từng screen.

**Bước 1 — Design Spec:**
```
/generate-design-spec specs/prd/auth/FEAT-01-login-prd.md
```
Output: `specs/design-spec/auth/FEAT-01-design-spec-web.md`

Mỗi screen cần Figma link node-id. Screen thiếu → flag ❌ Missing, BLOCKED. Sau khi Designer review → `@trace.status: approved`.

**Bước 2 — Generate BDD:**
```
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
```

| Lần | Platform | Output |
|---|---|---|
| 1 | `web` | `specs/bdd/auth/web/FEAT-01-UC1-login-web.feature` |
| 2 | `app` | `specs/bdd/auth/app/FEAT-01-UC1-login-app.feature` |
| 3 | `system` | `specs/bdd/auth/system/FEAT-01-UC1-login-system.feature` |

```bash
git add specs/design-spec/ specs/bdd/
git commit -m "feat(auth): add FEAT-01 design spec and BDD (web+app+system)"
git push
```

---

## Tình huống 4 — PRD bị review trả lại (NEEDS_REVISION)

**Bước 1 — Đọc findings:**
```bash
cat .agent/review/FEAT-01-login-prd-review-context-findings.yaml
```

**Bước 2 — Xử lý từng finding:** set status `accepted` · `modified: "note của bạn"` · `rejected`.

**Bước 3 — Apply:**
```
/review-context --resume specs/prd/auth/FEAT-01-login-prd.md
```

**Bước 4 — Re-review:**
```
/review-context specs/prd/auth/FEAT-01-login-prd.md
```
Lặp lại cho đến khi `recommendation: APPROVED`.

---

## Tình huống 5 — Requirements thay đổi sau khi PRD đã approved

**Bước 1 — Đổi status về draft:**
```yaml
@trace.status: draft
```

**Bước 2 — Sửa nội dung:**
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
@trace.version: 1.1     # minor bump vì chỉ thêm AC
                         # major bump (2.0) nếu thay đổi cơ bản
```
```bash
git add specs/prd/auth/FEAT-01-login-prd.md
git commit -m "feat(auth): update FEAT-01 PRD v1.1 — add OTP login AC"
git push
```

**Bước 5 — Re-generate BDD cho các platform bị ảnh hưởng:**
```
/generate-bdd → web     (nếu thay đổi ảnh hưởng FE behavior)
/generate-bdd → app     (nếu project có app)
/generate-bdd → system
→ BDD mới phản ánh AC đã thay đổi
```
```bash
git add specs/bdd/
git commit -m "feat(auth): update BDD for FEAT-01 v1.1 — OTP login scenarios"
git push
```
> **Lưu ý:** Dev **KHÔNG tự generate BDD** — đây là trách nhiệm của PO. Nếu bỏ qua bước này, dev sẽ code theo BDD cũ và traces sẽ broken.

**Thông báo dev team:** "FEAT-01 PRD + BDD updated v1.0 → v1.1, domain: `auth`. Pull spec submodule và bắt đầu với BDD mới."

---

## Tình huống 6 — Thêm domain mới vào project

**Bước 1 — Cập nhật project-context.yaml:**
```yaml
domains:
  - auth
  - payment
  - loyalty    # ← thêm vào đây
```

**Bước 2 — Thông báo dev team cập nhật umbrella config:**
```yaml
services:
  loyalty:
    path: "loyalty-service"
    module: "java-spring"
    specs_dir: "loyalty-service/specs/bdd"
```

**Bước 3 — Viết PRD bình thường với `@trace.domain: loyalty`.**

> **Lưu ý:** Nếu dev team chưa cập nhật services config → `/review-context` P0 check sẽ cảnh báo domain không match.

---

## Tình huống 7 — Conflict giữa 2 PRDs (P3 check)

**Bối cảnh:** FEAT-01 quy định "session hết hạn sau 30 phút", FEAT-05 quy định "session hết hạn sau 2 giờ".

1. Đọc findings file — P3 liệt kê cả 2 PRD và điểm mâu thuẫn.
2. Quyết định PRD nào đúng.
3. Sửa file:
   ```
   /review-context --resume specs/prd/auth/FEAT-01-login-prd.md
   # với finding P3: modified: "session timeout cập nhật thành 2 giờ theo FEAT-05"
   ```
4. Cả 2 PRD phải nhất quán trước khi dev team generate BDD.

---

## Tình huống 8 — Cập nhật Business Dictionary

**Cách 1 — Thủ công:** Mở `specs/domain-knowledge/business-dictionary.md` → thêm vào Canonical Terms hoặc Banned Terms.

**Cách 2 — Để agent phát hiện:** Khi chạy `/generate-prd` hoặc `/review-context`, agent sẽ hỏi nếu phát hiện term mới chưa có trong dictionary.

**Sau khi cập nhật:**
```bash
git add specs/domain-knowledge/business-dictionary.md
git commit -m "docs: add/update business dictionary — {term}"
git push
```

> ⚠️ Sau khi ban một term mới, chạy `/review-context --fix` trên các PRDs hiện tại để replace banned terms tự động.

---

## Tình huống 9 — Quản lý nhiều PRD song song

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

**Gợi ý:** Hoàn thiện 1 PRD đến `approved` trước khi bắt đầu PRD tiếp theo. Chỉ PRD `approved` mới được dev team sử dụng.

```bash
# Xem nhanh tất cả PRDs và status:
grep -r "@trace.status" specs/prd/ --include="*.md"
```

---

## Tình huống 10 — Handoff PRD cho dev team

**Checklist trước khi thông báo:**
```yaml
@trace.id: FEAT-01          ✅ có
@trace.domain: auth         ✅ khớp với services keys của dev team
@trace.status: approved     ✅ đã approved
@trace.version: 1.0         ✅ có version number
```

```
/review-context specs/prd/auth/FEAT-01-login-prd.md
→ Phải thấy recommendation: APPROVED và 0 critical findings
```

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
```

---

## Tình huống 11 — Brownfield: API đã tồn tại trên hệ thống cũ

**Bước 1 — Thêm `API Source: existing` vào PRD Metadata:**
```markdown
| **API Source** | existing |
```

**Bước 2 — Điền section "Existing API Contract" trong Appendix:**
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
Framework tự nhận ra `API Source: existing`, dùng contract trong PRD trực tiếp.

**Bước 4 — Generate Tech Docs:**
```
/generate-tech-docs
```
Chạy ở mode **Reverse-document**: mô tả lại API đã tồn tại, ghi chú gaps.

**Không cần:** Sign-off gate T7 (tự động skip) · `--phase=ui` (API đã live).

---

← [Commands](commands.md) · Tiếp theo: [Quy tắc viết PRD](prd-writing-rules.md)
