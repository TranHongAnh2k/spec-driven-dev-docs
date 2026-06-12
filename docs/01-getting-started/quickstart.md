[📚 Docs](../README.md) › [Getting Started](README.md) › Quick Start

# Quick Start

Chạy feature đầu tiên end-to-end theo happy path. Mở Claude Code tại root project và làm theo thứ tự.

## Mục lục

- [Bước 0 — Setup project (một lần)](#bước-0--setup-project-một-lần)
- [Happy-path command sequence](#happy-path-command-sequence)
- [Bước tiếp theo](#bước-tiếp-theo)

## Bước 0 — Setup project (một lần)

```bash
# 1. Cài framework (nếu chưa) — xem installation.md
npx @anhth2/spec-driven-dev-plugin --init

# 2. Mở project trong Claude Code, tạo cấu trúc + config files:
/setup-ai-first
```

`/setup-ai-first` tạo cấu trúc thư mục (`specs/`, `tech-docs/`, `.trace/`, `.agent/`, `CLAUDE.md`). Sau đó điền thông tin thực tế vào 4 file config:

| File | Nội dung |
|------|----------|
| `CLAUDE.md` | Architecture layers, coding standards, git conventions |
| `.agent/project-context.yaml` | Tech stack, services, paths, ticket prefix |
| `specs/domain-knowledge/business-dictionary.md` | Canonical terms, banned terms |
| `specs/domain-knowledge/core-entities.md` | Entity glossary (fields, relationships) |

> Project đã có sẵn `specs/`, `CLAUDE.md`, `.agent/project-context.yaml`? Bỏ qua bước này, vào thẳng happy path bên dưới. Chi tiết điền config + Figma setup: xem [../02-guides](../02-guides) và [../03-concepts](../03-concepts).

## Happy-path command sequence

Discovery → PRD → BDD → Tech Design → Code → Dev self-check:

```
# PHASE 1 — DISCOVERY
/define-product
    → specs/product-definition/{slug}.md

# PHASE 2 — PRD
/generate-prd specs/product-definition/{slug}.md
    → specs/prd/{domain}/{slug}.md
/refine-prd specs/prd/{domain}/{slug}.md         # AI suggestions → Review Board
/refine-prd --resume specs/prd/{domain}/{slug}.md  # apply + bump version
/review-context specs/prd/{domain}/{slug}.md      # quality gate (P1–P4)
/review-context --resume specs/prd/{domain}/{slug}.md
    → ✅ APPROVED → tiếp Phase 3

# PHASE 3 — SPEC & DESIGN
# (FE/App only) /generate-design-spec specs/prd/{domain}/{slug}.md → designer + PO sign-off
/generate-bdd specs/prd/{domain}/{slug}.md
    → specs/bdd/{domain}/{UC-ID}.feature
/review-context specs/bdd/{domain}/{UC-ID}.feature
/review-context --resume specs/bdd/{domain}/{UC-ID}.feature   # apply + bump bdd_version
/generate-tech-docs specs/bdd/{domain}/{UC-ID}.feature
    → tech-docs/{domain}/{UC-ID}-tech-design.md
/review-tech-docs tech-docs/{domain}/{UC-ID}-tech-design.md
/review-tech-docs --resume tech-docs/{domain}/{UC-ID}-tech-design.md

# PHASE 4 — CODE
/generate-code specs/bdd/{domain}/{UC-ID}.feature
    → src/...  (@trace.implements tags)
/review-code                                       # fix CRITICAL / MAJOR

# PHASE 5 — DEV SELF-CHECK (dev verify code của mình — KHÔNG phải QC suite chính thức)
/dev-gen-test specs/bdd/{domain}/{UC-ID}.feature
    → src/test/...  (@trace.verifies tags)
/dev-run-test                                      # sets dev_selftest in trace
/dev-smoke-test                                    # optional — live endpoint check
/validate-traces {domain}                          # coverage & drift
```

Mỗi review command ghi findings vào `.agent/review/*-findings.yaml`. Mở **Review Board** (VS Code panel) → Accept / Modify / Defer / Reject từng finding → chạy `--resume` để apply. Quick-fix không qua Review Board: `/review-context --fix {file}` (chỉ apply auto-fixable).

> **FE/App:** `/generate-code --phase=ui` (UI + mock adapter, tester test ngay) rồi `--phase=integration` (wire API thật sau sign-off).

## Bước tiếp theo

- Hiểu khái niệm phía sau pipeline → [core-concepts.md](core-concepts.md).
- QC suite chính thức (`/qc-analyze → /qc-plan → /qc-design-test → /qc-review → /qc-run-test → /qc-report`) → [../02-guides/qc-automation.md](../02-guides/qc-automation.md).
- Role guides, scenarios thực tế, multi-repo/umbrella → [../02-guides](../02-guides) và [../03-concepts](../03-concepts).
- Full command reference → [../05-reference](../05-reference).
