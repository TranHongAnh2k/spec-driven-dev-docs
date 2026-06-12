[📚 Docs](../../README.md) › [Guides](../README.md) › Developer

# Hướng Dẫn Developer — Spec-Driven Dev

Tài liệu dành cho **Developer (FE / BE / App)** — vai trò, commands, trace system, workflow, và các tình huống thực tế. Được chia nhỏ theo chủ đề để dễ đọc:

## Mục Lục

| Trang | Nội dung |
|---|---|
| [Commands](commands.md) | Bảng lệnh cho dev · project lessons · xử lý feedback tester · khi nào dùng `--phase` |
| [BDD & Trace System](bdd-and-trace.md) | Tại sao BDD quan trọng với dev · `@trace.*` fields · trace chain · khi nào `/validate-traces` |
| [Workflow](workflow.md) | Luồng làm việc cơ bản từ nhận PRD đến tạo PR |
| [Tình huống thực tế](scenarios.md) | 8 scenario: nhận PRD mới, đọc System/Web BDD, PRD đổi, API sign-off, bug từ tester, design spec, brownfield, umbrella, validate-traces |
| [Checklist trước khi tạo PR](pr-checklist.md) | Checklist verify trước khi mở PR |

## Vai Trò Dev Trong Framework

```
PO/BA                     Dev
──────────────────────    ──────────────────────────────────────
/define-product           /review-context (đọc PRD + BDD)
/generate-prd        →    đọc BDD từ spec submodule
/refine-prd               /generate-tech-docs (từ BDD → Tech Docs)
/review-context           /generate-code     (từ BDD + Tech Docs → Code)
/generate-design-spec →   /dev-gen-test
/generate-bdd (web)       /review-code
/generate-bdd (app)       /dev-run-test
/generate-bdd (system)    /fix-bug / /debug
                          /validate-traces
```

**Dev chịu trách nhiệm:**
- Đọc và hiểu PRD + BDD từ spec submodule trước khi bắt đầu
- **KHÔNG tự generate BDD** — BDD đã được PO generate trong spec repo
- Đảm bảo code trace về đúng BDD scenario, BDD trace về đúng PRD
- Báo PO/BA khi PRD hoặc BDD có gì không rõ hoặc mâu thuẫn — không tự suy diễn

**Dev KHÔNG làm:**
- Viết/sửa PRD — đó là việc của PO/BA
- Viết/sửa Design Spec — đó là việc của PO/BA + Designer
- Approve PRD — chỉ PO mới có quyền này

---

*Xem thêm:* [Product Owner Guide](../product-owner/README.md) · [Tester Guide](../tester/README.md) · [QC Automation Guide](../qc-automation.md) · [Concepts › Traceability](../../03-concepts/traceability.md) · [Reference › Commands](../../05-reference/commands.md)
