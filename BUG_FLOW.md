# Quy Trình Xử Lý Bug — PO · Dev · Tester

Tài liệu này mô tả cách **PO, Dev, và Tester phối hợp** khi có bug được phát hiện.  
Mọi bug đều được trace về spec layer để fix đúng chỗ và tránh lặp lại.

---

## Tổng Quan: Bug Thuộc Layer Nào?

```
                    ┌─────────────────────────────────────────┐
                    │           Bug được phát hiện            │
                    └──────────────────┬──────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────┐
                    │   Tester đọc spec chain: PRD→BDD→Code   │
                    └──────────────────┬──────────────────────┘
                                       │
             ┌─────────────────────────┼──────────────────────────┐
             │                         │                          │
    ┌────────▼────────┐    ┌──────────▼──────────┐   ┌──────────▼──────────┐
    │  PRD mơ hồ /    │    │  BDD sai so với PRD │   │  Code sai so với    │
    │  yêu cầu thay   │    │  hoặc thiếu scenario│   │  BDD (đúng nhất)    │
    │  đổi            │    │                     │   │                     │
    └────────┬────────┘    └──────────┬──────────┘   └──────────┬──────────┘
             │                        │                          │
        → PO xử lý               → Dev xử lý               → Dev xử lý
        (Case 3, 4)               (Case 2)                  (Case 1)
```

---

## Bảng Quyết Định Nhanh

| PRD | BDD | Code | Chẩn đoán | Ai fix | Re-test |
|---|---|---|---|---|---|
| ✅ rõ ràng | ✅ đúng | ❌ sai | **Code bug** | Dev | Tester |
| ✅ rõ ràng | ❌ sai | ❌ sai | **BDD bug** | Dev | Tester |
| ❌ mơ hồ | bất kỳ | bất kỳ | **PRD ambiguity** | PO → Dev | Tester |
| ✅ rõ ràng | ✅ đúng | ✅ đúng | **Env / data bug** | DevOps / Dev | Tester |
| Yêu cầu thay đổi | cũ | cũ | **PRD change** | PO → Dev | Tester |
| ✅ rõ ràng | ✅ đúng | ❌ UI sai | **Design Spec bug** | PO/Designer → Dev | Tester |

---

## Case 1 — Code Bug (Code ≠ BDD)

> Phổ biến nhất. Spec đúng, code implement sai.

```
Tester                    Dev
──────                    ────────────────────────────────────────────
Gửi bug report
(kèm spec context:
 PRD path, BDD path,
 AC bị vi phạm,
 BDD scenario fail)
                          Nhận bug report
                          Đọc BDD scenario được chỉ định
                          /fix-bug "{BUG-ID}: {mô tả}"
                              → agent tìm divergence code vs BDD
                              → propose fix
                          Review + apply fix
                          /run-tests → BDD pass ✅
                          /validate-traces → no broken
                          Tạo PR + notify tester:
                          "Fixed — root cause: {X}
                           Re-test: {BDD scenario}"

Nhận thông báo
Re-test scenario
→ PASS: đóng bug         
→ FAIL: gửi lại với
  actual behavior mới
```

**Ví dụ thực tế:**

```
Bug: FT-001 — tài khoản khoá sau 6 lần sai, không phải 5

PRD AC3 : "5 lần sai → khoá"          ✅
BDD SC3 : "When 5 failures, Then 423"  ✅
Code    : if failCount > 5: lock()     ❌  ← > thay vì >=

Fix: đổi > thành >=
→ chạy BDD test → pass
→ notify: "Case A fixed, re-test FT-001-UC2-SC3"
```

> **Nếu root cause là lỗi AI hay lặp khi gen code** (vd: AI cứ dùng `>` thay vì `>=` cho boundary, hay gọi repository thẳng từ controller): `/fix-bug` sẽ hỏi *"Record as a project lesson? (Y/N)"* → chọn **Y**. Lesson được nạp vào mọi lần gen sau → AI không tái phạm. Hoặc chủ động: `/learn "AI hay X, đúng phải Y"`.

---

## Case 2 — BDD Bug (BDD ≠ PRD)

> Dev gen BDD sai từ đầu, hoặc PRD đã update nhưng BDD chưa.

