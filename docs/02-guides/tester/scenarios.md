[📚 Docs](../../README.md) › [Guides](../README.md) › [Tester](README.md) › Tình huống thực tế

# Tình Huống Thực Tế

## Tình huống 1: Test feature mới được assign

**Bối cảnh:** PO thông báo FT-042 Checkout đã approved.

```bash
# 1. Cập nhật repo
git pull && git submodule update --remote --recursive

# 2. Refresh manifest
/generate-spec-manifest

# 3. Kiểm tra status
# FT-042:
#   status: approved  ← ✅ có thể test
#   prd: "specs/prd/payment/FT-042-checkout.md"
#   bdd:
#     be:  "free-trial-be/specs/bdd/payment/system/FT-042-UC1-checkout-system.feature"
#     web: "free-trial-web/specs/bdd/payment/web/FT-042-UC1-checkout-web.feature"
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

## Tình huống 2: PRD thay đổi giữa sprint

**Bối cảnh:** Đang test FT-042 thì PO update PRD v1.0 → v1.1 (giới hạn từ 5tr → 10tr).

```bash
git pull && git submodule update --remote --recursive
/generate-spec-manifest
# spec-manifest.yaml: FT-042 prd_version: "1.1"
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

> Nếu Dev chưa update code theo v1.1 → test fail vì code vẫn dùng limit 5tr → báo Dev, không phải bug.

---

## Tình huống 3: Test BE và Web cùng 1 feature

**Bối cảnh:** FT-001 Login có BDD cho cả BE (API) và Web (UI).

**BE test** — đọc `bdd.be` (System BDD) + `tech_docs.be`:

```
Endpoint: POST /api/v1/auth/login
Test tool: Postman / k6 / Jest supertest

Test cases từ BDD System (cho BE):
✅ SC1: Valid credentials → 200 + JWT
✅ SC2: Wrong password → 401 (message generic)
✅ SC3: 5 failures → 423 + retry_after
✅ SC4: Locked account → 423
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

## Tình huống 4: Feature chưa có BDD

**Manifest báo:**
```
⚠️  1 PRDs with no matching BDD:
    - FT-055 (notification) — run /generate-bdd first
```

**Xử lý:**
```
Không tự gen BDD — đó là việc của Dev.

1. PRD có status: approved nhưng chưa có BDD → báo Dev lead
   PRD còn draft → chờ PO approve trước

2. Nếu bị yêu cầu test mà chưa có BDD:
   → Test dựa trên PRD AC trực tiếp
   → Ghi chú rõ trong test report: "Tested against PRD AC, no BDD available"
   → Không thể trace chi tiết → coverage thấp hơn

3. Sau khi Dev gen BDD → refresh manifest → test lại đầy đủ
```

---

## Tình huống 5: Regression testing sau merge lớn

**Bối cảnh:** Dev merge refactor module Auth (đổi tên `AuthService` → `IdentityService`).

```bash
git pull && git submodule update --remote --recursive
/generate-spec-manifest
```

**Xác định scope regression từ spec-manifest:**

```
1. Domain bị ảnh hưởng: auth
2. Tất cả features có domain: auth:
   - FT-001 (login), FT-003 (register), FT-007 (password reset), FT-012 (2FA)

3. Features khác có @trace.module: IdentityService:
   → Grep trong tech_docs: "IdentityService"
   → FT-019 (profile) cũng phụ thuộc vào IdentityService

Regression scope: FT-001, FT-003, FT-007, FT-012, FT-019
```

---

## Tình huống 6: Test feature multi-service (BE + Web + App)

**Bối cảnh:** FT-042 Checkout có cả BE API, Web UI, và App UI.

```yaml
FT-042:
  domain: payment
  status: approved
  bdd:
    be:  "free-trial-be/specs/bdd/payment/system/FT-042-UC1-checkout-system.feature"
    web: "free-trial-web/specs/bdd/payment/web/FT-042-UC1-checkout-web.feature"
    app: "free-trial-app/specs/bdd/payment/app/FT-042-UC1-checkout-app.feature"
```

**Test strategy:**

```
Layer 1 — BE API:
→ Test /api/v1/checkout endpoint isolated
→ Tool: Postman hoặc Jest supertest

Layer 2 — Web E2E:
→ Tool: Playwright
→ Flow: Cart → Checkout page → Payment form → Order confirmation

Layer 3 — App E2E:
→ Tool: Maestro / Detox
→ Flow: Cart screen → Checkout → Payment → Success screen

Layer 4 — Cross-layer:
→ Web checkout → kiểm tra order trong BE database
→ App checkout → cùng order visible trong Web dashboard
```

---

← [Đọc Spec Chain](reading-specs.md) · Tiếp theo: [Báo cáo bug](bug-reporting.md)
