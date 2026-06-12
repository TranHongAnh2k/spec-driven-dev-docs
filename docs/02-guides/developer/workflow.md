[📚 Docs](../../README.md) › [Guides](../README.md) › [Developer](README.md) › Workflow

# Workflow Cơ Bản

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
/dev-gen-test {bdd-file}
→ Gen unit test (dev self-check, không phải coverage chính thức)
→ /dev-run-test để verify → ghi dev_selftest (pass/fail) + dev_selftest_at vào trace TSV
→ /validate-traces (hoặc /sync) để push trace lên spec-module Living Docs ({spec_source}/.living-docs/)
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

← [BDD & Trace System](bdd-and-trace.md) · Tiếp theo: [Tình huống thực tế](scenarios.md)
