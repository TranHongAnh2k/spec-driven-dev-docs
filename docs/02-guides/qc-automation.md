[📚 Docs](../README.md) › [Guides](README.md) › QC Automation

# Hướng Dẫn QC Automation — Spec-Driven Dev

Pipeline QC automation **chính thức** của framework — được port từ agent của team QC (reference repo `ai-automation-qc-base`) vào ngay trong framework. Nó sinh và chạy automation **Playwright/pytest** từ BDD `.feature` chính thức, rồi ghi tín hiệu **`qc_status`** (authoritative) vào trace TSV → Living Docs.

## Mục Lục

- [Hai luồng test: dev self-check vs QC chính thức](#hai-luồng-test-dev-self-check-vs-qc-chính-thức)
- [Pipeline 6 bước](#pipeline-6-bước)
- [Stack — module qc-playwright](#stack--module-qc-playwright)
- [Trace join: qc_status đến Living Docs như thế nào](#trace-join-qc_status-đến-living-docs-như-thế-nào)
- [Entry point](#entry-point)
- [Skills theo layer](#skills-theo-layer)

## Hai Luồng Test: Dev Self-Check vs QC Chính Thức

Pipeline QC này **khác** với developer **self-check** (`/dev-gen-test`, `/dev-run-test`, `/dev-smoke-test`):

| Signal | Owner | Command | Ý nghĩa |
|--------|-------|---------|---------|
| `dev_selftest` | Dev | `/dev-run-test` | smoke check của riêng dev (KHÔNG authoritative) |
| `qc_status` | QC | `/qc-run-test` | kết quả QC automation **chính thức** |

Cả hai hiển thị **cạnh nhau** trong Living Docs; không cái nào ghi đè cái nào. `dev_selftest: pass` chỉ nghĩa "dev đã tự smoke qua"; coverage chính thức nằm ở `qc_status`.

## Pipeline 6 Bước

```
/qc-analyze → /qc-plan → /qc-design-test → /qc-review → /qc-run-test → /qc-report
 (requirement   (risk/plan   (test-case      (review     (gen+run     (report +
  breakdown)     + Q-for-dev)  .Test.md)       gate)       Python →     evidence)
                                                            qc_status)
```

| Command | Vai trò | Output |
|---------|---------|--------|
| `/qc-analyze` | bóc tách spec chính thức thành requirement + BR/AC + data-flow + `DOC_GAPS` | `{refinement_dir}/qc/{UC-ID}/` |
| `/qc-plan` | risk / what-if / questions-for-dev + test plan | `TEST_PLAN.md` |
| `/qc-design-test` | thiết kế test case (Markdown `.Test.md`), tag `@trace.verifies={UC-ID}-SC{N}` | `test-cases/` |
| `/qc-review` | cổng review hai chiều: review test case (sau design) và script (sau run) | findings |
| `/qc-run-test` | sinh + chạy Python pytest-playwright; **ghi `qc_status`** per scenario | `{trace_dir}/{UC-ID}.tsv` |
| `/qc-report` | report pytest-html + Playwright trace + evidence | `reports/<feature>/report.html` |

## Stack — Module qc-playwright

`/qc-run-test` và `/qc-report` dùng stack module `qc-playwright`
(`.agent/modules/qc-playwright/stack-profile.yaml`): **Python + pytest-playwright + Page
Object** (slim `BasePage`, 3-layer), **Playwright Trace + pytest-html** (KHÔNG Allure). Stack
này độc lập với module implementation của dev (java-spring, react, flutter, …).

Per-layer guides chi tiết nằm trong `skills/qc/<stage>/` (port từ skills của agent QC):
functional / integration / e2e / non-functional / exploratory.

## Trace Join: qc_status Đến Living Docs Như Thế Nào

1. `.feature` chính thức định nghĩa scenario là `@trace.scenario={UC-ID}-SC{N}`.
2. `/qc-design-test` ghi SC mà test case thuộc về; `/qc-run-test` tag mỗi pytest test
   `# @trace.verifies={UC-ID}-SC{N}`.
3. `/qc-run-test` ghi `qc_status` (`pass|fail|skip|not_run`) + `qc_run_at` vào trace TSV của
   service (authoritative), rồi refresh panel mirror local.
4. `/validate-traces` (hoặc `/sync`) tổng hợp các TSV thành report Living Docs tại
   `{spec_source}/.living-docs/` — dashboard hiển thị `qc_status` cạnh `dev_selftest`.

## Entry Point

Pipeline QC bắt đầu ngay khi BDD của một UC được approved (`/review-context (BDD)` APPROVED):

```
/qc-analyze {UC-ID}   →  … → /qc-report {UC-ID}  →  /validate-traces {UC-ID}
```

## Skills Theo Layer

Các skill chi tiết theo từng giai đoạn QC nằm trong `skills/qc/`:

| Stage skill | Vai trò |
|---|---|
| `qa-analyst` | phân tích requirement, sinh `DOC_GAPS` |
| `qa-planner` | test plan, risk, questions-for-dev |
| `qa-designer` | thiết kế test case `.Test.md` theo layer (functional / integration / e2e / non-functional / exploratory) |
| `qa-reviewer` | cổng review test case và script |
| `qa-runner` | sinh + chạy Python pytest-playwright, sinh report (Playwright Trace + pytest-html) |

Mỗi skill **tự chứa** (self-contained) và có version frontmatter để theo dõi khi nâng cấp.

## Xem Thêm

- [Guide › Tester](tester/README.md) — vai trò tester, spec-manifest, `/report-bug`, `/propose-scenario`
- [Concepts › Traceability](../03-concepts/traceability.md) — trace TSV, dev_selftest vs qc_status
- [Reference › Modules](../05-reference/modules.md) — chi tiết module `qc-playwright`
- [Reference › Commands](../05-reference/commands.md) — danh mục đầy đủ mọi command
