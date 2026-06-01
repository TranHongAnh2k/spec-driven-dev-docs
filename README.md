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
- [Multi-Service Projects](#multi-service-projects)
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

```bash
bash scripts/upgrade.sh
# or:
npx @anhth2/spec-driven-dev-plugin@latest --init
```

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
```

---

### Phase overview

| Phase | Who | Commands | Output |
|-------|-----|----------|--------|
| **0. Setup** *(one-time)* | Tech Lead | `/setup-ai-first`, `/sync-figma-*` | Config files, component catalog |
| **1. Discovery** | PO + AI | `/define-product` | `specs/product-definition/{TICKET-ID}-{slug}.md` |
| **2. PRD** | AI → SA/PO review | `/generate-prd`, `/refine-prd`, `/review-context` | `specs/prd/{domain}/{TICKET-ID}-{slug}.md` |
| **3. BDD Spec** | AI → SA/Dev review | `/generate-bdd`, `/review-context` | `specs/bdd/{domain}/{UC-ID}.feature` |
| **4. Tech Design** | AI → SA/Lead review | `/generate-tech-docs`, `/review-tech-docs` | `tech-docs/{domain}/{UC-ID}-tech-design.md` |
| **5. Code** | AI → Dev review | `/generate-code`, `/review-code` | `src/...` |
| **6. Tests** | AI → CI | `/generate-tests`, `/run-tests` | `src/test/...` |
| **7. Trace** | Tech Lead | `/validate-traces` | Coverage matrix + drift report |

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
/generate-code specs/bdd/{domain}/{UC-ID}.feature
```
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

## Multi-Service Projects

### Setup

In `.agent/project-context.yaml`, declare each service:

```yaml
services:
  - name: web-admin
    module: react
    description: Admin dashboard (React SPA)
  - name: api-backend
    module: java-spring
    description: REST API backend
  - name: app-mobile
    module: flutter
    description: Customer mobile app
```

### How it works

Once `services` is configured, every command in the pipeline becomes service-aware:

| Command | Service context |
|---------|----------------|
| `/define-product` | Asks "Which service is this feature for?" |
| `/generate-prd` | Reads service from product-definition metadata |
| `/generate-bdd` | Reads service from PRD; writes to `specs/bdd/{domain}/{service}/{UC-ID}.feature` |
| `/generate-tech-docs` | Reads service from `.feature` header; writes to `tech-docs/{domain}/{service}/` |
| `/generate-code` | Reads service from `.feature` header; loads correct component catalog |
| `/generate-tests` | Reads service from `.feature` header; uses platform-correct test framework |

### Service context tracing

Service identity is carried through artifacts as trace headers:

```gherkin
# @trace.service: web-admin
# @trace.module: react
```

```markdown
@trace.service: web-admin
@trace.module: react
```

This ensures downstream commands always know which service they're generating for, even when files are opened independently.

### Orchestration in multi-service projects

When `/generate-bdd` detects a large PRD (> 3 UCs or > 300 lines), it automatically spawns sub-agents — one per UC. Each sub-agent receives the `active_service` and `active_module` in its context payload, so paths and platform vocabulary are always correct.

---

### Monorepo — Separate Workspaces per Service

In teams where BE, Web, and Mobile are reviewed independently, having one person generate code for all services creates noise and makes PRs hard to review. The recommended pattern is to split each service into its own sub-directory — each developer opens Claude Code only in their service folder.

#### Directory structure

```
my-project/
├── specs/                              ← PO/BA manages this (shared)
│   ├── product-definition/
│   ├── prd/
│   ├── bdd/
│   ├── tech-docs/
│   └── domain-knowledge/
│
├── .agent/                             ← PO/BA plugin install (root)
│   └── project-context.yaml           ← lists all services
├── .claude/commands/                   ← PO/BA command shortcuts
│
├── backend/                            ← BE dev opens Claude Code here
│   ├── CLAUDE.md                       ← BE-specific architecture
│   ├── .agent/
│   │   └── project-context.yaml       ← BE config, paths → ../specs/
│   ├── .claude/commands/
│   └── src/
│
├── web-admin/                          ← Web dev opens Claude Code here
│   ├── CLAUDE.md
│   ├── .agent/
│   │   └── project-context.yaml       ← Web config, paths → ../specs/
│   ├── .claude/commands/
│   └── src/
│
└── app-mobile/                         ← Mobile dev opens Claude Code here
    ├── CLAUDE.md
    ├── .agent/
    │   └── project-context.yaml       ← Mobile config, paths → ../specs/
    ├── .claude/commands/
    └── src/
```

#### Install — one command from project root

```bash
npx @anhth2/spec-driven-dev-plugin --init \
  --services backend:java-spring,web-admin:react,app-mobile:flutter
```

This single command:
- Installs the framework at **root** (PO/BA workspace)
- Installs into **each service subfolder** (`backend/`, `web-admin/`, `app-mobile/`)
- Generates a starter `.agent/project-context.yaml` in each location (paths pre-configured)
- Creates `.claude/commands/` shortcuts in every workspace

#### Per-service `project-context.yaml`

Each service config points paths to the shared `specs/` folder at the root:

```yaml
# backend/.agent/project-context.yaml
project:
  name: My Project — Backend

tech_stack:
  language: Java 17
  framework: Spring Boot 3.2
  module: java-spring

paths:
  specs_dir: ../specs/bdd
  prd_dir: ../specs/prd
  tech_docs_dir: ../specs/tech-docs
  domain_knowledge_dir: ../specs/domain-knowledge
  business_dictionary: ../specs/domain-knowledge/business-dictionary.md
  core_entities: ../specs/domain-knowledge/core-entities.md
  refinement_dir: ../specs/.review
  product_definitions_dir: ../specs/product-definition
  trace_dir: ../.trace
```

Web and Mobile configs follow the same pattern with their respective `tech_stack.module`.

#### Who works where

| Role | Opens Claude Code at | Commands used |
|------|---------------------|---------------|
| PO / BA | `my-project/` (root) | `/define-product` `/generate-prd` `/refine-prd` `/review-context` `/generate-bdd` |
| Tech Lead | `my-project/` (root) | `/generate-bdd` `/validate-traces` |
| BE Dev | `my-project/backend/` | `/generate-tech-docs` `/generate-code` `/review-code` `/generate-tests` `/run-tests` |
| Web Dev | `my-project/web-admin/` | `/generate-tech-docs` `/generate-code` `/review-code` `/generate-tests` `/run-tests` |
| Mobile Dev | `my-project/app-mobile/` | `/generate-tech-docs` `/generate-code` `/review-code` `/generate-tests` `/run-tests` |

Each developer's Claude Code session only sees their service's `project-context.yaml` — no accidental cross-service code generation. Each `/generate-code` run produces a separate branch and PR scoped to one service and one reviewer.

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
| `/sync-figma-components {module}` | Module name | Updated `figma-components/{module}.md` | After Figma design system changes |
| `/sync-figma-tokens {url?}` | Optional Figma URL | Updated `figma-tokens.md` | After design token changes |

### Feature lifecycle

| Command | Input | Output | When to use |
|---------|-------|--------|-------------|
| `/define-product` | — | `specs/product-definition/*.md` | Starting any new feature |
| `/generate-prd` | product-definition file | `specs/prd/{domain}/{slug}.md` | After define-product |
| `/refine-prd` | prd file | `.agent/review/{prd-slug}-findings.yaml` | After generate-prd |
| `/review-context` | prd or feature file | `.agent/review/*-findings.yaml` | After PRD update; after generate-bdd |
| `/review-context --fix` | prd or feature file | Applies all auto-fixable findings immediately | Dev quick-fix without Review Board |
| `/review-context --resume` | prd or feature file | Applies accepted findings | After reviewing in Review Board |
| `/generate-bdd` | prd file | `specs/bdd/{domain}/{slug}.feature` | After PRD approved |
| `/generate-tech-docs` | feature file | `tech-docs/{domain}/{slug}-tech-design.md` | After BDD approved |
| `/review-tech-docs` | tech-design file | `.agent/review/*-findings.yaml` | After generate-tech-docs |
| `/review-tech-docs --resume` | tech-design file | Applies accepted findings | After reviewing in Review Board |
| `/generate-code` | feature file | `src/...` | After tech-design approved |
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

---

## Command Internals — Step Architecture

Every command in `.agent/commands/` is composed of shared **step files** in `.agent/steps/`. These steps are the foundation — every command calls them in sequence before running its own logic.

### Step files at a glance

| File | Role | Called by |
|------|------|-----------|
| `gate.md` | Universal entry — model check, file resolution, context load, user checkpoint | Every command |
| `context-loader.md` | 7-step context loading sequence (stack → arch → safety → domain → UI → recap) | `gate.md` Step 2 |
| `spawn-agent.md` | Sub-agent orchestration for large PRDs | `generate-bdd`, `generate-code`, `generate-tests` |
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

### context-loader.md — 7-Step Context Loading Sequence

Loads all project context into working memory, in strict priority order:

| Step | Priority | What loads |
|------|----------|-----------|
| 1 | PROJECT-CONFIG | `project-context.yaml` → stack, conventions, domains, services, paths |
| 2 | PROJECT-CONFIG | `.agent/modules/{module}/stack-profile.yaml` → framework-specific layer patterns |
| 3 | **CRITICAL** | `CLAUDE.md` → architecture layers, coding standards, naming conventions |
| 4 | SAFETY | `.agent/rules/data-protection.md` → sensitive file patterns never to access |
| 5 | DOMAIN | `business-dictionary.md` → canonical terms + banned terms (enforced for entire session) |
| 6 | DOMAIN | `core-entities.md` → entity catalog (field names, types, invariants, relationships) |
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
Dict       : loaded — 42 canonical terms, 8 banned terms
Entities   : loaded — User, Order, Product
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
