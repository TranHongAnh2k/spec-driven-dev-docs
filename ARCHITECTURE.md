# Framework Architecture

> **Nguyên tắc**: Nhìn file này để hiểu toàn bộ framework trước khi đọc bất kỳ file chi tiết nào.

---

## Layer Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║               SPEC-DRIVEN DEV FRAMEWORK  v0.3                       ║
╚══════════════════════════════════════════════════════════════════════╝

┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 0 — PROTECTION                           (luôn chạy trước)   │
│                                                                      │
│  hooks/data-guard.js   → PreToolUse hook: chặn đọc file nhạy cảm   │
│  rules/data-protection.md → Quy tắc declarative cho AI agent        │
└──────────────────────────────────────────────────────────────────────┘
                               │ safe ↓ blocked → ✋ STOP
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — ENTRY POINT                          (user trigger)       │
│                                                                      │
│  commands/*.tmpl              skills/*/SKILL.tmpl                   │
│  (slash commands)             (auto-trigger by description match)    │
│  /generate-bdd                detect: "tạo BDD", "sinh feature"     │
│  /generate-code               detect: "viết code", "implement"      │
└──────────────────────────────────────────────────────────────────────┘
                               │ both built by bin/build.js
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — SHARED STEPS                         (DRY, injected)      │
│                                                                      │
│  steps/gate.md            → resolve target file + CHECKPOINT        │
│  steps/context-loader.md  → load project config + rules             │
│  steps/report-footer.md   → standard output format                  │
│                                                                      │
│  Injected at build time via {{include:steps/X.md}}                  │
│  Source: *.tmpl  →  Output: *.md  (gitignored)                      │
└──────────────────────────────────────────────────────────────────────┘
                               │ context loaded
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — PROJECT CONTEXT              (read from consumer project) │
│                                                                      │
│  .agent/project-context.yaml  → paths, tech_stack, domains          │
│  CLAUDE.md                    → architecture, coding standards       │
│  rules/data-protection.md     → what AI must never access           │
│  .agent/modules/{stack}/      → stack rules (plug-in, optional)     │
└──────────────────────────────────────────────────────────────────────┘
                               │ context-aware
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 4 — EXECUTION                            (command logic)      │
│                                                                      │
│  DISCOVERY        PRD / BDD           CODE / TEST     DEBUG          │
│  /define-product  /generate-prd       /generate-code  /fix-bug      │
│                   /refine-prd         /generate-tests /debug        │
│                   /generate-bdd       /run-tests      /validate-    │
│                   /generate-tech-docs /smoke-test      traces       │
│                   /review-code        /setup-ai-first               │
└──────────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 5 — OUTPUT                       (artifacts in consumer proj) │
│                                                                      │
│  specs/product-definition/   specs/prd/   specs/bdd/               │
│  tech-docs/                  src/         .trace/   .agent/review/ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow — Một command chạy như thế nào

```
User types: /generate-bdd specs/prd/payment/PAY-01.md
                │
                ▼
[L0] data-guard.js checks tool calls in real-time
     → nếu AI cố đọc .env / *.key → BLOCK + warn
                │
                ▼
[L1] commands/generate-bdd.md được Claude đọc
     (assembled từ generate-bdd.tmpl + injected steps)
                │
                ▼
[L2] gate.md   → resolve file path từ $ARGUMENTS
     context-loader.md → đọc project-context.yaml, CLAUDE.md,
                          rules/data-protection.md,
                          modules/{stack}/stack-profile.yaml
                │
                ▼
[L3] Claude biết: tech_stack, domains, architecture rules,
     sensitive files to avoid, stack-specific patterns
                │
                ▼
[L4] generate-bdd logic:
     → đọc PRD → extract UC/BR/AC
     → apply BDD rules R1-R10
     → viết specs/bdd/{domain}/{UC-ID}.feature
                │
                ▼
[L5] Output: specs/bdd/payment/PAY-01-UC1.feature
```

---

## Build System

```
Source (committed to git)        Build output (gitignored)
──────────────────────────       ─────────────────────────
commands/*.tmpl         ──┐
skills/**/SKILL.tmpl    ──┤  node bin/build.js  →  commands/*.md
steps/*.md   (shared)   ──┘                         skills/**/SKILL.md

Trigger:
  npm run build          ← manual
  prepublishOnly hook    ← auto before npm publish
  bin/index.js install   ← auto when user runs npx
```

