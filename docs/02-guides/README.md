[📚 Docs](../README.md) › Guides

# Guides — Role Playbooks

Mỗi role trong framework spec-driven-dev có một guide riêng: vai trò, commands, workflow, và các tình huống thực tế. Bắt đầu từ guide khớp với role của bạn.

## Mục Lục

| Guide | Dành cho | Nội dung |
|---|---|---|
| [Product Owner / BA](product-owner/README.md) | PO, BA | Viết PRD platform-agnostic, generate Design Spec + BDD, handoff cho dev team |
| [Developer (FE / BE / App)](developer/README.md) | Dev | Đọc PRD + BDD từ spec submodule, generate tech-docs + code, dev self-check, trace system |
| [Tester / QA](tester/README.md) | QA, Tester | Spec-manifest, đọc spec chain, viết test cases, `/report-bug` + `/propose-scenario` |
| [QC Automation](qc-automation.md) | QC | Pipeline `/qc-*` 6 bước, `qc_status` chính thức, stack module `qc-playwright` |
| Designer | Designer, UX | Tham gia giai đoạn Design Spec — sign-off màn hình + component trước khi PO gen BDD. Chưa có guide riêng; xem [Product Owner › Design Spec](product-owner/scenarios.md#tình-huống-3--tạo-design-spec-và-bdd-sau-khi-prd-approved) để biết điểm giao. |

## Phân biệt nhanh hai luồng test

- **Dev self-check** — `/dev-gen-test` · `/dev-run-test` · `/dev-smoke-test`, ghi `dev_selftest`. Smoke check của riêng dev, KHÔNG phải coverage chính thức.
- **QC chính thức** — pipeline `/qc-analyze … /qc-report`, ghi `qc_status`. Bộ test authoritative, do QC chạy.

Hai tín hiệu này đứng cạnh nhau trong Living Docs và không ghi đè nhau.

## Xem thêm

- [Concepts › Traceability](../03-concepts/traceability.md) — trace chain PRD → BDD → Code
- [Reference › Commands](../05-reference/commands.md) — danh mục đầy đủ mọi command