```
Tester                    Dev                       PO/BA
──────                    ────────────              ──────
Gửi bug report
(chỉ rõ: BDD scenario
 mâu thuẫn với PRD AC)

                          Nhận bug report
                          Đọc PRD → BDD:
                          xác nhận BDD sai
                          
                          Nếu PRD rõ ràng:
                          → Update BDD scenario
                          → Update code nếu cần
                          → /run-tests → pass
                          → /validate-traces
                          → PR + notify tester
                          "BDD corrected + code updated
                           Re-test: {scenario}"
                          
                          Nếu PRD chưa đủ rõ:
                          → báo PO
                                                    Clarify PRD
                                                    (không đổi version
                                                     nếu chỉ là làm rõ)
                          Update BDD + code
                          → PR + notify

Nhận thông báo
Re-test
```

**Ví dụ thực tế:**

```
Bug: FT-001 — BDD ghi "3 lần sai → khoá", PRD ghi "5 lần"

PRD AC3 : "5 lần sai → khoá"           ✅
BDD SC3 : "When 3 failures, Then 423"   ❌  ← dev gen BDD sai
Code    : if failCount >= 3: lock()     ❌  ← code theo BDD sai

Fix:
1. Update BDD: 3 → 5
2. Update code: >= 3 → >= 5
3. Update test data
→ notify tester: "BDD + code fixed, re-test với 5 lần"
```

---

## Case 3 — PRD Ambiguity (PRD không rõ)

> Không ai sai — PRD viết thiếu, dev và tester hiểu khác nhau.

```
Tester                    Dev                       PO/BA
──────                    ────────────              ──────
Gửi bug report
(chỉ rõ: AC mơ hồ,
 dev hiểu X, tester
 expect Y)

                          Nhận bug report
                          Đọc PRD → xác nhận
                          AC chưa đủ rõ
                          KHÔNG tự fix code
                          Báo PO:
                          "AC3 mơ hồ:
                           dev assume 5 lần,
                           tester expect 3 lần.
                           Đúng là bao nhiêu?"
                                                    Quyết định: 5 lần
                                                    Update PRD AC3:
                                                    "5 lần liên tiếp"
                                                    Bump version nếu
                                                    thay đổi behavior
                                                    Notify dev + tester

                          Nhận PRD mới
                          Update BDD nếu cần
                          Update code nếu cần
                          /validate-traces
                          PR + notify tester

Nhận thông báo
Re-test theo PRD mới
```

**Ví dụ thực tế:**

```
PRD AC3: "Sai password nhiều lần → khoá tài khoản"  ← "nhiều lần" = bao nhiêu?

Dev assume: 5 lần
Tester expect: 3 lần (theo thông lệ ngân hàng)

→ Không phải bug code hay BDD — là PRD thiếu thông tin
→ PO quyết định: 5 lần (theo yêu cầu sản phẩm)
→ PO update: "Sai password 5 lần liên tiếp → khoá 30 phút"
→ BDD đã đúng → chỉ cần close bug, không fix code
```

---

## Case 4 — PRD Change (Yêu Cầu Thay Đổi)

> Không phải bug — business requirement thay đổi sau khi đã implement.

```
Tester                    Dev                       PO/BA
──────                    ────────────              ──────
Gửi bug report
(behavior thay đổi
 theo yêu cầu mới
 của business)

                          Nhận bug report
                          So sánh PRD version:
                          code theo v1.0,
                          tester expect v1.1
                          Báo PO xác nhận

                                                    Xác nhận: đây là
                                                    PRD change, không
                                                    phải bug
                                                    Update PRD v1.1
                                                    Notify dev + tester

                          Nhận PRD v1.1
                          /review-context → xem diff
                          Update BDD
                          Update code
                          Update tests
                          /validate-traces
                          PR + notify tester:
                          "Implemented per PRD v1.1
                           Re-test với behavior mới"

Nhận thông báo
Re-test theo PRD v1.1
```

**Lưu ý quan trọng:** Bug report phải được **re-classify** thành "PRD Change Request" trước khi xử lý — không đưa vào bug backlog.

---

## Case 5 — Design Spec Bug (UI ≠ Design Spec)

> Chỉ áp dụng cho FE/App. Code không khớp Design Spec.

