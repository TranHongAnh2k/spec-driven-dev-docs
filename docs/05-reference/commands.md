[📚 Docs](../README.md) › [Reference](README.md) › Command Reference

# Command Reference

> Mọi slash command của framework, gom theo phase. Mỗi command kèm **Input**, **Output**, và **When to use**. Phần cuối mô tả **Command Internals** — kiến trúc step file dùng chung bởi mọi command.

---

## Mục lục

- [Setup & Maintenance](#setup--maintenance)
- [Phase 1 — Discovery](#phase-1--discovery)
- [Phase 2 — PRD](#phase-2--prd)
- [Phase 2b — Design Spec (FE/App)](#phase-2b--design-spec-feapp)
- [Phase 3 — BDD Spec](#phase-3--bdd-spec)
- [Phase 4 — Tech Design](#phase-4--tech-design)
- [Phase 5 — Code](#phase-5--code)
- [Phase 6 — Dev Self-Check](#phase-6--dev-self-check)
- [Phase 6b — QC Automation (official QC suite)](#phase-6b--qc-automation-official-qc-suite)
- [Phase 7 — Traceability Audit](#phase-7--traceability-audit)
- [Phase 8 — Tester Feedback](#phase-8--tester-feedback)
- [Debugging & Cross-cutting](#debugging--cross-cutting)
- [Command Internals — Step Architecture](#command-internals--step-architecture)

> **Lưu ý số phase:** README dùng cả "Phase 5b / Phase 6b" (sơ đồ workflow) và "Phase 6b" (bảng tổng quan) cho QC Automation. Ở đây dùng nhãn **Phase 6b** cho QC suite để khớp bảng "Phase overview". Dù số khác nhau, ý nghĩa không đổi: QC suite là pipeline `/qc-*` riêng biệt với dev self-check `/dev-*`.

---

## Setup & Maintenance

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/setup-ai-first` | — | Project structure + config files | First-time project setup |
| `/sync` | — | Git pull + submodule update + Living Docs sync | **Daily driver** — đồng bộ *nội dung* project (code/specs trong submodule) |
| `/update-framework` | — | Refreshed `.agent/` command files from npm | **Occasionally** — nâng cấp *bản thân framework* khi có version mới |
| `/sync-figma-components {module}` | Module name | Updated `figma-components/{module}.md` | After Figma design system changes |
| `/sync-figma-tokens {url?}` | Optional Figma URL | Updated `figma-tokens.md` | After design token changes |

> **`/sync` vs `/update-framework` là hai thứ khác nhau:**
> - `/sync` cập nhật **nội dung project** — pull code/specs mới từ git remote + refresh Living Docs. Nguồn = git remotes của bạn. Chạy thường xuyên.
> - `/update-framework` cập nhật **framework tooling** — refresh `.agent/commands/`, `steps/`, `modules/` lên version mới nhất. Nguồn = npm registry. Chạy hiếm. Không bao giờ đụng `project-context.yaml`, `CLAUDE.md`, domain-knowledge, hay `.trace/`.

---

## Phase 1 — Discovery

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/define-product` | — | `specs/product-definition/{TICKET-ID}-{slug}.md` | Starting any new feature. AI dẫn PO qua 7 phase Q&A có cấu trúc — không cần spec viết sẵn. |

---

## Phase 2 — PRD

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/generate-prd` | product-definition file | `specs/prd/{domain}/{TICKET-ID}-{slug}.md` | After define-product. Thêm `API Source: existing` + section "Existing API Contract" cho feature brownfield. |
| `/refine-prd` | prd file | `.agent/review/{prd-slug}-findings.yaml` | After generate-prd — fan-out 4 review lens + completeness-critic loop, mở Review Board. |
| `/refine-prd --resume` | prd file | Applied findings + bumped version | Sau khi review trong Review Board. |
| `/review-context` (PRD) | prd file | `.agent/review/{prd-slug}-review-context-findings.yaml` | Quality gate bắt buộc trước Phase 3. Checks: banned terms (P1), ambiguity (P2), conflicts (P3), completeness (P4). |
| `/review-context --fix` | prd file | Applies all auto-fixable findings immediately | Dev quick-fix, không cần Review Board. |
| `/review-context --resume` | prd file | Applies accepted findings + bump version | PO/SA review — human quyết từng finding. |

---

## Phase 2b — Design Spec (FE/App)

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/generate-design-spec` | prd file | `specs/design-spec/{domain}/{TICKET-ID}-design-spec-{platform}.md` | FE/App: sau khi PRD approved, trước BDD. PO phải cấp **node-level Figma frame link** (URL có `?node-id=`) cho mỗi screen; AI fetch từng frame qua Figma MCP. Screen thiếu link → ❌ Missing, spec ở draft và chặn sign-off + `/generate-bdd`. |

> BE teams bỏ qua Phase 2b — đọc PRD trực tiếp rồi `/generate-bdd`.

---

## Phase 3 — BDD Spec

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/generate-bdd` | prd file | `specs/bdd/{domain}/{UC-ID}.feature` (multi-service: `{domain}/{service}/{UC-ID}.feature`) | After PRD approved (+ Design Spec sign-off cho FE/App). Đọc Service/Module từ PRD metadata, áp platform vocabulary, hiện SC outline chờ confirm. PRD lớn (>3 UC hoặc >300 dòng) → orchestration mode (1 sub-agent / UC). |
| `/review-context` (BDD) | feature file | `.agent/review/{uc-id}-review-bdd-findings.yaml` | Quality gate bắt buộc trước Phase 4. Checks: PRD coverage (mỗi AC + BR → ≥1 SC), Gherkin R1–R10, compliance C1–C5. |
| `/review-context --fix` | feature file | Auto-fix terminology, metadata, coverage matrix | Dev quick-fix. |
| `/review-context --resume` | feature file | Applies accepted findings + bump bdd_version | SA review. |

---

## Phase 4 — Tech Design

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/generate-tech-docs` | feature file | `tech-docs/{domain}/{UC-ID}-tech-design.md` (multi-service: `{domain}/{service}/...`) | After BDD approved. Đọc `@trace.module` → chọn template theo platform type. Brownfield (`@trace.api_source: existing`) → reverse-document mode. |
| `/review-tech-docs` | tech-design file | `.agent/review/{uc-id}-tech-review-findings.yaml` | After generate-tech-docs — mở Review Board. |
| `/review-tech-docs --resume` | tech-design file | Applies accepted findings + bump revision | Sau khi review trong Review Board. |

---

## Phase 5 — Code

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/generate-code` | feature file | `src/...` (tags `@trace.implements`) | After tech-design approved. Default = full impl/BE. |
| `/generate-code --phase=ui` | feature file | UI + mock API adapter | FE: BE chưa sẵn sàng — sinh UI + mock adapter từ System BDD contract; tester test FE ngay được. |
| `/generate-code --phase=integration` | feature file | Wires real API | FE: sau khi T7 sign-off gate approved — thay mock adapter bằng real API calls. |
| `/review-code` | — | Review report: Critical / Major / Minor | After generate-code. Check: mỗi scenario có endpoint, trace tags, layer rules, error handling khớp CLAUDE.md §5. Fix CRITICAL + MAJOR trước khi merge. |

---

## Phase 6 — Dev Self-Check

> Đây là **developer self-check / smoke** — dev tự kiểm tra code mình vừa sinh ra. **Không phải** official QC suite (đó là Phase 6b `/qc-*`). Signal = `dev_selftest`, tách biệt với `qc_status`.

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/dev-gen-test` | feature file | `src/test/...` (tags `@trace.verifies`) | After generate-code. Tự chọn test framework đúng platform. |
| `/dev-run-test` | — | Self-check report — ghi `dev_selftest` (pass/fail/not_run) + `dev_selftest_at` vào trace TSV | After dev-gen-test. |
| `/dev-smoke-test` | — | Live endpoint check | Optional — sau deploy. |

---

## Phase 6b — QC Automation (official QC suite)

> Pipeline `/qc-*` **là** official QC suite. Chạy **theo thứ tự** sau khi BDD approved (`/review-context` (BDD) APPROVED). Tách biệt hoàn toàn với dev self-check: QC sở hữu `qc_status`, dev sở hữu `dev_selftest`. `/qc-run-test` & `/qc-report` dùng module `qc-playwright`, độc lập với dev implementation module. Xem [../02-guides/qc-automation.md](../02-guides/qc-automation.md).

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/qc-analyze` | feature file / UC-ID | QC analysis of scenarios + risks | After BDD approved — đầu pipeline QC. |
| `/qc-plan` | QC analysis | Test plan (scope, layers, priorities) | After `/qc-analyze`. |
| `/qc-design-test` | test plan | Test case designs mapped to `{UC-ID}-SC{N}` | After `/qc-plan`. |
| `/qc-review` | test designs | Reviewed/approved test design set | After `/qc-design-test`. |
| `/qc-run-test` | test designs | Execution results — ghi `qc_status` (pass/fail/skip/not_run) + `qc_run_at` vào trace TSV, keyed by `@trace.verifies={UC-ID}-SC{N}` | After `/qc-review`. |
| `/qc-report` | run results | QC report (Playwright Trace + pytest-html) | After `/qc-run-test`. |

---

## Phase 7 — Traceability Audit

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/validate-traces {domain}` | domain name (hoặc UC-ID) | Coverage matrix + drift report + `trace-report.json` | Anytime — verify traceability. Đọc mọi `.trace/{UC-ID}.tsv` rồi tính status OK / DRIFT / GAP / UNTRACKED. |

---

## Phase 8 — Tester Feedback

> Cả hai đều **read-only trên canonical specs/code**. Chúng chỉ ghi vào `feedback/` của shared spec repo và **commit + push** → PO/Dev thấy qua `/sync`. Testers không bao giờ sửa `.feature` trực tiếp.

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/report-bug {UC-ID} {desc}` | UC/TICKET + description | `{spec_repo}/feedback/bug-reports/{BUG-ID}.md` + spec context + layer classification | Tester tìm thấy bug. Phân loại layer (Code/BDD/PRD/Design/Env) để route. |
| `/propose-scenario {UC-ID} {desc}` | UC + edge-case description | `{spec_repo}/feedback/bdd-proposals/{UC-ID}-*.md` draft Gherkin | Tester tìm thấy edge case BDD chưa cover. Map vào PRD AC sẵn có, hoặc emit PRD change request nếu hành vi thực sự mới. |

---

## Debugging & Cross-cutting

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/fix-bug` | — | Bug fix | When a bug is found. |
| `/debug` | — | Debug session | Deep debugging. |
| `/learn {text}` | "AI does X, should Y" | Appends guardrail vào project lessons file | Khi AI lặp lại lỗi bạn muốn chặn. |
| `/generate-spec-manifest` | — | `spec-manifest.yaml` | Sinh manifest làm entry point cho external/tester agents. |

> **Project Lessons:** lessons lưu ở `paths.lessons_file` (default `specs/domain-knowledge/lessons-learned.md`; per-service `.agent/project-lessons.md` trong umbrella mode). Context-loader **Step 6.7** load mọi lesson đầu **mỗi** command như hard constraint. Đây là *project memory*, không phải fine-tuning model.

---

## Command Internals — Step Architecture

Mọi command trong `.agent/commands/` được compose từ **step file** dùng chung trong `.agent/steps/`. Các step này là nền móng — mọi command gọi chúng tuần tự trước khi chạy logic riêng.

### Step files at a glance

| File | Role | Called by |
|------|------|-----------|
| `gate.md` | Universal entry — model check, file resolution, context load, user checkpoint | Every command |
| `context-loader.md` | Multi-step context loading (stack → service routing → service conventions → arch → safety → domain → UI → recap) | `gate.md` Step 2 |
| `spawn-agent.md` | Sub-agent orchestration cho PRD lớn | `generate-bdd`, `generate-code`, `dev-gen-test` |
| `capture-lesson.md` | Append/refine guardrail vào project lessons file | `learn`, `review-code`, `fix-bug`, `debug` |
| `report-footer.md` | Standard output format (status badge, artifact list, next command) | Every command |

### gate.md — Universal Entry Procedure

| Step | What it does |
|------|-------------|
| **Step 0** | Sub-agent mode check — nếu `$ARGUMENTS` là JSON payload có `_agent_mode: true`, skip Steps 1–3 và dùng slim context trực tiếp |
| **Step 0-B** | Model check — prompt chuyển sang `claude-opus`; `S` để skip |
| **Step 1** | Resolve target file từ `$ARGUMENTS`; nếu thiếu, list candidates và hỏi user |
| **Step 2** | Execute `context-loader.md` — load mọi project context vào working memory |
| **Step 3** | CHECKPOINT — hiện summary (target file, stack, module, domains), chờ `Y` |

### context-loader.md — Context Loading Sequence

Load context theo thứ tự ưu tiên nghiêm ngặt (anti-lost-in-middle: facts critical load trước, RECAP restate cuối):

| Step | Priority | What loads |
|------|----------|-----------|
| 1 | PROJECT-CONFIG | `project-context.yaml` → stack, conventions, domains, services, paths |
| 1.5 | SERVICE ROUTING | *(umbrella only)* detect active domain, route `specs_dir`/`tech_docs_dir` → service submodule, store `service_root` |
| 1.6 | SERVICE CONVENTIONS | *(umbrella only)* load `{service_root}/.agent/project-context.yaml` → override `test_command`, `build_command`, `paths.trace_dir` |
| 2 | PROJECT-CONFIG | `.agent/modules/{module}/stack-profile.yaml` → framework-specific layer patterns |
| 3 | **CRITICAL** | `CLAUDE.md` → architecture layers, coding standards, naming |
| 4 | SAFETY | `.agent/rules/data-protection.md` → sensitive file patterns |
| 5 | DOMAIN | `business-dictionary.md` → canonical + banned terms |
| 6 | DOMAIN | `core-entities.md` → entity catalog |
| 6.7 | **GUARDRAILS** | `lessons_file` → accumulated lessons (via `/learn`) as hard constraints |
| 6-B | UI COMPONENTS | `figma-components/{module}.md` → Figma → code component + import path |
| 6-C | UI TOKENS | `figma-tokens.md` → design tokens |
| 7 | **RECAP** | `[CTX LOADED]` block printed để khóa critical facts vào working memory |

Status values của RECAP: `FULL` · `PARTIAL — missing: {list}` · `MINIMAL` (chỉ project-context.yaml).

### spawn-agent.md — Sub-Agent Orchestration

Khi PRD vượt ngưỡng complexity, `generate-bdd` / `generate-code` / `dev-gen-test` tự chuyển từ single-session sang orchestration mode.

| Signal | Threshold | Action |
|--------|-----------|--------|
| UC count in PRD | > 3 UCs | Spawn 1 sub-agent per UC |
| PRD length | > 300 lines | Spawn agents regardless of UC count |

Orchestrator: Step A build slim context JSON → Step B scan PRD → UC list → Step C announce plan → Step D spawn 1 sub-agent/UC (parallel, mỗi agent chỉ đọc UC section của mình) → Step E collect + merge results.

### report-footer.md — Standard Output Format

Mọi command kết thúc bằng footer: `Status` badge (`✅ Complete` · `⚠️ Warnings` · `❌ Failed`), danh sách **Output Artifacts** (created/updated), và field **Next** gợi ý command kế tiếp.

---

*Xem thêm:* [Trace TSV Schema](trace-schema.md) · [Stack Modules](modules.md) · [02 · Guides](../02-guides/) (theo vai trò).
