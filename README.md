# Spec-Driven Dev Plugin

An **AI-First Spec-Driven Development** workflow plugin for Claude Code.

Guides your team through a complete feature lifecycle —
**Discovery → PRD → Design-Spec → BDD → Tech Design → Code → Dev self-check → QC automation** —
with AI review gates at every transition and end-to-end traceability.

---

## 📚 Documentation

Toàn bộ tài liệu nằm trong **[`docs/`](docs/README.md)**, chia theo **vai trò** và **chủ đề**:

| Bạn là… | Bắt đầu ở |
|---------|-----------|
| Người mới / cài đặt lần đầu | [01 · Getting Started](docs/01-getting-started/) |
| Product Owner / BA | [Guide · Product Owner](docs/02-guides/product-owner.md) |
| Developer (FE / BE / App) | [Guide · Developer](docs/02-guides/developer.md) |
| Tester / QA | [Guide · Tester](docs/02-guides/tester.md) |
| QC Automation | [Guide · QC Automation](docs/02-guides/qc-automation.md) |
| Muốn hiểu kiến trúc | [03 · Concepts](docs/03-concepts/) |
| Vận hành / admin | [04 · Operations](docs/04-operations/) |
| Tra cứu lệnh / schema / module | [05 · Reference](docs/05-reference/) |

→ **[Mở mục lục đầy đủ](docs/README.md)**

---

## Quick Start

Requires [Node.js](https://nodejs.org) (check: `node -v`).

```bash
# 1. Install framework
#    Single-service:
npx @anhth2/spec-driven-dev-plugin --init --module java-spring
#    Multi-service monorepo:
npx @anhth2/spec-driven-dev-plugin --init \
  --services backend:java-spring,web-admin:react,app-mobile:flutter

# 2. Open the project in Claude Code, then:
/setup-ai-first

# 3. Fill in the generated config (CLAUDE.md, .agent/project-context.yaml,
#    specs/domain-knowledge/*), then start your first feature:
/define-product
```

Full install paths (upgrade, umbrella, uninstall) → **[Getting Started › Installation](docs/01-getting-started/installation.md)**.
First feature end-to-end → **[Getting Started › Quick Start](docs/01-getting-started/quickstart.md)**.

---

## Workflow at a glance

```
Phase 1  Discovery        /define-product
Phase 2  PRD              /generate-prd · /refine-prd · /review-context
Phase 2b Design Spec      /generate-design-spec            (FE/App only)
Phase 3  BDD              /generate-bdd · /review-context
Phase 4  Tech Design      /generate-tech-docs · /review-tech-docs
Phase 5  Code             /generate-code · /review-code
Phase 6  Dev self-check   /dev-gen-test · /dev-run-test · /dev-smoke-test   → dev_selftest
Phase 6b QC automation    /qc-analyze → /qc-plan → /qc-design-test
                          → /qc-review → /qc-run-test → /qc-report          → qc_status
Phase 7  Trace            /validate-traces
Phase 8  Tester feedback  /report-bug · /propose-scenario → /sync
Cross-cutting             /sync · /learn · /update-framework
```

Pipeline chi tiết → **[Concepts › Pipeline](docs/03-concepts/pipeline.md)**.
Bảng đầy đủ mọi command → **[Reference › Commands](docs/05-reference/commands.md)**.

---

## Philosophy

> **Write the spec first. Generate the code from the spec. Trace everything.**

- Humans define *what* — acceptance criteria, business rules, platform requirements.
- AI generates *how* — BDD scenarios, tech design, code, tests — adapted to the platform.
- Every artifact is reviewed before proceeding to the next phase.
- Every line of code traces back to a scenario in a `.feature` file.
- In multi-service projects, each service evolves independently while sharing the same workflow.

---

*Framework version: see `core/FRAMEWORK_VERSION`. Build internals & directory map → [Concepts › Architecture](docs/03-concepts/architecture.md).*
