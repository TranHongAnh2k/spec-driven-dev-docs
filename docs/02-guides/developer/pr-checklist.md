[📚 Docs](../../README.md) › [Guides](../README.md) › [Developer](README.md) › PR Checklist

# Checklist Trước Khi Tạo PR

- [ ] `/validate-traces` → all green (không broken trace)
- [ ] `/dev-run-test` → all pass *(umbrella: đảm bảo service có `.agent/project-context.yaml` với `test_command` trước khi chạy)*
- [ ] `/review-code` → không có issue Critical hoặc Major chưa xử lý
- [ ] Code trace về đúng BDD scenarios trong `my-project-specs/specs/bdd/`
- [ ] Code có `@trace.bdd` comment cho các function implement BDD scenario
- [ ] Tech Docs đã được update nếu có thay đổi API/DB schema
- [ ] **Không tự sửa BDD** — BDD là của PO, nếu cần update thì báo PO rồi pull lại

---

← [Tình huống thực tế](scenarios.md)
