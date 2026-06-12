[📚 Docs](../../README.md) › [Guides](../README.md) › [Tester](README.md) › Workflow

# Workflow Cơ Bản

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
→ ghi lại paths (tất cả đều nằm trong submodule):
    prd:          {spec_source}/specs/prd/{domain}/FT-042-*.md
    bdd.be:       {service-be}/specs/bdd/{domain}/FT-042-*.feature
    bdd.web:      {service-web}/specs/bdd/{domain}/FT-042-*.feature
    tech_docs.be: {spec_source}/specs/tech-docs/{domain}/FT-042-*.md
        │
        ▼
Đọc PRD tại path manifest.prd
→ file nằm trong SPEC submodule (shared — PO viết)
→ hiểu AC, UC, BR — đây là "ground truth"
        │
        ▼
Đọc BDD tại path manifest.bdd.be / bdd.web
→ file nằm trong SERVICE submodule (be/web/app — PO gen từ PRD)
→ mỗi Scenario = 1 test case cần cover
→ BE có BDD riêng (system/), Web có BDD riêng (web/), App có BDD riêng (app/)
        │
        ▼
Đọc Tech Docs tại path manifest.tech_docs.be / tech_docs.web
→ file nằm trong SPEC submodule (shared — Dev gen, SA sign-off)
→ BE: API endpoints, request/response schema, error codes
→ Web: screen flow, component states
        │
        ▼
Chạy QC automation pipeline (6 bước — ghi kết quả chính thức vào TSV)
        │
        ├─ /qc-analyze {UC-ID}       → bóc tách spec, xác định scope, phát hiện DOC_GAPS
        ├─ /qc-plan {UC-ID}          → risk + test plan + questions-for-dev
        ├─ /qc-design-test {UC-ID}   → thiết kế test case chi tiết (.Test.md)
        ├─ /qc-review {UC-ID}        → review test design trước khi chạy
        ├─ /qc-run-test {UC-ID}      → sinh + chạy pytest-playwright
        │                               → ghi qc_status (pass/fail/skip) per scenario
        │                               → vào {service}/.trace/{UC-ID}.tsv  ← AUTHORITATIVE
        └─ /qc-report {UC-ID}        → report pytest-html + Playwright trace evidence
                                        → reports/<feature>/report.html  ← LOCAL, gitignored
        │
        ▼
Commit TSV vào service submodule + update umbrella pointer
  cd {service}
  git add .trace/{UC-ID}.tsv
  git commit -m "qc: record qc_status for {UC-ID} — pass/fail"
  git push
  cd ..   ← umbrella root
  git add {service}
  git commit -m "chore: update {service} submodule pointer — {UC-ID} qc_status"
  git push
        │
        ▼
/validate-traces (hoặc /sync)
→ tổng hợp TSVs từ tất cả services → Living Docs cập nhật cột qc_status
        │
        ▼
Kết quả:
  qc_status: pass  → done, Living Docs phản ánh coverage chính thức
  qc_status: fail  → /report-bug với BDD scenario + AC bị vi phạm
                      đính kèm path evidence: reports/<feature>/report.html
                      (evidence local — share file hoặc upload riêng nếu cần)
```

---

← [Spec Manifest & Setup](spec-manifest.md) · Tiếp theo: [Đọc Spec Chain](reading-specs.md)