```
Tester                    Dev (FE/App)              PO/BA + Designer
──────                    ────────────              ──────────────────
Gửi bug report
(chỉ rõ: screen X
 không khớp Design
 Spec section Y)

                          Nhận bug report
                          Đọc Design Spec
                          → xác nhận Design Spec
                            nói gì
                          
                          Nếu code sai Design Spec:
                          → fix code theo spec
                          → PR + notify tester
                          
                          Nếu Design Spec sai/lỗi thời:
                          → báo PO/Designer

                                                    Review Design Spec
                                                    Update nếu cần
                                                    Notify dev

                          Nhận Design Spec mới
                          Fix code
                          PR + notify tester

Re-test UI
```

---

## Case 6 — Environment / Data Bug

> Spec đúng hết, code đúng hết — lỗi ở infra hoặc test data.

```
Dấu hiệu nhận biết:
- Bug chỉ xảy ra trên staging, không reproduce local
- Bug xảy ra với 1 user cụ thể, không xảy ra với user khác  
- Bug xảy ra sau deploy, không xảy ra trước
- /run-tests pass nhưng manual test fail

Xử lý:
Tester → cung cấp: environment, user ID, timestamp, request ID
Dev    → check logs, config, database state
DevOps → check infra, env vars, deploy artifacts
```

---

## Giao Tiếp Hiệu Quả

### Format thông báo bug (Tester → Dev)

Tester chạy `/report-bug {UC-ID} {mô tả}` → tự sinh report theo format dưới (gồm cả phân loại layer + phát hiện coverage gap), commit+push vào **spec repo** `feedback/bug-reports/{BUG-ID}.md`. PO/Dev thấy khi chạy `/sync` (dòng `📥 New tester feedback`):

```
[BUG-{ID}] {Feature} — {mô tả ngắn}

Spec:  {PRD path} v{x.x} | AC{N}: "{AC text}"
BDD:   {BDD path} → "{Scenario title}"   (hoặc: ⚠️ no scenario covers this)
Layer: Code / BDD / PRD / Env  ← Tester đề xuất, Dev xác nhận

Expected: {theo spec}
Actual:   {thực tế}
Repro:    {steps}
Env:      staging / {date deploy}
```

> **Nếu Layer = "coverage gap"** (behavior đúng nhưng chưa scenario nào cover): tester chạy `/propose-scenario {UC-ID}` → draft scenario vào `bdd-proposals/` cho PO/Dev duyệt. Xem Case 2.

### Format thông báo fix xong (Dev → Tester)

```
[BUG-{ID}] Fixed ✅

Root cause: Case {A/B/C} — {mô tả ngắn}
Changed:
  - {file/component}: {what changed}
  - BDD: {updated / unchanged}
  - PRD: {unchanged / clarified by PO}

Deploy: staging @ {time} — {commit/PR link}
Re-test: {BDD scenario ID hoặc AC number}
```

### Format thông báo cần PO (Dev → PO)

```
[PRD-CLARIFY] {Feature} — AC{N} mơ hồ

Tình huống:
  Dev implement: {X}
  Tester expect: {Y}
  Triggered by: BUG-{ID}

PRD AC hiện tại:
  "{AC text}"

Câu hỏi:
  {câu hỏi cụ thể — 1 câu}

Cần trả lời trước: {date} để unblock tester
```

---

## Checklist Đóng Bug

Trước khi đánh "Resolved":

**Dev:**
- [ ] Root cause đã được xác định rõ (Case 1/2/3/4/5/6)
- [ ] Fix đúng layer — không patch code khi lỗi ở BDD hoặc PRD
- [ ] `/validate-traces` → no broken traces
- [ ] `/run-tests` → pass
- [ ] Nếu root cause là lỗi AI gen hay lặp → đã `/learn` (hoặc accept prompt khi `/fix-bug`) để không tái phạm
- [ ] Notify tester với đầy đủ thông tin re-test

**Tester:**
- [ ] Re-test đúng scenario được chỉ định
- [ ] Kiểm tra regression: các AC khác của cùng PRD không bị ảnh hưởng
- [ ] Confirm PASS trước khi close

**PO** *(nếu PRD được cập nhật)*:
- [ ] PRD version mới đã được bump
- [ ] Changelog có entry cho thay đổi
- [ ] Notify dev và tester về version mới