---

## Module Plug-in System

```
Available modules (modules/):
  java-spring, angular, react, nextjs,
  dotnet, golang, php-laravel, context-engineering

Usage:
  npx @anhth2/spec-driven-dev-plugin --module java-spring
    └─ copies modules/java-spring/ → consumer/.agent/modules/java-spring/

At runtime, context-loader.md reads:
  .agent/modules/{tech_stack.module}/stack-profile.yaml
  .agent/modules/{tech_stack.module}/architecture-snippets/
```

---

## Hook System — Data Protection

```
Consumer project setup:
  .claude/settings.json   ← registers hook (provided as template)
  hooks/data-guard.js     ← copied from this package on install

Runtime:
  Every tool use (Read, Write, Edit, Bash)
       │
       ▼
  data-guard.js checks:
    .env* / *.key / *.pem / *secret* / *password* / *credential*
    application-prod.* / appsettings.Production.*
       │
    safe → allow   blocked → exit(2) + warn user
```

---

## Future — Sub-Agent Pattern (v2 roadmap)

Khi một số command quá nặng cho single context window:

```
Main session (orchestrator — lightweight, chỉ coordinate)
  │
  ├─ spawn spec-agent ──→ /refine-prd analysis (own context window)
  │    └─ returns: findings.yaml
  │
  ├─ spawn codegen-agent ──→ /generate-code UC1 (own context window)
  │    └─ returns: src/ changes
  │
  └─ spawn test-agent ──→ /generate-tests UC1 (own context window)
       └─ returns: test files

Benefits:
  - Main session không bị bloat bởi large file reads
  - Mỗi agent focus vào 1 task, ít hallucination hơn
  - Parallel execution cho multiple UCs

Implementation path:
  1. Tạo steps/spawn-agent.md — pattern + handoff payload format
  2. Tạo commands/orchestrate-feature.tmpl — top-level orchestrator
  3. Sub-agent commands nhận input qua $ARGUMENTS (JSON payload)
```

---

## Directory Map

```
spec-driven-dev/
├── ARCHITECTURE.md          ← Đọc đây trước  ◀◀◀
├── bin/
│   ├── build.js             ← assembles *.tmpl → *.md
│   └── index.js             ← npm installer + hook installer
├── commands/
│   └── *.tmpl               ← slash commands (23 commands)
├── hooks/
│   ├── data-guard.js        ← PreToolUse sensitive file protection
│   └── settings.json        ← hook registration template
├── modules/
│   └── {stack}/             ← 8 stacks: java-spring, angular, react,
│       ├── module.yaml          nextjs, dotnet, golang, php-laravel,
│       ├── stack-profile.yaml   context-engineering
│       └── architecture-snippets/
├── rules/
│   ├── data-protection.md   ← what AI must NEVER read/write
│   └── workflow.md          ← general AI behavior rules
├── skills/
│   └── {name}/SKILL.tmpl    ← Claude plugin skills (8 skills)
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

> Build output (gitignored): `commands/*.md`, `skills/**/SKILL.md`, and `core/` (the distributable copied into a consumer's `.agent/` by `--init`). Consumer-side tester artifacts live in the shared spec repo under `feedback/bug-reports/` and `feedback/bdd-proposals/`. In umbrella mode the **API contract (tech-docs)** also lives in the shared spec repo (`{spec_source}/specs/tech-docs/`) so BE and FE/App read the same contract via the spec submodule.

---

## Maintenance Guide

| Muốn thay đổi gì | Sửa file nào |
|------------------|-------------|
| Logic của 1 command cụ thể | `commands/{name}.tmpl` |
| Logic của 1 skill cụ thể | `skills/{name}/SKILL.tmpl` |
| Gate / checkpoint pattern | `steps/gate.md` |
| Context loading | `steps/context-loader.md` |
| Report format | `steps/report-footer.md` |
| Sensitive file patterns | `hooks/data-guard.js` + `rules/data-protection.md` |
| Stack-specific rules | `modules/{stack}/stack-profile.yaml` |
| Project setup template | `templates/project-context.yaml` |
| Build system | `bin/build.js` |
| Installer | `bin/index.js` |

Sau khi sửa bất kỳ `.tmpl` hoặc `steps/*.md`:
```bash
npm run build   # regenerate tất cả *.md
```
