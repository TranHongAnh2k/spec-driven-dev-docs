# Spec-Driven Dev Plugin

An AI-First Spec-Driven Development workflow plugin for Claude Code.

Guides your team through a complete feature lifecycle:  
**Discovery → PRD → Review → BDD Spec → Review → Tech Design → Review → Code → Tests**  
— with AI review gates at every transition and end-to-end traceability.

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Full Workflow](#full-workflow)
- [Operations Guide](#operations-guide)
- [PO/BA Guide](#poba-guide)
- [Multi-Repo / Umbrella Setup](#multi-repo--umbrella-setup)
- [Figma Design System Integration](#figma-design-system-integration)
- [Platform-Adaptive Generation](#platform-adaptive-generation)
- [Stack Modules](#stack-modules)
- [Command Reference](#command-reference)
- [Command Internals — Step Architecture](#command-internals--step-architecture)
- [Domain Knowledge Files](#domain-knowledge-files)
- [Traceability System](#traceability-system)
- [VS Code Extension](#spec-driven-dev-vs-code-extension)
- [Model Recommendation](#model-recommendation)
- [Philosophy](#philosophy)

---

## Installation

Requires: [Node.js](https://nodejs.org) (check: `node -v`).

### New projects — recommended

```bash
# Single-service project:
npx @anhth2/spec-driven-dev-plugin --init --module java-spring

# Multi-service monorepo (installs into each subfolder in one command):
npx @anhth2/spec-driven-dev-plugin --init \
  --services backend:java-spring,web-admin:react,app-mobile:flutter
```

Installs the framework to `.agent/` (commit to git → whole team shares it) and creates lightweight shortcuts in `.claude/commands/`. For monorepo setups, also generates a starter `project-context.yaml` in each service subfolder with paths pre-configured to point to the shared `specs/` folder.

### Upgrade existing project

From **inside Claude Code** (recommended — checks version, handles umbrella mode, reviews the diff):

```
/update-framework
```

Or from the **terminal**:

```bash
bash scripts/upgrade.sh
# or:
npx @anhth2/spec-driven-dev-plugin@latest --init
```

> This upgrades only the framework command files. Your `project-context.yaml`, `CLAUDE.md`, domain-knowledge, and `.trace/` are never overwritten. To sync project *content* (submodule code/specs), use `/sync` instead.

### Legacy install (global / per-project)

```bash
npx @anhth2/spec-driven-dev-plugin           # global: ~/.claude/commands/
npx @anhth2/spec-driven-dev-plugin --project # project: ./.claude/commands/
```

### Uninstall

**Mac/Linux:**
```bash
rm -rf .agent/ .claude/commands/
```

**Windows (PowerShell):**
```powershell
Remove-Item -Recurse -Force .agent, .claude\commands
```

---

## Quick Start

```bash
# 1. Install framework
npx @anhth2/spec-driven-dev-plugin --init

# 2. Open project in Claude Code, run:
/setup-ai-first

# 3. Fill in the generated config files:
#    CLAUDE.md                                       ← architecture + coding standards
#    .agent/project-context.yaml                     ← tech stack + paths
#    specs/domain-knowledge/business-dictionary.md   ← canonical terms
#    specs/domain-knowledge/core-entities.md         ← entity glossary

# 4. Start your first feature:
/define-product
```

---

## Full Workflow

```
                    SPEC-DRIVEN DEVELOPMENT WORKFLOW

Phase 1: Discovery
  /define-product ──────────────────────→ specs/product-definition/{slug}.md

Phase 2: PRD
  /generate-prd ─────────────────────────→ specs/prd/{domain}/{slug}.md
  /refine-prd ───────────────────────────→ .agent/review/{prd-slug}-findings.yaml
                              │
                  [Review Board: Accept/Modify/Reject]
                              │
                /refine-prd --resume → apply + bump version
                              │
                          Update PRD
  /review-context {prd-file} ──────────→ .agent/review/{prd-slug}-review-context-findings.yaml
                              │
             [Quick fix]                   [Review Board]
       --fix → apply auto-fixable          Accept/Modify/Reject
            immediately                    → --resume → apply + bump version

Phase 3: Spec & Design
  /generate-design-spec ─────────────────→ specs/design-spec/{domain}/{TICKET-ID}-design-spec-{platform}.md
                         (FE/App only)       Screen specs, component inventory, AC-UI
                              │
              [Designer review + PO sign-off]
                              │
  /generate-bdd ─────────────────────────→ specs/bdd/{domain}/{UC-ID}.feature
  /review-context {feature} ────────────→ .agent/review/{uc-id}-review-bdd-findings.yaml
                              │
             [Quick fix]                   [Review Board]
       --fix → apply auto-fixable          Accept/Modify/Reject
            immediately                    → --resume → apply + bump bdd_version
  /generate-tech-docs ───────────────────→ tech-docs/{domain}/{UC-ID}-tech-design.md
  /review-tech-docs ─────────────────────→ .agent/review/{uc-id}-tech-review-findings.yaml
                              │
                  [Review Board: Accept/Modify/Reject]
                              │
            /review-tech-docs --resume → apply + bump revision

Phase 4: Code
  /generate-code ────────────────────────→ src/...  (@trace.implements tags)
  /review-code ──────────────────────────→ Report: Critical / Major / Minor
                              │
                    [Fix CRITICAL / MAJOR]

Phase 5: Test
  /generate-tests ───────────────────────→ src/test/...  (@trace.verifies tags)
  /run-tests ────────────────────────────→ Test report
  /smoke-test ───────────────────────────→ Live endpoint check (optional)
  /validate-traces {domain} ─────────────→ Coverage matrix + drift detection

Phase 6: Tester Feedback  (QA → PO/Dev, closes the loop)
  /report-bug {UC-ID} ───────────────────→ {spec}/feedback/bug-reports/{BUG-ID}.md  → commit+push
  /propose-scenario {UC-ID} ─────────────→ {spec}/feedback/bdd-proposals/{...}.md   → commit+push
                              │
              PO/Dev: /sync → "📥 New tester feedback"
                              │
        bug → /fix-bug · proposal → add to BDD · new req → update PRD

Cross-cutting  (any role, anytime)
  /sync ─────────────────────────────────→ git pull + submodules + Living Docs + surface feedback
  /learn "AI does X, should Y" ──────────→ project guardrail (loaded into every command)
  /update-framework ─────────────────────→ upgrade framework tooling from npm
```

---

### Phase overview

| Phase | Who | Commands | Output |
|-------|-----|----------|--------|
| **0. Setup** *(one-time)* | Tech Lead | `/setup-ai-first`, `/sync`, `/sync-figma-*` | Config files, submodules, component catalog |
| **1. Discovery** | PO + AI | `/define-product` | `specs/product-definition/{TICKET-ID}-{slug}.md` |
| **2. PRD** | AI → SA/PO review | `/generate-prd`, `/refine-prd`, `/review-context` | `specs/prd/{domain}/{TICKET-ID}-{slug}.md` |
| **2b. Design Spec** *(FE/App only)* | AI → Designer/PO sign-off | `/generate-design-spec` | `specs/design-spec/{domain}/{TICKET-ID}-design-spec-{platform}.md` |
| **3. BDD Spec** | AI → SA/Dev review | `/generate-bdd`, `/review-context` | `specs/bdd/{domain}/{UC-ID}.feature` |
| **4. Tech Design** | AI → SA/Lead review | `/generate-tech-docs`, `/review-tech-docs` | `tech-docs/{domain}/{UC-ID}-tech-design.md` |
| **5. Code** | AI → Dev review | `/generate-code` *(FE: `--phase=ui`/`--phase=integration`)*, `/review-code` | `src/...` |
| **6. Tests** | AI → CI | `/generate-tests`, `/run-tests` | `src/test/...` |
| **7. Trace** | Tech Lead | `/validate-traces` | Coverage matrix + drift report |
| **8. Tester Feedback** | QA → PO/Dev | `/report-bug`, `/propose-scenario` → `/sync` surfaces | `{spec}/feedback/...` → bug fix / new scenario / PRD update |
| **Cross-cutting** | Any role | `/sync`, `/learn`, `/update-framework` | Synced repo · project guardrails · upgraded tooling |

> **PRD is platform-agnostic (Option C pattern):** One PRD serves all platforms. PRD describes business outcomes only — no UI details, no API specifics.
> FE/App teams read PRD → generate Design Spec (Phase 2b) → generate BDD.
> BE teams read PRD directly → generate BDD (skip Phase 2b).
> Adding a new platform later requires no PRD changes — just set up its umbrella and run `/generate-bdd`.

---

### Phase 0 — Project Setup *(one-time per project)*

**Who:** Tech Lead / SA

```
npx @anhth2/spec-driven-dev-plugin --init --module {module}
/setup-ai-first
```

Fill in the generated files:

| File | Content |
|------|---------|
| `CLAUDE.md` | Architecture layers, coding standards, git conventions |
| `.agent/project-context.yaml` | Tech stack, service list (if multi), Figma URLs |
| `specs/domain-knowledge/business-dictionary.md` | Canonical terms, banned terms |
| `specs/domain-knowledge/core-entities.md` | Entity glossary (fields, relationships) |

If a Figma design system exists:
```
/sync-figma-components {module}    ← populate specs/domain-knowledge/figma-components/{module}.md
/sync-figma-tokens                 ← populate specs/domain-knowledge/figma-tokens.md
```

---

### Phase 1 — Feature Discovery

**Who:** PO + AI  
**Command:** `/define-product`

AI walks PO through 7 phases of structured Q&A — no pre-written spec needed:

| Phase | Content | Confirmed by |
|-------|---------|-------------|
| 0. Knowledge Sync | AI auto-fills related entities, existing rules, term mapping | AI only |
| 1. Feature Definition | Context, problem, goal, actors, scope, user story | PO |
| 2. User Flow | Entry point, flow steps, exit point — optional Figma URL | PO |
| 3. Clarification Log | AI asks follow-up questions until 0 unresolved items | PO |
| 4. Business Rules | WHAT the system must / must not do | PO |
| 5. Business Logic | HOW the system executes each rule, error handling | PO |
| 6. Acceptance Criteria | Testable, observable outcomes derived from BRs | PO |
| 7. Validation | Coverage matrix: every flow action has rule + logic + AC | Auto |

> **Multi-service:** At Phase 0, AI asks *"Which service is this feature for?"* and loads the matching Figma component catalog.

> **Figma:** If PO provides a Figma URL at Phase 2, AI calls `get_design_context` + `get_screenshot`, maps every detected component to the catalog, and pre-fills the user flow table. Unrecognized components are flagged as `[NEW]`.

**Output:** `specs/product-definition/{TICKET-ID}-{slug}.md`  
**Next:** `/generate-prd {output-file}`

---

### Phase 2 — Product Requirements Document

**Who:** AI generates → SA / PO reviews

#### Step 2-A — Generate
```
/generate-prd specs/product-definition/{TICKET-ID}-{slug}.md
```
- Reads service + module from product-definition metadata
- Enforces business dictionary (replaces banned terms, flags new terms ≥ 2 occurrences)
- Writes Service + Module rows to PRD metadata — carried downstream by all subsequent commands

**Output:** `specs/prd/{domain}/{TICKET-ID}-{slug}.md`

#### Step 2-B — Refine (optional AI suggestions)
```
/refine-prd specs/prd/{domain}/{TICKET-ID}-{slug}.md
```
Opens VS Code **Review Board** → Accept / Modify / Reject each finding  
```
/refine-prd --resume {prd-file}    ← apply accepted findings + bump version
```

#### Step 2-C — Quality gate *(required before proceeding)*
```
/review-context specs/prd/{domain}/{TICKET-ID}-{slug}.md
```
Checks: banned terms (P1) · ambiguity in ACs/BRs (P2) · conflicts with other PRDs (P3) · structural completeness (P4)

| Fix path | When to use |
|----------|-------------|
| `/review-context --fix {prd}` | Dev quick-fix — applies all auto-fixable findings immediately |
| Review Board → `/review-context --resume {prd}` | PO / SA review — human decides on each finding |

- ✅ **APPROVED** → proceed to Phase 3
- ❌ **NEEDS_FIX** → fix PRD and re-run `/review-context`

---

### Phase 3 — BDD Specification

**Who:** AI generates → SA / Dev reviews

#### Step 3-A — Generate
```
/generate-bdd specs/prd/{domain}/{TICKET-ID}-{slug}.md
```
- Reads `| **Service** |` and `| **Module** |` from PRD metadata
- **Brownfield shortcut:** if PRD Metadata has `| **API Source** | existing |` → System BDD uses the "Existing API Contract" table in PRD directly, skips synthesis from FE/App BDDs and skips conflict resolution
- Applies platform vocabulary automatically:

  | Module type | "click" | "type" | "navigate" |
  |-------------|---------|--------|------------|
  | Web (react/vue/angular/…) | "clicks" | "types into" | "navigates to" |
  | Mobile (flutter/RN/iOS/…) | "taps" | "enters" | "opens" |
  | Backend (java/go/dotnet/…) | "submits a request" | — | — |

- Shows SC outline per UC → waits for confirmation before writing
- Writes `@trace.service` + `@trace.module` to feature file header

**Output (single-service):** `specs/bdd/{domain}/{UC-ID}.feature`  
**Output (multi-service):** `specs/bdd/{domain}/{service}/{UC-ID}.feature`

> **Large PRDs (> 3 UCs or > 300 lines):** Orchestration mode — spawns 1 sub-agent per UC in parallel, each receiving the active service context.

#### Step 3-B — Quality gate *(required before proceeding)*
```
/review-context specs/bdd/{domain}/{UC-ID}.feature
```
Checks: PRD coverage (every AC + every BR bullet → ≥1 scenario) · Gherkin rules R1–R10 · compliance C1–C5

| Fix path | When to use |
|----------|-------------|
| `/review-context --fix {feature}` | Dev quick-fix — terminology, metadata, coverage matrix |
| Review Board → `/review-context --resume {feature}` | SA review — coverage gaps, scenario redesign |

- ✅ **APPROVED** → proceed to Phase 4
- ❌ **NEEDS_FIX** → fix and re-run `/review-context`

---

### Phase 4 — Technical Design

**Who:** AI generates → SA / Tech Lead reviews

#### Step 4-A — Generate
```
/generate-tech-docs specs/bdd/{domain}/{UC-ID}.feature
```
- Reads `@trace.module` from feature header → determines platform type → selects template
- **Brownfield mode:** if `@trace.api_source: existing` → runs in **reverse-document** mode — describes the existing API as-is, notes any gaps, skips design decisions for §2

  | backend | web-frontend | mobile |
  |---------|-------------|--------|
  | §4 API Endpoints | §4 Screen/Component Breakdown | §4 Screen/Widget Breakdown |
  | §5 Service Flow | §5 State Management | §5 State & Data Flow |
  | §6 Data Model | §6 API Integration | §6 API Integration |
  | §7 Caching | §7 Navigation/Routing | §7 Navigation |
  | §8 Cross-Service | §8 Token/Style Usage | §8 Platform Considerations |
  | | §9 Cross-Service | §9 Token/Style Usage |
  | | | §10 Cross-Service |

**Output (single-service):** `tech-docs/{domain}/{UC-ID}-tech-design.md`  
**Output (multi-service):** `tech-docs/{domain}/{service}/{UC-ID}-tech-design.md`

#### Step 4-B — Review *(required before proceeding)*
```
/review-tech-docs tech-docs/{domain}/{UC-ID}-tech-design.md
```
Opens VS Code Review Board → Accept / Modify / Reject  
```
/review-tech-docs --resume {file}    ← apply accepted findings + bump revision
```

- ✅ **APPROVED** → proceed to Phase 5
- ❌ **NEEDS_FIX** → fix and re-run `/review-tech-docs`

---

### Phase 5 — Code Generation

**Who:** AI → Dev review

#### Step 5-A — Generate
```
/generate-code specs/bdd/{domain}/{UC-ID}.feature            # default — full impl or BE
/generate-code specs/bdd/{domain}/{UC-ID}.feature --phase=ui          # FE: UI + mock API adapter
/generate-code specs/bdd/{domain}/{UC-ID}.feature --phase=integration # FE: wire real API
```

| Flag | When to use |
|------|-------------|
| *(none)* | BE, or FE when API is already live |
| `--phase=ui` | FE: BE not ready yet — generates UI + mock adapter from System BDD contract. Tester can test FE immediately |
| `--phase=integration` | FE: after T7 sign-off gate approved — replaces mock adapter with real API calls |

1. Reads `@trace.module` and tech-design revision from feature header
2. **Component enforcement** (frontend/mobile only):
   ```
   ✅ Button       → @/components/ui/Button    (use existing)
   ⚠️  StatusBadge → [TODO — fill catalog]      (blocked until resolved)
   ❌ OrderSummary → [NEW — will generate]      (confirm first)
   ```
3. Generates layers in order from `CLAUDE.md §2` (e.g., DTO → Entity → Repository → Service → Controller)
4. Tags every endpoint: `@trace.implements={UC-ID}-SC{N}`, `@trace.prd_version`, `@trace.bdd_version`
5. Runs `conventions.build_command` — fails fast if build breaks

#### Step 5-B — Review
```
/review-code
```
AI checks: every scenario has an endpoint · trace tags present · layer rules respected · error handling matches CLAUDE.md §5

Fix all CRITICAL and MAJOR issues before merging.

---

### Phase 6 — Tests

**Who:** AI → CI

#### Step 6-A — Generate
```
/generate-tests specs/bdd/{domain}/{UC-ID}.feature
```
Platform-correct test framework selected automatically:

| Platform | Unit / Widget test | Integration / E2E test |
|----------|--------------------|----------------------|
| Backend (Java) | JUnit 5 + Mockito `ServiceImplTest` | `@WebMvcTest` ControllerTest |
| Backend (Go/dotnet/…) | equivalent unit test | equivalent HTTP test |
| Web Frontend | Vitest/Jest + Testing Library component test | Playwright/Cypress E2E spec |
| Flutter | `testWidgets` in `group()` | Integration test |
| React Native | Jest `describe/it` + RNTL | — |
| iOS (Swift) | `XCTestCase async` ViewModel test | — |
| Android (Compose) | `@HiltAndroidTest` + Compose rule | — |

Every test file is tagged: `@trace.verifies={UC-ID}`, `@trace.service={service}`

#### Step 6-B — Run
```
/run-tests
/smoke-test           ← optional: live endpoint check after deploy
```

---

### Phase 7 — Traceability Audit

**Who:** Tech Lead (run anytime)

```
/validate-traces {domain}
```

Reads all `.trace/{UC-ID}.tsv` files in the domain and reports:

| Status | Meaning |
|--------|---------|
| ✅ OK | Code version matches spec version |
| ⚠️ DRIFT | Code was generated from an older PRD/BDD version — re-generate |
| 🔴 GAP | Scenario exists in spec but no implementing code found |
| — UNTRACKED | Scenario recorded but never code-generated |

---

## Operations Guide

For day-to-day operations including role-based workflows, git submodule procedures, and common scenarios, see **[OPERATIONS.md](OPERATIONS.md)**.

Covers:
- Setup per role: PO, FE/Web, BE, App
- Full feature lifecycle with commands at each step
- Git submodule: clone, update, commit 2-layer, troubleshooting
- Common scenarios: PO-first, adding a platform later, PRD updates, domain key mismatch

---

## Bug Flow — PO · Dev · Tester

For cross-role bug handling coordination, see **[BUG_FLOW.md](BUG_FLOW.md)**.

Covers all 6 bug cases with swimlane flows:
- **Case 1** — Code bug (code ≠ BDD) → Dev fixes code
- **Case 2** — BDD bug (BDD ≠ PRD) → Dev fixes BDD + code
- **Case 3** — PRD ambiguity → PO clarifies → Dev cascades
- **Case 4** — PRD change → PO updates → Dev re-implements
- **Case 5** — Design Spec bug (UI ≠ Design Spec) → Designer/Dev
- **Case 6** — Environment / data bug → DevOps / Dev
- Communication templates: bug report, fix notification, PO clarify request
- Close checklist for Dev + Tester + PO

---

## Tester / QA Guide

For Testers and QA engineers, see **[TESTER_GUIDE.md](TESTER_GUIDE.md)**.

Covers:
- How to set up tester's agent using `spec-manifest.yaml` as entry point
- Tester-facing commands: **`/report-bug`** (spec-traced bug report + layer classification) and **`/propose-scenario`** (draft a BDD scenario for an uncovered edge case)
- The feedback handoff: both write to the shared spec repo's `feedback/` area and push → PO/Dev see them via `/sync`
- How to read and understand the spec chain (PRD → BDD → Tech Docs) before testing
- 6 real-world scenarios: new feature, PRD change mid-sprint, BE + Web same feature, missing BDD, regression after refactor, multi-service E2E
- Bug classification by spec layer (PRD unclear vs BDD wrong vs code mismatch)

---

## Developer Guide

For Developers (FE / BE / App), see **[DEV_GUIDE.md](DEV_GUIDE.md)**.

Covers:
- All commands available to dev with when to use each
- Why BDD matters: executable spec, architecture driver, living documentation
- Understanding the trace system (`@trace.module`, `@trace.domain`, `@trace.bdd`)
- 8 real-world scenarios: receive new PRD, generate BDD per platform (BE vs FE vs App), PRD changes mid-sprint, API design sign-off gate, bug fix flow, Design Spec handoff, umbrella submodule setup, brownfield (existing API)
- FE 2-phase code generation: `--phase=ui` (mock adapter, tester-ready) → `--phase=integration` (real API after sign-off)
- Pre-PR checklist

---

## PO/BA Guide

For Product Owners and Business Analysts, see **[PO_GUIDE.md](PO_GUIDE.md)**.

Covers:
- All commands available to PO/BA with when to use each
- 11 real-world scenarios: new feature, design spec, PRD revision, requirements change, domain conflict, business dictionary update, handoff to dev team, brownfield (existing API → skip T7 gate), and more
- PRD writing rules: platform-agnostic AC, testable criteria, negative paths
- Handoff checklist before notifying dev team

---

## Setup & Architecture Reference

- **[SETUP_GUIDE.md](SETUP_GUIDE.md)** — step-by-step install (single-service, monorepo, umbrella), filling `CLAUDE.md` + `.agent/project-context.yaml`, domain-knowledge files.
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — how the framework is built: template→build→`core/` pipeline, step-file composition, module plug-in system, data-guard hook, directory map.

---

## Multi-Repo / Umbrella Setup

For projects where multiple services live in separate git repos (microservices, multi-platform apps), the recommended pattern is an **umbrella repo** — a coordinator repo that contains each service repo as a git submodule.

### Concept

```
my-project-umbrella/          ← Agent opens here (umbrella)
├── .agent/
│   └── project-context.yaml  ← routing config
├── my-project-specs/         ← submodule: PO's spec repo (PRD, design-spec)
├── user-service/             ← submodule: BE microservice
├── order-service/            ← submodule: BE microservice
└── web-app/                  ← submodule: FE app
```

The agent reads PRDs from the spec submodule and generates BDD + code into each service submodule based on the active domain; tech-docs (the API contract) are written to the shared spec submodule so every umbrella reads the same contract.

### Setup

**Install framework at umbrella level** (do NOT install inside service submodules):

```bash
# Example: web umbrella with one NextJS app reading from a spec submodule
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source free-trial-specs \
  --services mass-product-web:nextjs

# Example: BE umbrella with multiple microservices
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services user:java-spring,order:java-spring,payment:golang
```

This installs the framework to `.agent/` at the umbrella root and generates a `project-context.yaml` with service routing configured.

### Generated project-context.yaml

**Umbrella level** (`my-project-be/.agent/project-context.yaml`):
```yaml
setup:
  mode: umbrella
  spec_source: "free-trial-specs"  # path to PO spec submodule

paths:
  prd_dir:         "free-trial-specs/specs/prd"    # auto-derived from spec_source
  design_spec_dir: "free-trial-specs/specs/design-spec"
  tech_docs_dir:   "free-trial-specs/specs/tech-docs"  # API contract — shared, auto-derived from spec_source
  # ... other spec paths also auto-derived

services:                          # domain → service submodule routing
  user:
    path: "user-service"
    module: "java-spring"
    specs_dir: "user-service/specs/bdd"
  order:
    path: "order-service"
    module: "java-spring"
    specs_dir: "order-service/specs/bdd"
```

**Service level** — each submodule needs its own config (`user-service/.agent/project-context.yaml`):
```yaml
tech_stack:
  language: "Java 17"
  framework: "Spring Boot 3.2"
  module: "java-spring"

conventions:
  test_command: "mvn test"
  build_command: "mvn compile"

paths:
  trace_dir: ".trace"
```

> **Why two configs?** The umbrella config handles *routing* (which domain maps to which service). The service config handles *execution* — `test_command`, `build_command`, and `trace_dir` are service-specific and loaded by context-loader Step 1.6 when running `/run-tests` or `/generate-tests` from umbrella root.

### How routing works

When you run a command (e.g., `/generate-bdd free-trial-specs/specs/prd/user/FEAT-01.md`):
1. Context-loader detects domain = `user` from the PRD path
2. **Step 1.5** routes `specs_dir` → `user-service/specs/bdd` and (in shared-spec setups) auto-routes `tech_docs_dir` → `free-trial-specs/specs/tech-docs`. Stores `service_root = "user-service"`.
3. **Step 1.6** loads `user-service/.agent/project-context.yaml` → overrides `conventions.test_command`, `conventions.build_command`, `paths.trace_dir` with service-specific values.
4. BDD, code, and tests are generated inside the correct service submodule (tech-docs go to the shared spec submodule). `/run-tests` runs as `cd user-service && {test_command}` automatically.

> **Prerequisite:** Each service submodule must have its own `.agent/project-context.yaml` with `conventions.test_command` and `conventions.build_command`. Without it, Step 1.6 falls back to umbrella defaults (which may be empty or wrong).

**Key requirement:** PRD files must have `@trace.domain` in their frontmatter matching a key in the `services` config:

```yaml
# In PRD file frontmatter:
@trace.domain: user      ← must match a services key
@trace.id: FEAT-01
@trace.status: approved  ← dev team should not generate BDD from draft PRDs
```

`/review-context` automatically validates this with a **P0 — Umbrella Routing Check** before running other checks. It catches missing `@trace.domain`, domain key mismatches, and draft status — warning the dev team before they run `/generate-bdd`.

### File ownership by repo

| File type | Where it lives | Who commits |
|-----------|---------------|-------------|
| PRD, Design Spec, Product Definition | Spec submodule (`my-project-specs/`) | PO's team |
| BDD feature files | Each service submodule (`user-service/specs/bdd/`) | Dev team |
| Tech docs (API contract) | **Spec submodule** (`free-trial-specs/specs/tech-docs/`) | BE dev → push to spec repo |
| Code | Each service submodule (`user-service/src/`) | Dev team |

### Committing generated files

Generated files go inside service submodules → requires a two-layer commit:

```bash
# 1. Commit inside the service submodule
cd user-service
git add specs/bdd/user/FEAT-01.feature
git commit -m "feat(FEAT-01): add BDD spec"
git push

# 2. Update submodule pointer in umbrella
cd ..
git add user-service
git commit -m "chore: update user-service submodule pointer"
git push
```

### Updating the spec submodule

When PO pushes new PRDs or design specs, dev teams must update their spec submodule before generating:

```bash
# Pull latest PRDs from PO (run from umbrella repo)
git submodule update --remote my-project-specs
git add my-project-specs
git commit -m "chore: update spec submodule to latest"
git push
```

### PO starts before dev repos exist

A common pattern: PO writes PRDs weeks before dev teams set up their umbrella repos. This works seamlessly because the spec repo is fully independent.

**PO's workflow (no dev repos needed):**
```bash
# PO runs setup on their spec repo
cd my-project-spec
npx @anhth2/spec-driven-dev-plugin --init
/setup-ai-first   # choose: 3. PO Spec repo

# PO writes features — agent asks for domain list upfront
/define-product
/generate-prd       # PRD lands in specs/prd/{domain}/
/generate-design-spec  # Design spec for FE/App
```

**Critical:** PO must define domain names before writing PRDs (prompted by `/setup-ai-first`) and ensure every PRD has `@trace.domain` set. Dev teams use these exact names as `services` keys in their umbrella config.

**When dev team joins later:**
1. Set up umbrella repo with spec repo as submodule
2. Configure `services` keys to match PO's domain names
3. Run `git submodule update --remote {spec-source}` to pull existing PRDs
4. Run `/review-context {prd-file}` — P0 check verifies domain alignment
5. If P0 passes → `/generate-bdd`, `/generate-tech-docs`, `/generate-code`

### Multi-platform setup

For a multi-platform project (PO + Web + BE + App), each platform has its own umbrella:

```
my-project-spec/   ← PO umbrella (npx ... --init)
my-project-web/    ← Web umbrella (npx ... --init --umbrella --spec-source my-project-specs ...)
my-project-be/     ← BE umbrella  (npx ... --init --umbrella --spec-source my-project-specs ...)
my-project-app/    ← App umbrella (npx ... --init --umbrella --spec-source my-project-specs ...)
```

Each platform umbrella has its own `project-context.yaml` pointing PRD paths to the shared spec submodule.

---

## Figma Design System Integration

### Overview

The framework integrates with the [Figma MCP server](https://www.figma.com/developers/mcp) to:
- Read designs directly from Figma URLs (no manual copy-paste)
- Match Figma components to code components before generating UI code
- Enforce design token usage (no hardcoded colors/sizes)
- Detect when new Figma components need to be built vs. reused

### Initial setup

Run `/setup-ai-first` — it will guide you through:
1. Choosing single-service or multi-service
2. Collecting a Figma URL per service
3. Creating `specs/domain-knowledge/figma-components/{module}.md` for each service
4. Updating `project-context.yaml` with Figma URLs

### Keeping catalogs in sync

```bash
# Sync component catalog for a specific module:
/sync-figma-components react
/sync-figma-components flutter

# Sync design tokens:
/sync-figma-tokens
/sync-figma-tokens https://figma.com/design/...
```

These commands detect drift between the local catalog and the live Figma file, then update the markdown catalog.

### How it affects code generation

When a Figma component catalog is loaded, every code-generation command enforces it:

```
UI Components used in this feature:
  ✅ Button        → @/components/ui/Button           (existing)
  ✅ DataTable     → @/components/ui/DataTable         (existing)
  ⚠️  StatusBadge  → [TODO — fill catalog]             (blocked)
  ❌ OrderSummary  → [NEW — will generate]             (new file)
```

- **Matched** → import from the catalog path exactly, never create a duplicate
- **Matched but TODO** → block generation until catalog is filled
- **Not matched** → generate only after user confirms

### Design token enforcement

When `specs/domain-knowledge/figma-tokens.md` is present:

```
# AI will refuse to generate:
color: #1E40AF
fontSize: 14
padding: 16

# AI will generate instead:
color: tokens.color.primary[500]
fontSize: tokens.font.size.sm
padding: tokens.spacing[4]
```

---

## Platform-Adaptive Generation

The framework detects the platform type from `tech_stack.module` (or `active_module` in multi-service) and adapts all generated artifacts automatically.

### Platform types

| Platform | Modules |
|----------|---------|
| `backend` | `java-spring`, `golang`, `dotnet`, `php-laravel`, `context-engineering` |
| `web-frontend` | `react`, `nextjs`, `vue`, `nuxt`, `angular` |
| `mobile` | `flutter`, `react-native`, `ios-swiftui`, `android-compose` |

### BDD vocabulary adapts per platform

| Platform | "click" | "type" | "see" | "navigate" |
|----------|---------|--------|-------|------------|
| Web | "clicks" | "types into" | "sees" | "navigates to" |
| Mobile | "taps" | "enters" | "sees" | "opens" |
| Backend | "submits a request" | — | "receives response" | — |

### Tech design sections adapt per platform

| `backend` | `web-frontend` | `mobile` |
|-----------|---------------|---------|
| §4 API Endpoints | §4 Screen/Component Breakdown | §4 Screen/Widget Breakdown |
| §5 Service Flow | §5 State Management | §5 State & Data Flow |
| §6 Data Model | §6 API Integration | §6 API Integration |
| §7 Caching | §7 Navigation/Routing | §7 Navigation |
| §8 Cross-Service | §8 Token/Style Usage | §8 Platform Considerations |
| — | §9 Cross-Service | §9 Token/Style Usage |
| — | — | §10 Cross-Service |

### Test templates adapt per platform

| Platform | Unit / Widget | Integration / E2E |
|----------|--------------|-------------------|
| Backend (Java) | JUnit 5 + Mockito `ServiceImplTest` | Spring `@WebMvcTest` ControllerTest |
| Web Frontend | Vitest/Jest + Testing Library component test | Playwright/Cypress E2E spec |
| Flutter | `testWidgets` in `group()` | Integration test |
| React Native | Jest `describe/it` + RNTL | — |
| iOS (Swift) | `XCTestCase async` ViewModel test | — |
| Android (Compose) | `@HiltAndroidTest` + Compose rule | — |

---

## Stack Modules

Each module ships a `stack-profile.yaml` with framework-specific layer patterns, naming rules, and test patterns.

| Module | Language / Framework | Platform |
|--------|---------------------|---------|
| `java-spring` | Java + Spring Boot | Backend |
| `golang` | Go + Gin/Echo | Backend |
| `dotnet` | C# + .NET / ASP.NET Core | Backend |
| `php-laravel` | PHP + Laravel | Backend |
| `context-engineering` | Generic AI/LLM projects | Backend |
| `react` | TypeScript + React | Web Frontend |
| `nextjs` | TypeScript + Next.js | Web Frontend |
| `vue` | TypeScript + Vue 3 | Web Frontend |
| `nuxt` | TypeScript + Nuxt 3 + @nuxt/ui | Web Frontend |
| `angular` | TypeScript + Angular | Web Frontend |
| `flutter` | Dart + Flutter | Mobile |
| `react-native` | TypeScript + React Native / Expo | Mobile |
| `ios-swiftui` | Swift + SwiftUI | Mobile |
| `android-compose` | Kotlin + Jetpack Compose | Mobile |

---

## Command Reference

### Setup & Maintenance

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/setup-ai-first` | — | Project structure + config files | First-time project setup |
| `/sync` | — | Git pull + submodule update + Living Docs sync | **Daily driver** — sync project *content* (submodule code/specs) |
| `/update-framework` | — | Refreshed `.agent/` command files from npm | **Occasionally** — upgrade the *framework itself* when a new version ships |
| `/sync-figma-components {module}` | Module name | Updated `figma-components/{module}.md` | After Figma design system changes |
| `/sync-figma-tokens {url?}` | Optional Figma URL | Updated `figma-tokens.md` | After design token changes |

> **`/sync` vs `/update-framework` — two different things:**
> - `/sync` updates your **project content**: pulls the latest submodule code/specs from git and refreshes the Living Docs panel. Source = your git remotes. Run often.
> - `/update-framework` updates the **framework tooling**: refreshes `.agent/commands/`, `steps/`, `modules/` etc. to the latest published version. Source = npm registry. Run rarely. It never touches your `project-context.yaml`, `CLAUDE.md`, domain-knowledge, or `.trace/`.

### Feature lifecycle

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/define-product` | — | `specs/product-definition/*.md` | Starting any new feature |
| `/generate-prd` | product-definition file | `specs/prd/{domain}/{slug}.md` | After define-product. Add `API Source: existing` + "Existing API Contract" section for brownfield features |
| `/refine-prd` | prd file | `.agent/review/{prd-slug}-findings.yaml` | After generate-prd |
| `/review-context` | prd or feature file | `.agent/review/*-findings.yaml` | After PRD update; after generate-bdd |
| `/review-context --fix` | prd or feature file | Applies all auto-fixable findings immediately | Dev quick-fix without Review Board |
| `/review-context --resume` | prd or feature file | Applies accepted findings | After reviewing in Review Board |
| `/generate-design-spec` | prd file | `specs/design-spec/{domain}/{TICKET-ID}-design-spec-{platform}.md` | FE/App: after PRD approved — before BDD |
| `/generate-bdd` | prd file | `specs/bdd/{domain}/{slug}.feature` | After PRD approved (+ Design Spec sign-off for FE/App) |
| `/generate-tech-docs` | feature file | `tech-docs/{domain}/{slug}-tech-design.md` | After BDD approved |
| `/review-tech-docs` | tech-design file | `.agent/review/*-findings.yaml` | After generate-tech-docs |
| `/review-tech-docs --resume` | tech-design file | Applies accepted findings | After reviewing in Review Board |
| `/generate-code` | feature file | `src/...` | After tech-design approved. Use `--phase=ui` (FE mock) or `--phase=integration` (FE real API) when BE not ready |
| `/review-code` | — | Review report | After generate-code |
| `/generate-tests` | feature file | `src/test/...` | After generate-code |
| `/run-tests` | — | Test report | After generate-tests |
| `/smoke-test` | — | Live endpoint check | After deploy |

### Debugging & Tracing

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/fix-bug` | — | Bug fix | When a bug is found |
| `/debug` | — | Debug session | Deep debugging |
| `/validate-traces {domain}` | domain name | Coverage matrix + drift report | Anytime — verify traceability |
| `/learn {text}` | "AI does X, should Y" | Appends a guardrail to the project lessons file | When the AI repeats a mistake you want to stop |

### Tester (read-only on specs)

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/report-bug {UC-ID} {desc}` | UC/TICKET + description | `{spec_repo}/feedback/bug-reports/{BUG-ID}.md` + spec context + layer classification | Tester finds a bug |
| `/propose-scenario {UC-ID} {desc}` | UC + edge-case description | `{spec_repo}/feedback/bdd-proposals/{UC-ID}-*.md` draft Gherkin | Tester finds an edge case not covered by BDD |

> Both are **read-only on canonical specs/code**. They write only to the shared spec repo's `feedback/` area and **commit + push it** there — so PO/Dev see the items on their next `/sync` (which lists newly-pulled feedback). `/report-bug` classifies the likely layer (Code/BDD/PRD/Design/Env) to route the bug; `/propose-scenario` drafts a scenario mapped to an existing PRD AC (or emits a PRD change request if the behavior is genuinely new). PO/Dev promote proposals into canonical BDD — testers never edit `.feature` directly.
>
> **Handoff loop:** tester `/report-bug` or `/propose-scenario` → commit+push to spec repo `feedback/` → PO/Dev `/sync` surfaces `📥 N new` → they act (`/fix-bug`, promote proposal, or update PRD). A file alone notifies no one; the commit + `/sync` surfacing is what closes the loop.

### Project Lessons (self-improving guardrails)

The framework accumulates a **project memory** of mistakes the AI should not repeat:

- **Capture** — run `/learn "AI keeps calling repositories from controllers — must go through the service layer"`, or accept the prompt that `/review-code`, `/fix-bug`, and `/debug` show when they find a repeatable AI mistake.
- **Store** — lessons land in `paths.lessons_file` (default `specs/domain-knowledge/lessons-learned.md`; per-service `.agent/project-lessons.md` in umbrella mode). Commit it so the whole team shares the guardrails.
- **Apply** — context-loader **Step 6.7** loads every lesson at the start of **every** command and treats each as a hard constraint (same priority as CLAUDE.md). Generation commands check their output against matching lessons before presenting it.

> This is **project memory, not model fine-tuning** — the model's weights never change. Lessons work by being injected into context on every run, which functionally stops the repeat.

---

## Command Internals — Step Architecture

Every command in `.agent/commands/` is composed of shared **step files** in `.agent/steps/`. These steps are the foundation — every command calls them in sequence before running its own logic.

### Step files at a glance

| File | Role | Called by |
|------|------|-----------|
| `gate.md` | Universal entry — model check, file resolution, context load, user checkpoint | Every command |
| `context-loader.md` | Multi-step context loading sequence (stack → service routing → service conventions → arch → safety → domain → UI → recap) | `gate.md` Step 2 |
| `spawn-agent.md` | Sub-agent orchestration for large PRDs | `generate-bdd`, `generate-code`, `generate-tests` |
| `capture-lesson.md` | Append/refine a guardrail in the project lessons file | `learn`, `review-code`, `fix-bug`, `debug` |
| `report-footer.md` | Standard output format (status badge, artifact list, next command) | Every command |

---

### gate.md — Universal Entry Procedure

Every command starts here, in order:

| Step | What it does |
|------|-------------|
| **Step 0** | Sub-agent mode check — if `$ARGUMENTS` is a JSON payload with `_agent_mode: true`, skip Steps 1-3 and use the slim context directly |
| **Step 0-B** | Model check — prompts to switch to `claude-opus` for complex generation; `S` to skip |
| **Step 1** | Resolve target file from `$ARGUMENTS`; if missing, lists candidates and asks the user to choose |
| **Step 2** | Execute `context-loader.md` — load all project context into working memory |
| **Step 3** | CHECKPOINT — display summary (target file, stack, module, domains) and wait for `Y` before proceeding |

```
CHECKPOINT
-----------
Target     : specs/prd/auth/FEAT-042-login.md
Project    : My App
Tech stack : TypeScript / React 18
Module     : react
Domains    : auth, profile

Proceed? (Y/N)
```

---

### context-loader.md — Context Loading Sequence

Loads all project context into working memory, in strict priority order:

| Step | Priority | What loads |
|------|----------|-----------|
| 1 | PROJECT-CONFIG | `project-context.yaml` → stack, conventions, domains, services, paths |
| 1.5 | SERVICE ROUTING | *(umbrella only)* detect active domain, route `specs_dir`/`tech_docs_dir` to matching service submodule, store `service_root` |
| 1.6 | SERVICE CONVENTIONS | *(umbrella only)* load `{service_root}/.agent/project-context.yaml` → override `test_command`, `build_command`, `paths.trace_dir` with service-specific values |
| 2 | PROJECT-CONFIG | `.agent/modules/{module}/stack-profile.yaml` → framework-specific layer patterns |
| 3 | **CRITICAL** | `CLAUDE.md` → architecture layers, coding standards, naming conventions |
| 4 | SAFETY | `.agent/rules/data-protection.md` → sensitive file patterns never to access |
| 5 | DOMAIN | `business-dictionary.md` → canonical terms + banned terms (enforced for entire session) |
| 6 | DOMAIN | `core-entities.md` → entity catalog (field names, types, invariants, relationships) |
| 6.7 | **GUARDRAILS** | `lessons_file` → accumulated mistakes (via `/learn`) loaded as hard constraints for this run |
| 6-B | UI COMPONENTS | `figma-components/{module}.md` → Figma component name → code component + import path |
| 6-C | UI TOKENS | `figma-tokens.md` → design tokens (colors, typography, spacing, radius, shadow) |
| 7 | **RECAP** | `[CTX LOADED]` block printed to lock critical facts into working memory |

**Anti-lost-in-middle principle:** critical facts (Step 3) load first; the RECAP (Step 7) restates them at the end so they are fresh when generation begins (recency effect).

**RECAP block format (printed after every context load):**
```
[CTX LOADED]
Stack      : TypeScript / React 18 / PostgreSQL
Layers     : Controller → Facade → Service → Repository
Ticket     : FEAT-
Services   : 2 services: web-admin(react), api-backend(java-spring)
Svc Root   : api-backend — conventions + trace_dir loaded from service config
Dict       : loaded — 42 canonical terms, 8 banned terms
Entities   : loaded — User, Order, Product
Lessons    : loaded — 6 guardrails
Components : loaded — web-admin (react) — 23 components mapped
Tokens     : loaded — colors: 18, spacing: 12, typography: 6
Status     : FULL
```

Status values: `FULL` (all files loaded) · `PARTIAL — missing: {list}` · `MINIMAL` (only project-context.yaml).

---

### spawn-agent.md — Sub-Agent Orchestration

When a PRD exceeds complexity thresholds, `generate-bdd`, `generate-code`, and `generate-tests` automatically switch from single-session to orchestration mode.

**Complexity thresholds:**

| Signal | Threshold | Action |
|--------|-----------|--------|
| UC count in PRD | > 3 UCs | Spawn 1 sub-agent per UC |
| PRD length | > 300 lines | Spawn agents regardless of UC count |

**Orchestration flow:**

```
Main session (orchestrator)
  │
  ├── Step A: Build slim context JSON
  │     (stack + active_service + active_module + paths + arch summary + banned_terms)
  │
  ├── Step B: Scan PRD → extract UC list [{uc_id, line_start, line_end}]
  │
  ├── Step C: Announce plan → "Spawning N sub-agents..."
  │
  ├── Step D: Spawn 1 sub-agent per UC (parallel)
  │     Each receives: { _agent_mode: true, uc_id, target_file, uc_section, context }
  │     Each reads only its own UC section (not the full PRD)
  │
  └── Step E: Collect results → merge into single report
        { uc_id, files_created, status, errors }
```

**Context window savings:**

| Mode | What loads per agent |
|------|---------------------|
| Single session (≤ 3 UCs) | Full context + full PRD + all UCs |
| Orchestrator | Slim context + UC heading list only |
| Each sub-agent | Slim context + **1 UC section only** |

---

### report-footer.md — Standard Output Format

Every command ends its output with this footer:

```
---
Status : ✅ Complete
Output Artifacts:
  created  specs/bdd/auth/FEAT-042-UC1-login.feature (3 scenarios)
  updated  .trace/FEAT-042.tsv
Next   : /review-context specs/bdd/auth/FEAT-042-UC1-login.feature
```

**Status badges:** `✅ Complete` · `⚠️ Warnings` · `❌ Failed`

The **Next** field always suggests the logical next command — so you never have to look up what comes after.

---

## Domain Knowledge Files

After `/setup-ai-first`, fill in these files — they are loaded by **every** command:

| File | Purpose | Managed by |
|------|---------|-----------|
| `CLAUDE.md` | Architecture rules, coding standards, git conventions | Tech Lead |
| `.agent/project-context.yaml` | Tech stack, services, paths, ticket prefix | Tech Lead |
| `specs/domain-knowledge/business-dictionary.md` | Canonical terms, banned terms, enum registry | PO / SA |
| `specs/domain-knowledge/core-entities.md` | Entity definitions (fields, invariants, relationships) | Tech Lead |
| `specs/domain-knowledge/figma-components/{module}.md` | Figma → code component mapping (one per module) | Designer + Dev |
| `specs/domain-knowledge/figma-tokens.md` | Design tokens (colors, typography, spacing, etc.) | Designer |

### project-context.yaml structure

```yaml
project:
  name: My Project

tech_stack:
  language: TypeScript
  framework: React 18
  module: react            # ← determines platform type and catalog file

conventions:
  ticket_prefix: FEAT

# Optional: multi-service projects
services:
  - name: web-admin
    module: react
    description: Admin dashboard
  - name: api-backend
    module: java-spring
    description: REST backend

# Optional: Figma integration
figma_tokens_url: https://figma.com/design/...
```

---

## Traceability System

Every artifact links to every other through `@trace.*` tags.

### Artifact chain

```
product-definition.md
  └─► PRD.md (Service, Module in metadata)
        └─► {UC-ID}.feature (@trace.service, @trace.module, @trace.prd_version)
              └─► tech-design.md (@trace.bdd_version)
                    └─► src/ code (@trace.implements, @trace.prd_version, @trace.bdd_version)
                          └─► src/test/ (@trace.verifies, @trace.service)
                                └─► .trace/{UC-ID}.tsv (drift tracking)
```

### Tags in `.feature` files

```gherkin
# @trace.id: FEAT-001-UC1
# @trace.service: web-admin
# @trace.module: react
# @trace.prd_version: 1.2
# @trace.bdd_version: 3
# @trace.api_source: existing   ← brownfield: API đã tồn tại, contract lấy từ PRD
```

### Tags in code

```java
// @trace.implements=FEAT-001-UC1-SC2
// @trace.prd_version=1.2
// @trace.bdd_version=3
// @trace.tech_doc_revision=2
```

### Tags in tests

```java
// @trace.verifies=FEAT-001-UC1
// @trace.service=api-backend
// @trace.test_type=integration
```

### Drift detection

Run `/validate-traces {domain}` anytime to check:
- Which scenarios have no implementing code (gap)
- Which code was generated from an older PRD/BDD version (drift)
- Overall coverage percentage per domain

---

## Spec Driven Dev VS Code Extension

Install once, get two panels:

```bash
code --install-extension edupia-team.spec-driven-dev-team
```

### Panel 1 — Review Board

Reads `*-findings.yaml` from `.agent/review/` — picks up findings from all review commands automatically.

| Command | Findings file | Quick-fix | Review Board |
|---------|--------------|-----------|--------------|
| `/refine-prd` | `{prd-slug}-findings.yaml` | — | `/refine-prd --resume {prd}` |
| `/review-context` (PRD) | `{prd-slug}-review-context-findings.yaml` | `/review-context --fix {prd}` | `/review-context --resume {prd}` |
| `/review-context` (BDD) | `{uc-id}-review-bdd-findings.yaml` | `/review-context --fix {feature}` | `/review-context --resume {feature}` |
| `/review-tech-docs` | `{uc-id}-tech-review-findings.yaml` | — | `/review-tech-docs --resume {tech-doc}` |

**Two paths for `review-context`:**

| Path | When to use |
|------|-------------|
| `--fix` | Dev cleaning up BDD/PRD — apply all auto-fixable issues immediately, no Review Board needed |
| Review Board → `--resume` | PO or SA reviewing PRD — human accepts/modifies each finding before applying |

- Actions: Accept · Modify (add decision note) · Defer · Reject
- Each finding shows `auto_fixable: true/false`
- "Apply" button spawns terminal and runs `--resume`

### Panel 2 — Living Documentation

Reads `.trace/*.tsv` — shows project-wide traceability health at a glance.

```
┌──────────────────────────────────────────────────────────────────┐
│ PRDs  Use Cases  Scenarios  Code Cov.  Test Cov.  Drift  Gap     │
│  19     86        1077        93%        89%       212    93      │
└──────────────────────────────────────────────────────────────────┘
```

- Drill down: PRD → UC → per-scenario table with Spec ver, Gen ver, Code, Tests, Status
- Status badges: ✅ OK · ⚠️ DRIFT · 🔴 GAP · — UNTRACKED
- Filter by domain, PRD status, doc status
- Search by UC/SC ID or title

**Data source:** `.trace/{UC-ID}.tsv` — written by `/generate-bdd`, `/generate-code`, `/generate-tests`, updated by `/validate-traces`.

**Umbrella / submodule mode:** each service submodule has its own `.trace/` dir. The panel at umbrella root reads from `{umbrella}/.trace/`, which is populated by `/validate-traces` (it aggregates from all service trace dirs). Run `/validate-traces` after each codegen session to refresh the panel. Add `.trace/` to the umbrella's `.gitignore` — it is a read-only mirror, not committed.

---

## Model Recommendation

All generation commands prompt you to confirm you're using **claude-opus** before proceeding. Smaller models (haiku, sonnet) risk missed edge cases in spec analysis and code generation.

```
⚙️  MODEL CHECK
──────────────────────────────────────────────
  Recommended  : claude-opus-4-5 (or claude-opus-4)
  Why needed   : Spec analysis, architecture review,
                 code generation require deep reasoning.
──────────────────────────────────────────────
  Y — yes, on claude-opus → proceed
  S — skip check (accept lower quality risk)
```

---

## Philosophy

> **Write the spec first. Generate the code from the spec. Trace everything.**

- Humans define *what* — acceptance criteria, business rules, platform requirements
- AI generates *how* — BDD scenarios, tech design, code, tests — adapted to the platform
- Every artifact is reviewed before proceeding to the next phase
- Every line of code traces back to a scenario in a `.feature` file
- In multi-service projects, each service evolves independently while sharing the same workflow
