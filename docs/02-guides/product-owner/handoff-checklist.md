[📚 Docs](../../README.md) › [Guides](../README.md) › [Product Owner](README.md) › Checklist Handoff

# Checklist Handoff Cho Dev Team

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
- [ ] Design Spec đã tạo và `@trace.status: approved` (không còn `draft`)
- [ ] Mỗi screen có Figma link node-level (`?node-id=`) — không screen nào bị flag ❌ Missing
- [ ] Designer đã sign-off (gate này bị BLOCKED nếu còn screen thiếu node-id link)

**BDD (bắt buộc trước khi handoff):**
- [ ] BDD web đã gen: `specs/bdd/{domain}/web/{TICKET-ID}-UC*.feature`
- [ ] BDD app đã gen: `specs/bdd/{domain}/app/{TICKET-ID}-UC*.feature` (nếu project có app)
- [ ] System BDD đã gen: `specs/bdd/{domain}/system/{TICKET-ID}-UC*.feature`
- [ ] Tất cả BDD files đã được review và không có `MISSING` trong Coverage Matrix

**Git:**
- [ ] Đã commit và push toàn bộ (`specs/prd/`, `specs/design-spec/`, `specs/bdd/`) lên spec repo
- [ ] Thông báo dev team domain name và BDD path để họ cập nhật `git submodule update --remote`

---

← [Quy tắc viết PRD](prd-writing-rules.md)
