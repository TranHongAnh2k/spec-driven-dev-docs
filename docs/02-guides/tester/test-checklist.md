[📚 Docs](../../README.md) › [Guides](../README.md) › [Tester](README.md) › Checklist Test Pass

# Checklist Trước Khi Báo "test pass"

**Spec & coverage:**
- [ ] Tất cả BDD scenarios đã được cover (happy path + edge + error)
- [ ] Mọi AC trong PRD đã được verify
- [ ] Manifest version khớp với PRD version đang deploy
- [ ] Không có scenarios bị skip mà không có lý do ghi chú

**Môi trường:**
- [ ] Test đã chạy trên đúng môi trường (staging, không phải local dev)
- [ ] Cross-service flows đã được verify (nếu feature span nhiều service)

**TSV & Living Docs:**
- [ ] `/qc-run-test` đã chạy → `qc_status` đã ghi vào `{service}/.trace/{UC-ID}.tsv`
- [ ] TSV đã được commit vào service submodule + umbrella pointer đã update + push
- [ ] `/validate-traces` (hoặc `/sync`) đã chạy → Living Docs panel hiển thị `qc_status: pass`
- [ ] Không còn scenario nào `qc_status: not_run` trong UC đang test
- [ ] Nếu có `qc_status: fail` → đã `/report-bug` kèm path evidence (`reports/<feature>/report.html`)

## Xem Thêm

- [Guide › QC Automation](../qc-automation.md) — pipeline `/qc-*` chi tiết, `qc_status`, stack `qc-playwright`
- [Operations › Bug Flow](../../04-operations/bug-flow.md) — flow phối hợp Tester ↔ Dev ↔ PO
- [Concepts › Traceability](../../03-concepts/traceability.md) — trace TSV, dev_selftest vs qc_status
- [Reference › Commands](../../05-reference/commands.md) — danh mục đầy đủ mọi command

---

← [Báo cáo bug](bug-reporting.md)
