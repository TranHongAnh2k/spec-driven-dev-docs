[📚 Docs](../README.md) › [Concepts](README.md) › Architecture

# Architecture

> **Nguyên tắc**: Đọc file này để hiểu toàn bộ framework trước khi đọc bất kỳ file chi tiết nào.

Cách framework được xây dựng: 6 lớp runtime, build pipeline `*.tmpl → *.md → core/`, module plug-in system, hook data-protection, và sub-agent orchestration.

## Mục lục

- [Layer diagram (L0 → L5)](#layer-diagram-l0--l5)
- [Data flow — một command chạy như thế nào](#data-flow--một-command-chạy-như-thế-nào)
- [Build system](#build-system)
- [Module plug-in system](#module-plug-in-system)
- [Hook system — data protection](#hook-system--data-protection)
- [Sub-agent orchestration](#sub-agent-orchestration)
- [Directory map](#directory-map)
- [Maintenance guide](#maintenance-guide)

---

## Layer diagram (L0 → L5)

Mỗi command chạy qua 6 lớp, từ bảo vệ dữ liệu đến sinh artifact:

```
LAYER 0 — PROTECTION                              (luôn chạy trước)
  hooks/data-guard.js     → PreToolUse hook: chặn đọc file nhạy cảm
  rules/data-protection.md → quy tắc declarative cho AI agent
                          │ safe ↓   blocked → ✋ STOP
LAYER 1 — ENTRY POINT                             (user trigger)
  commands/*.tmpl          (slash commands: /generate-bdd, /generate-code, …)
  skills/*/SKILL.tmpl      (auto-trigger theo description match)
                          │ cả hai build bởi bin/build.js
LAYER 2 — SHARED STEPS                            (DRY, injected)
  steps/gate.md            → resolve target file + CHECKPOINT
  steps/context-loader.md  → load project config + rules
  steps/report-footer.md   → standard output format
  Injected ở build time qua {{include:steps/X.md}}  (*.tmpl → *.md, gitignored)
                          │ context loaded
LAYER 3 — PROJECT CONTEXT                 (đọc từ consumer project)
  .agent/project-context.yaml  → paths, tech_stack, domains
  CLAUDE.md (root)             → umbrella-wide shared rules (base layer)
  {service_root}/CLAUDE.md     → service architecture + coding standards (overlay, wins)
  rules/data-protection.md     → AI không được đọc gì
  .agent/modules/{stack}/      → stack rules (plug-in, optional)
                          │ context-aware
LAYER 4 — EXECUTION                               (command logic)
  Discovery   /define-product
  PRD / BDD   /generate-prd · /refine-prd · /generate-bdd · /generate-tech-docs
  Code        /generate-code · /review-code
  Dev check   /dev-gen-test · /dev-run-test · /dev-smoke-test  → set dev_selftest
  QC suite    /qc-analyze → /qc-plan → /qc-design-test → /qc-review →
              /qc-run-test → /qc-report                        → set qc_status
  Trace/Debug /validate-traces · /fix-bug · /debug
                          │
LAYER 5 — OUTPUT                          (artifacts in consumer proj)
  Spec module (cross-team, via {spec_source}):
    specs/product-definition/ · specs/prd/ · specs/bdd/
    specs/tech-docs/ (API contract) · .living-docs/ (gitignored)
  Service submodule (per-service):
    src/ · .trace/*.tsv (authoritative, committed) · .agent/review/
  QC automation outputs:
    QC test cases / scripts (Python pytest-playwright, Page Object)
    → guides per-layer ở skills/qc/<stage>/ · qc_status trong .trace/*.tsv
```

> **Dev self-check vs QC chính thức:** `/dev-*` set `dev_selftest` (smoke, dev tự kiểm code của mình); 6 lệnh `/qc-*` là QC automation pipeline CHÍNH THỨC (port từ agent của team QC; QC repo nay chỉ còn reference) → set `qc_status`. Hai tín hiệu **orthogonal**, cả hai surface trong Living Docs. Chi tiết: [traceability.md](traceability.md).

---

## Data flow — một command chạy như thế nào

Ví dụ `/generate-bdd specs/prd/payment/PAY-01.md`:

```
User types: /generate-bdd specs/prd/payment/PAY-01.md
  │
[L0] data-guard.js check tool calls real-time → đọc .env/*.key → BLOCK + warn
  │
[L1] commands/generate-bdd.md (assembled từ .tmpl + injected steps) được Claude đọc
  │
[L2] gate.md → resolve file path từ $ARGUMENTS
     context-loader.md → đọc project-context.yaml, CLAUDE.md,
                         rules/data-protection.md, modules/{stack}/stack-profile.yaml
  │
[L3] Claude biết: tech_stack, domains, architecture rules, sensitive files, stack patterns
  │
[L4] generate-bdd logic: đọc PRD → extract UC/BR/AC → apply BDD rules R1–R10
  │
[L5] Output: specs/bdd/payment/PAY-01-UC1.feature
```

Chi tiết về step-architecture (gate / context-loader / report-footer) và sub-agent model: xem [pipeline.md](pipeline.md#command-internals--step-architecture).

---

## Build system

`*.tmpl` (committed) được assemble thành `*.md` (gitignored) bởi `bin/build.js`:

```
Source (committed)                Build output (gitignored)
commands/*.tmpl       ──┐
skills/**/SKILL.tmpl  ──┤  node bin/build.js  →  commands/*.md · skills/**/SKILL.md
steps/*.md (shared)   ──┘

Trigger:
  npm run build         ← manual
  prepublishOnly hook   ← auto trước npm publish
  bin/index.js install  ← auto khi user chạy npx
```

> Build output cũng bao gồm `core/` — bản distributable được copy vào `.agent/` của consumer khi `--init`.

---

## Module plug-in system

Stack module là plug-in tùy chọn: cài qua `--module`, đọc ở runtime bởi `context-loader.md`.

```
Available modules (modules/):
  java-spring, angular, react, nextjs, vue, nuxt, dotnet, golang,
  php-laravel, flutter, react-native, ios-swiftui, android-compose,
  context-engineering, qc-playwright

Usage:
  npx @anhth2/spec-driven-dev-plugin --module java-spring
    └─ copies modules/java-spring/ → consumer/.agent/modules/java-spring/

Runtime, context-loader.md đọc:
  .agent/modules/{tech_stack.module}/stack-profile.yaml
  .agent/modules/{tech_stack.module}/architecture-snippets/
```

> **`qc-playwright`** là stack module cho native QC pipeline (`/qc-run-test`, `/qc-report`) — Python + pytest-playwright + Page Object — **ĐỘC LẬP** với dev implementation module (java-spring / react / flutter / …). Per-layer guides ở `skills/qc/<stage>/`. Danh sách module đầy đủ: [../05-reference/modules.md](../05-reference/modules.md).

---

## Hook system — data protection

```
Consumer project setup:
  .claude/settings.json   ← registers hook (template)
  hooks/data-guard.js     ← copied khi install

Runtime — mỗi tool use (Read, Write, Edit, Bash):
  data-guard.js checks:
    .env* · *.key · *.pem · *secret* · *password* · *credential*
    application-prod.* · appsettings.Production.*
       │
    safe → allow      blocked → exit(2) + warn user
```

---

## Sub-agent orchestration

Khi một command quá nặng cho single context window, orchestrator spawn các sub-agent độc lập (mỗi agent có context window riêng):

```
Main session (orchestrator — lightweight, chỉ coordinate)
  ├─ spawn spec-agent   ──→ /refine-prd analysis      → returns findings.yaml
  ├─ spawn codegen-agent ──→ /generate-code UC1        → returns src/ changes
  └─ spawn test-agent   ──→ /dev-gen-test UC1          → returns self-check test files
```

**Lợi ích:** main session không bị bloat bởi large file reads · mỗi agent focus 1 task, ít hallucination · parallel cho nhiều UC.

Pattern này được hiện thực hóa qua `steps/spawn-agent.md` và tự kích hoạt cho `/generate-bdd`, `/generate-code`, `/dev-gen-test` khi PRD vượt ngưỡng phức tạp (> 3 UC hoặc > 300 dòng). Chi tiết flow + tiết kiệm context: [pipeline.md](pipeline.md#spawn-agentmd--sub-agent-orchestration).

---

## Directory map

```
spec-driven-dev/
├── docs/                    ← Toàn bộ tài liệu (bắt đầu ở docs/README.md)  ◀◀◀
│   ├── 01-getting-started/
│   ├── 02-guides/
│   ├── 03-concepts/         ← file này (architecture.md)
│   ├── 04-operations/
│   └── 05-reference/
├── bin/
│   ├── build.js             ← assembles *.tmpl → *.md
│   └── index.js             ← npm installer + hook installer
├── commands/
│   └── *.tmpl               ← slash commands
├── hooks/
│   ├── data-guard.js        ← PreToolUse sensitive file protection
│   └── settings.json        ← hook registration template
├── modules/
│   └── {stack}/             ← java-spring, react, …, qc-playwright
│       ├── module.yaml
│       ├── stack-profile.yaml
│       └── architecture-snippets/
├── rules/
│   ├── data-protection.md   ← what AI must NEVER read/write
│   └── workflow.md          ← general AI behavior rules
├── skills/
│   ├── {name}/SKILL.tmpl    ← Claude plugin skills
│   └── qc/<stage>/          ← per-layer QC automation guides
├── steps/
│   ├── gate.md              ← shared: file resolve + checkpoint
│   ├── context-loader.md    ← shared: load all project context
│   ├── spawn-agent.md       ← shared: sub-agent orchestration
│   ├── capture-lesson.md    ← shared: record a guardrail (/learn etc.)
│   └── report-footer.md     ← shared: standard output format
└── templates/
    ├── project-context.yaml ← consumer project config template
    ├── architecture.template.md
    └── platform-guide.template.md
```

> **Build output (gitignored):** `commands/*.md`, `skills/**/SKILL.md`, và `core/`. Consumer-side tester artifacts nằm trong shared spec repo tại `feedback/bug-reports/` và `feedback/bdd-proposals/`.

> **Umbrella mode — API contract (tech-docs) là cross-team artifact:** khi `setup.spec_source` được set, tech-docs LUÔN route về `{spec_source}/specs/tech-docs/` (giống PRD / design-spec / domain-knowledge), KHÔNG per-service — để FE/App đọc contract qua spec submodule ở `/generate-code --phase=integration`. Chỉ khi không có `spec_source` thì tech-docs mới nằm per-service.

> **Living Docs / trace data location:** `.trace/*.tsv` của mỗi service là **authoritative** và commit trong chính service submodule. Canonical report (`trace-report.json` + TSV mirror namespaced) sinh vào spec module tại `{spec_source}/.living-docs/` (gitignored), cộng một panel mirror cục bộ tại `./.trace`. Chi tiết: [traceability.md](traceability.md#living-docs--canonical-trong-spec-module--panel-mirror).

---

## Maintenance guide

| Muốn thay đổi gì | Sửa file nào |
|------------------|-------------|
| Logic của 1 command cụ thể | `commands/{name}.tmpl` |
| Logic của 1 skill cụ thể | `skills/{name}/SKILL.tmpl` |
| Gate / checkpoint pattern | `steps/gate.md` |
| Context loading | `steps/context-loader.md` |
| Report format | `steps/report-footer.md` |
| Sensitive file patterns | `hooks/data-guard.js` + `rules/data-protection.md` |
| Stack-specific rules | `modules/{stack}/stack-profile.yaml` |
| QC automation rules (per-layer) | `skills/qc/<stage>/` + `modules/qc-playwright/` |
| Project setup template | `templates/project-context.yaml` |
| Build system | `bin/build.js` |
| Installer | `bin/index.js` |

Sau khi sửa bất kỳ `.tmpl` hoặc `steps/*.md`:
```bash
npm run build   # regenerate tất cả *.md
```
