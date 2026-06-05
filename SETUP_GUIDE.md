# Hướng Dẫn Setup Spec-Driven Development

Hướng dẫn này dành cho developer mới tham gia team cần setup workflow Spec-Driven Dev vào dự án.

---

## Mục Lục

1. [Yêu cầu](#1-yêu-cầu)
2. [Cài đặt một lần (per machine)](#2-cài-đặt-một-lần-per-machine)
2b. [Multi-Repo / Umbrella Setup](#2b-multi-repo--umbrella-setup)
3. [Setup project (làm 1 lần khi join project)](#3-setup-project)
4. [Tùy chỉnh cấu hình project](#4-tùy-chỉnh-cấu-hình-project)
5. [Cài Spec Driven Dev Extension (khuyến nghị)](#5-cài-spec-driven-dev-extension)
6. [Bắt đầu feature đầu tiên](#6-bắt-đầu-feature-đầu-tiên)
7. [Workflow đầy đủ](#7-workflow-đầy-đủ)
8. [Command Reference](#8-command-reference)
9. [Câu hỏi thường gặp](#9-câu-hỏi-thường-gặp)

---

## 1. Yêu Cầu

| Tool | Version | Link |
|------|---------|------|
| Claude Code CLI | Latest | [claude.ai/code](https://claude.ai/code) |
| VS Code | ≥ 1.85 | [code.visualstudio.com](https://code.visualstudio.com) |
| Git | Any | |

> **Lưu ý:** Claude Code cần có subscription (Claude Pro / Team / API key).

---

## 2. Cài Đặt Framework

> Chạy từ **thư mục root của project** của bạn.

### Cách mới — `--init` (khuyến nghị)

```bash
# Cài đặt framework vào .agent/ (commit vào git, cả team dùng chung):
npx @anhth2/spec-driven-dev-plugin --init

# Hoặc kèm stack module:
npx @anhth2/spec-driven-dev-plugin --init --module java-spring --hooks
```

Kết quả:
- `.agent/` — chứa toàn bộ framework files (commit vào git, shared với team)
- `.claude/commands/` — shortcut files trỏ về `.agent/commands/`
- `.agent/FRAMEWORK_VERSION` — tracking version để upgrade sau này

**Sau khi init:**
```bash
git add .agent/ .claude/commands/
git commit -m "chore: init spec-driven-dev v0.5.0"
```

### Upgrade lên version mới

```bash
bash scripts/upgrade.sh
# hoặc:
npx @anhth2/spec-driven-dev-plugin@latest --init
```

### Cách cũ — global / per-project (legacy)

```bash
# Global — cài một lần, dùng ở mọi project (không commit vào git):
npx @anhth2/spec-driven-dev-plugin

# Per-project — chỉ dùng trong project hiện tại:
npx @anhth2/spec-driven-dev-plugin --project
```

### Kiểm tra cài đặt thành công

Mở Claude Code tại project, gõ `/` và bạn sẽ thấy:

```
/setup-ai-first
/define-product
/generate-prd
/refine-prd
/generate-bdd
...
```

---

## 2b. Multi-Repo / Umbrella Setup

Dành cho project có nhiều service repo riêng (microservices, multi-platform). Pattern khuyến nghị là dùng **umbrella repo** — repo tổng chứa các service repo dưới dạng git submodule.

### Cấu trúc

```
my-project-be/                ← Umbrella repo (mở Claude Code ở đây)
├── .agent/
│   └── project-context.yaml  ← config routing
├── my-project-specs/         ← submodule: repo spec của PO
├── user-service/             ← submodule: microservice 1
└── order-service/            ← submodule: microservice 2
```

### Cài đặt

Chạy từ thư mục umbrella repo:

```bash
# BE umbrella với nhiều microservices
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services user:java-spring,order:java-spring

# Web umbrella với một NextJS app
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services web-app:nextjs
```

**Lưu ý:** KHÔNG cài framework vào bên trong service submodule — chỉ cài ở umbrella level.

### File đứng ở đâu?

| Loại file | Nằm trong repo | Ai commit |
|-----------|---------------|-----------|
| PRD, Design Spec | Spec submodule | PO team |
| BDD feature files | Service submodule (`user-service/specs/bdd/`) | Dev team |
| Tech docs | Service submodule (`user-service/specs/tech-docs/`) | Dev team |
| Source code | Service submodule (`user-service/src/`) | Dev team |

### Routing tự động

Khi chạy lệnh như `/generate-bdd free-trial-specs/specs/prd/user/FEAT-01.md`:
- Context-loader đọc domain = `user` từ path
- Tự động route `specs_dir` → `user-service/specs/bdd`
- BDD được generate vào đúng submodule

**Trước khi chạy `/generate-bdd`, chạy `/review-context` trước:**

```bash
/review-context free-trial-specs/specs/prd/user/FEAT-01-prd.md
```

Trong umbrella mode, `/review-context` chạy thêm **P0 — Umbrella Routing Check**:

| Check | Mô tả | Nếu fail |
|-------|-------|----------|
| P0.1 | `@trace.domain` có trong PRD frontmatter không? | ❌ critical — phải có trước khi gen BDD |
| P0.2 | Domain value khớp với key trong `services` config không? | ❌ critical — BDD sẽ ra sai chỗ |
| P0.3 | `@trace.status: approved` chưa? | ⚠️ major — không nên gen BDD từ draft PRD |

Nếu P0.1 hoặc P0.2 fail → agent hiện cảnh báo và gợi ý fix trước khi tiếp tục.

### Sau khi cài đặt — review project-context.yaml

File được generate tự động, nhưng cần chỉnh 2 điểm quan trọng:

```yaml
setup:
  mode: umbrella
  spec_source: "my-project-specs"   # ← kiểm tra path này khớp với thư mục submodule thực tế

paths:
  prd_dir: "my-project-specs/specs/prd"   # ← auto-derived, không cần sửa nếu spec_source đúng

services:
  user:                                   # ← đây là domain key — phải khớp với @trace.domain trong PRD
    path: "user-service"                  # ← tên thư mục submodule thực tế
    module: "java-spring"
    specs_dir: "user-service/specs/bdd"
    tech_docs_dir: "user-service/specs/tech-docs"
  order:
    path: "order-service"
    module: "java-spring"
    specs_dir: "order-service/specs/bdd"
    tech_docs_dir: "order-service/specs/tech-docs"
```

> **Quan trọng — domain key phải khớp với `@trace.domain` trong PRD file:**
> Khi PO viết PRD, file sẽ có header như:
> ```
> @trace.domain: user
> @trace.id: FEAT-01
> ```
> Context-loader đọc `@trace.domain: user` → tra trong `services` → route vào `user-service/specs/bdd/`.
> Nếu domain key không khớp → agent sẽ dùng path mặc định (có thể không đúng).

### Chiến lược tài liệu — PRD platform-agnostic (Option C)

PRD mô tả **WHAT** (nghiệp vụ), không phụ thuộc platform:

| ✅ Viết trong PRD | ❌ Không viết trong PRD |
|---|---|
| "Đăng nhập thành công → truy cập tính năng" | "Click button → toast" → Design Spec |
| "Sai password 5 lần → khoá 30 phút" | "API trả về JWT" → Tech Docs |

**Một PRD — nhiều platform:**
- **FE/Web/App** → PRD + Design Spec → BDD (UI scenarios)
- **BE** → PRD trực tiếp → BDD (API scenarios)
- Thêm platform mới sau → **không cần sửa PRD**, chỉ cần setup umbrella mới

### PO chạy trước khi có dev repo

Trường hợp phổ biến: PO setup spec repo và viết PRD nhiều tuần trước khi dev team setup umbrella.

**PO làm việc độc lập (chưa cần dev repo):**
```bash
cd my-project-spec
npx @anhth2/spec-driven-dev-plugin --init
/setup-ai-first        # chọn: 3. PO Spec repo
                       # → agent hỏi domain names → điền vào
/define-product
/generate-prd          # PRD ra specs/prd/{domain}/
/generate-design-spec  # Design spec cho FE/App
```

> ⚠️ **Quan trọng:** Mỗi PRD phải có `@trace.domain` trong frontmatter.
> Domain names PO đặt ở đây sẽ trở thành `services` keys trong umbrella config của dev team.
> **Đồng bộ domain names với dev team trước khi họ setup umbrella.**

**Khi dev team join muộn:**
1. Setup umbrella repo, add spec submodule
2. Dùng đúng domain names của PO làm `services` keys trong `project-context.yaml`
3. Pull PRD đã có: `git submodule update --init --remote {spec-source}`
4. Chạy `/review-context {prd-file}` — P0 check tự động verify domain alignment
5. Nếu P0 OK → `/generate-bdd`, tiếp tục workflow bình thường

### Lấy PRD mới từ PO

Khi PO team push PRD/Design Spec mới, dev cần cập nhật spec submodule trước:

```bash
# Từ umbrella repo — lấy version mới nhất của spec submodule
git submodule update --remote my-project-specs

# Commit lại pointer (để team khác cùng version)
git add my-project-specs
git commit -m "chore: update spec submodule to latest"
git push
```

> **Lưu ý:** Không cần chạy bước này nếu spec submodule đã được update bởi người khác trong team và bạn chỉ cần `git pull` umbrella repo.

### Workflow hằng ngày

**Dev nhận task mới:**
```bash
# 1. Pull umbrella repo
git pull

# 2. Cập nhật spec submodule (lấy PRD mới từ PO)
git submodule update --remote my-project-specs

# 3. Mở Claude Code tại umbrella repo
# 4. Chạy lệnh
/review-context my-project-specs/specs/prd/user/FEAT-01-prd.md
/generate-bdd   my-project-specs/specs/prd/user/FEAT-01-prd.md
/generate-tech-docs user-service/specs/bdd/user/FEAT-01.feature
/generate-code  user-service/specs/tech-docs/user/FEAT-01-tech-design.md
```

**Commit files sau khi generate:**
```bash
# BDD/tech-docs/code nằm trong service submodule → commit 2 tầng

# Bước 1: commit trong service submodule
cd user-service
git add specs/bdd/ specs/tech-docs/ src/
git commit -m "feat(FEAT-01): add BDD, tech-docs, and implementation"
git push

# Bước 2: update pointer ở umbrella
cd ..
git add user-service
git commit -m "chore: update user-service to FEAT-01"
git push
```

---

## 3. Setup Project

> **Nếu project đã có sẵn `specs/`, `CLAUDE.md`, `.agent/project-context.yaml`** — bỏ qua bước này, chuyển thẳng đến [Bước 6](#6-bắt-đầu-feature-đầu-tiên).

**Nếu bạn là người đầu tiên setup project:**

1. Mở Claude Code tại root của project
2. Gõ lệnh:

```
/setup-ai-first
```

Lệnh này sẽ tự động tạo cấu trúc thư mục:

```
{project-root}/
├── specs/
│   ├── product-definition/        ← Product definitions
│   ├── prd/                       ← PRD files theo domain
│   ├── bdd/                       ← BDD .feature files
│   └── domain-knowledge/
│       ├── business-dictionary.md ← Canonical terms + banned terms
│       └── core-entities.md       ← Entity glossary cho code generation
├── tech-docs/                     ← Technical design docs
├── .trace/                        ← Traceability matrix files
├── .agent/
│   ├── review/                    ← findings files (PRD / BDD / tech-review)
│   └── project-context.yaml      ← Cấu hình project
└── CLAUDE.md                      ← "System prompt" cho Claude
```

---

## 4. Tùy Chỉnh Cấu Hình Project

Sau khi chạy `/setup-ai-first`, cần điền thông tin thực tế vào **4 file**:

### 4.1 — `CLAUDE.md`

Đây là "system prompt" cho Claude — định nghĩa architecture, coding standards, naming conventions của project.

Mở file và thay tất cả `{{PLACEHOLDER}}` bằng thông tin thực tế:

```yaml
# §1. Project Overview
Project: Payment Service          # ← tên thực
Language: Java 17
Framework: Spring Boot 3.2
Build: mvn clean install -DskipTests
Test: mvn test

Domains: payment, invoice, refund  # ← các domain trong project

# §2. Architecture
ARCHITECTURE:
  layers: "Controller → Facade → Service → Repository"
  rules:
    - "Controllers must not contain business logic"
    - "Services own transaction boundaries"

# §3. Coding Standards
CODING_STANDARDS:
  naming:
    classes: "PascalCase"
    methods: "camelCase"
  patterns:
    mapping: "MapStruct"
    response_wrapper: "ApiResponse<T>"

# §5. Error Handling
ERROR_HANDLING:
  not_found: "ResourceNotFoundException"
  validation: "ValidationException"

# §7. Git Conventions
GIT:
  branch_feature: "feature/PAY-{N}-{slug}"
  commit_feature: "feat(PAY-{N}): {description}"
```

**Các section quan trọng nhất:**
- `§1` — Tech stack để Claude biết generate code đúng ngôn ngữ/framework
- `§2` — Architecture rules để Claude không vi phạm layering
- `§7` — Git conventions để `/generate-code` tạo đúng branch/commit format

### 4.2 — `.agent/project-context.yaml`

Cấu hình machine-readable cho các commands:

```yaml
project:
  name: "Payment Service"
  description: "Handles payment processing, invoicing, and refunds"

tech_stack:
  language: "Java 17"
  framework: "Spring Boot 3.2"
  build_tool: "Maven"
  test_framework: "JUnit 5 + Mockito"
  database: "PostgreSQL"

conventions:
  build_command: "mvn clean install -DskipTests"
  test_command: "mvn test"
  service_run: "mvn spring-boot:run"
  ticket_prefix: "PAY"          # ← prefix Jira ticket

domains:
  - payment
  - invoice
  - refund
```

> **Tip:** Commit cả 4 file vào Git để cả team dùng cùng config.

### 4.3 — `specs/domain-knowledge/business-dictionary.md`

Định nghĩa thuật ngữ chuẩn cho toàn project. AI dùng file này để:
- Kiểm tra tất cả PRD, BDD, code comment không dùng thuật ngữ bị cấm
- Tự động thay banned term bằng canonical term

```markdown
## Canonical Terms
| Canonical Term | Description / Context        |
|----------------|------------------------------|
| Consumer       | Người dùng cuối của platform |
| Order          | Đơn hàng đã được xác nhận   |

## Banned Terms
| ❌ Do NOT use | ✅ Use instead | Reason         |
|---------------|----------------|----------------|
| User          | Consumer       | Ambiguous role |
| Cart          | DraftOrder     | Not finalized  |

## Status / Enum Registry
| Entity | Field  | Allowed Values                      |
|--------|--------|-------------------------------------|
| Order  | status | PENDING, CONFIRMED, SHIPPED, CLOSED |
```

> Managed by: PO / SA team. Commit và cập nhật khi có thuật ngữ mới.

### 4.4 — `specs/domain-knowledge/core-entities.md`

Glossary về các domain entities cho AI sinh code chính xác mà **không cần đọc source code**:

```markdown
## Entity: Order

**Purpose**: Đơn hàng của Consumer sau khi checkout
**Domain**: order
**Storage**: `orders` table — PostgreSQL
**Owner service**: OrderService

| Field       | Type      | Nullable | Description             |
|-------------|-----------|----------|-------------------------|
| id          | UUID      | No       | Primary key             |
| consumerId  | UUID      | No       | FK → Consumer.id        |
| status      | OrderStatus | No     | Enum: PENDING → CONFIRMED → SHIPPED → CLOSED |
| totalAmount | BigDecimal | No      | Tổng tiền (VND)         |

**Business invariants:**
- status chỉ đi một chiều: PENDING → CONFIRMED → SHIPPED → CLOSED
- totalAmount phải bằng sum của tất cả OrderItem.lineTotal

**Relationships:**
- Order 1:N OrderItem — mỗi order có ít nhất 1 item
```

> Managed by: Tech Lead / Architect. Cập nhật khi thêm/đổi entity.

---

## 5. Cài Spec Driven Dev Extension

**Spec Driven Dev** là VS Code extension với 2 panels. **Không bắt buộc nhưng khuyến nghị.**

```bash
code --install-extension edupia-team.spec-driven-dev-team
```

Hoặc VS Code → `Ctrl+Shift+P` → **"Extensions: Install from Marketplace"** → search **Spec Driven Dev**.

VS Code tự động cập nhật khi có version mới.

### Panel 1 — Review Board (v0.3.0)

Đọc `*-findings.yaml` từ `.agent/review/` — hỗ trợ findings từ cả 3 review commands.

| Tính năng | Mô tả |
|-----------|-------|
| **Sidebar panel** | Activity Bar — danh sách findings files với pending count |
| **Multi-command** | Hỗ trợ `/refine-prd`, `/review-context`, `/review-tech-docs` |
| **Lens tabs** | All / QA / DEV / SA / PO hoặc PRD / BDD / TECH tùy loại findings |
| **auto_fixable badge** | `⚡ auto-fix` / `👤 human` trên mỗi finding |
| **4 actions** | Accept · Modify (có note) · Defer · Reject (có lý do) |
| **↩ Change** | Hoàn tác quyết định đã chọn |
| **Progress bar** | Thanh tiến độ theo dõi % đã review |
| **Search** | Tìm kiếm toàn văn qua issue / suggestion / section |
| **Smart Apply** | Spawn terminal chạy đúng lệnh `--resume` theo loại findings file |

### Panel 2 — Living Documentation (v0.3.0)

Đọc `.trace/*.tsv` — dashboard traceability health toàn project.

| Tính năng | Mô tả |
|-----------|-------|
| **Stat cards** | PRDs, Use Cases, Scenarios, Code Coverage%, Test Coverage%, Drift, Gap |
| **Drill-down tree** | PRD → UC → per-scenario table (SC ID, Spec ver, Gen ver, Code, Tests, Status) |
| **Status badges** | ✅ OK · ⚠️ DRIFT · 🔴 GAP · UNTRACKED |
| **Filter & Search** | Filter theo PRD status, UC status, SC status; search theo ID/title |
| **Live reload** | Tự cập nhật khi `.trace/*.tsv` thay đổi |

---

## 6. Bắt Đầu Feature Đầu Tiên

Mở Claude Code tại root project, rồi làm theo thứ tự:

```
Bước 1 — Define product
  /define-product

Bước 2 — Tạo PRD
  /generate-prd specs/product-definition/{tên-file}.md

Bước 3 — Review PRD với AI (refine + AI review)
  /refine-prd specs/prd/{domain}/{tên-file}.md

  → Claude ghi kết quả vào .agent/review/{prd-slug}-findings.yaml
  → Mở Review Board (sidebar panel hoặc right-click file)
  → Accept / Modify / Defer / Reject từng finding
  → Bấm "Apply to PRD" khi review xong

Bước 4 — Kiểm tra chất lượng PRD
  /review-context specs/prd/{domain}/{tên-file}.md

  → Claude ghi kết quả vào .agent/review/{prd-slug}-review-context-findings.yaml
  → Mở Review Board (cùng UI với Bước 3) → Accept / Modify / Reject từng finding
     • auto_fixable = true  → Accept → AI tự sửa
     • auto_fixable = false → Modify + viết note quyết định (vd: "SLA = 3 ngày")
  → Sau khi review xong:
  /review-context --resume specs/prd/{domain}/{tên-file}.md
  → Claude apply accepted findings, bump PRD version, ghi Changelog

Bước 5 — Tạo BDD Spec
  /generate-bdd specs/prd/{domain}/{tên-file}.md

Bước 6 — Review BDD
  /review-context specs/bdd/{domain}/{UC-ID}.feature

  → Claude ghi kết quả vào .agent/review/{UC-ID}-review-bdd-findings.yaml
  → Mở Review Board → Accept / Modify / Reject từng finding
  → Sau khi review xong:
  /review-context --resume specs/bdd/{domain}/{UC-ID}.feature
  → Claude apply: thêm scenarios còn thiếu, sửa terminology, fix Gherkin, bump bdd_version

Bước 7 — Tạo Technical Design
  /generate-tech-docs specs/bdd/{domain}/{UC-ID}.feature

Bước 8 — Review Technical Design
  /review-tech-docs tech-docs/{domain}/{UC-ID}-tech-design.md

  → Claude ghi kết quả vào .agent/review/{UC-ID}-tech-review-findings.yaml
  → Mở Review Board → Accept / Modify / Reject từng finding
     • Architecture violations (T1) → Modify + ghi cách fix cụ thể
     • Field name drift (T2) → Accept → AI tự rename
     • Missing sections (T6) → Accept → AI thêm skeleton
  → Sau khi review xong:
  /review-tech-docs --resume tech-docs/{domain}/{UC-ID}-tech-design.md
  → Claude apply accepted findings, bump revision, reset status về draft

Bước 9 — Generate Code
  /generate-code specs/bdd/{domain}/{UC-ID}.feature

Bước 10 — Review Code
  /review-code

Bước 11 — Generate Tests
  /generate-tests specs/bdd/{domain}/{UC-ID}.feature

Bước 12 — Chạy Tests
  /run-tests

Bước 13 — Validate Traceability
  /validate-traces {domain}
```

---

## 7. Workflow Đầy Đủ

```
┌─────────────────────────────────────────────────────────────────┐
│                 SPEC-DRIVEN DEVELOPMENT WORKFLOW                 │
└─────────────────────────────────────────────────────────────────┘

PHASE 1 — DISCOVERY
  /define-product
      │
      ▼
  specs/product-definition/{slug}.md

PHASE 2 — PRD
  /generate-prd {product-definition-file}
      │
      ▼
  specs/prd/{domain}/{slug}.md
      │
  /refine-prd {prd-file}
      │
      ▼
  .agent/review/{prd-slug}-findings.yaml
      │
      ▼
  [Human Review via Review Board]
  Accept / Modify / Defer / Reject
      │
      ▼
  Update PRD
      │
  /review-context {prd-file}
      │
      ▼
  .agent/review/{prd-slug}-review-context-findings.yaml
      │
  [Review Board: Accept/Modify/Reject] ← viết note cho auto_fixable=false
      │
  /review-context --resume {prd-file}  ← apply findings + bump version

PHASE 3 — SPEC & DESIGN
  [FE/App only] /generate-design-spec {prd-file}
      │
      ▼
  specs/design-spec/{domain}/{TICKET-ID}-design-spec-{platform}.md
      │
  [Designer review + PO sign-off]
      │
  /generate-bdd {prd-file}
      │
      ▼
  specs/bdd/{domain}/{UC-ID}.feature
      │
  /review-context {feature-file}
      │
      ▼
  .agent/review/{uc-id}-review-bdd-findings.yaml
      │
  [Review Board: Accept/Modify/Reject]
      │
  /review-context --resume {feature-file}  ← apply + bump bdd_version
      │
  /generate-tech-docs {feature-file}
      │
      ▼
  tech-docs/{domain}/{UC-ID}-tech-design.md
      │
  /review-tech-docs {tech-design}
      │
      ▼
  .agent/review/{uc-id}-tech-review-findings.yaml
      │
  [Review Board: Accept/Modify/Reject]
      │
  /review-tech-docs --resume {tech-design}  ← apply + bump revision

PHASE 4 — CODE
  /generate-code {feature-file}
      │
      ▼
  src/... (với @trace.implements tags)

  /review-code
      │
      ▼
  Review report (Critical / Major / Minor)
      │
  [Fix CRITICAL / MAJOR issues]

PHASE 5 — TEST
  /generate-tests {feature-file}
      │
      ▼
  src/test/... (với @trace.verifies tags)

  /run-tests
  /smoke-test                    ← optional, kiểm tra live endpoint

  /validate-traces {domain}      ← check coverage & drift
```

---

## 8. Command Reference

| Command | Input | Output | Khi nào dùng |
|---------|-------|--------|--------------|
| `/setup-ai-first` | — | Cấu trúc thư mục + 4 config files | Lần đầu setup project |
| `/define-product` | — | `specs/product-definition/*.md` | Bắt đầu feature mới |
| `/generate-prd` | product-definition file | `specs/prd/**/*.md` | Sau define-product |
| `/refine-prd` | prd file | `.agent/review/*-findings.yaml` | Sau generate-prd |
| `/review-context` | prd hoặc feature file | `.agent/review/*-findings.yaml` | Sau update PRD; sau generate-bdd |
| `/review-context --fix` | prd hoặc feature file | Apply ngay tất cả auto-fixable findings | Dev quick-fix không qua Review Board |
| `/review-context --resume` | prd hoặc feature file | Apply accepted findings vào file | Sau review trong Review Board |
| `/generate-design-spec` | prd file | `specs/design-spec/**/*.md` | FE/App: sau PRD approved, trước BDD |
| `/generate-bdd` | prd file | `specs/bdd/**/*.feature` | Sau PRD approved (+ Design Spec sign-off nếu FE/App) |
| `/generate-tech-docs` | feature file | `tech-docs/**/*-tech-design.md` | Sau BDD approved |
| `/review-tech-docs` | tech-design file | `.agent/review/*-findings.yaml` | Sau generate-tech-docs |
| `/review-tech-docs --resume` | tech-design file | Apply findings vào file | Sau review trong Review Board |
| `/generate-code` | feature file | `src/...` | Sau tech-design approved |
| `/review-code` | — | Review report | Sau generate-code |
| `/generate-tests` | feature file | `src/test/...` | Sau generate-code |
| `/run-tests` | — | Test report | Sau generate-tests |
| `/smoke-test` | — | Live endpoint check | Sau deploy |
| `/fix-bug` | — | Bug fix | Khi có bug |
| `/debug` | — | Debug session | Khi cần debug sâu |
| `/validate-traces` | domain | Coverage matrix | Bất kỳ lúc nào |

---

## 9. Câu Hỏi Thường Gặp

### Q: Tôi không thấy commands khi gõ `/` trong Claude Code?

**A:** Chạy lại lệnh cài đặt (Bước 2). Commands được lưu tại `~/.claude/commands/` (global).

Kiểm tra:
```bash
# Mac/Linux
ls ~/.claude/commands/

# Windows PowerShell
ls "$env:USERPROFILE\.claude\commands\"
```

---

### Q: Mở Review Board như nào?

**A:** 3 cách:

1. **Sidebar panel** (khuyến nghị) — Click icon trên Activity Bar → danh sách findings files hiện ra → click file cần review
2. **Right-click** — Right-click vào file `*-findings.yaml` trong Explorer → "Open Review Board"
3. **Command Palette** — `Ctrl+Shift+P` → gõ "Spec Driven Dev" → chọn "Open Review Board" → chọn file từ danh sách

---

### Q: Review Board mở lên nhưng không hiển thị data?

**A:** Đảm bảo:
1. File có đuôi `-findings.yaml` (ví dụ: `create-invoice-findings.yaml`)
2. File nằm trong thư mục `.agent/review/`
3. **Đóng hẳn và mở lại VS Code** (không chỉ Reload Window)

---

### Q: Mở Living Documentation như nào?

**A:** `Ctrl+Shift+P` → **"Spec Driven Dev: Open Living Documentation"**.

Nếu dashboard trống, chạy `/generate-bdd` cho ít nhất 1 feature để tạo file `.trace/{UC-ID}.tsv` đầu tiên.

---

### Q: Project đã có cấu trúc thư mục rồi, có cần chạy `/setup-ai-first` không?

**A:** Không cần. Chỉ cần đảm bảo có `CLAUDE.md` và `.agent/project-context.yaml`. Nếu chưa có thì tạo thủ công theo mẫu ở Bước 4.

---

### Q: Có thể dùng workflow này với ngôn ngữ/framework nào?

**A:** Có. Các commands đều generic — chỉ cần điền đúng tech stack vào `CLAUDE.md` và `project-context.yaml`. Claude sẽ generate code phù hợp với ngôn ngữ/framework đó.

---

### Q: `@trace.implements` tag viết ở đâu?

**A:** Trong comment của controller method (hoặc tương đương), adapt theo syntax ngôn ngữ:

**Java:**
```java
/**
 * @trace.implements=PAY-UC01-SC1
 * @trace.source=specs/bdd/payment/PAY-UC01.feature
 */
@PostMapping("/invoices")
public ResponseEntity<ApiResponse<InvoiceResponse>> createInvoice(...) { ... }
```

**TypeScript/JavaScript:**
```typescript
/**
 * @trace.implements PAY-UC01-SC1
 * @trace.source specs/bdd/payment/PAY-UC01.feature
 */
async createInvoice(...) { ... }
```

**Python:**
```python
def create_invoice(...):
    """
    @trace.implements PAY-UC01-SC1
    @trace.source specs/bdd/payment/PAY-UC01.feature
    """
```

---

### Q: Upgrade framework lên version mới như nào?

**A:** Nếu cài theo cách mới (`--init`):
```bash
bash scripts/upgrade.sh
```

Hoặc tương đương:
```bash
npx @anhth2/spec-driven-dev-plugin@latest --init
# Sau đó review và commit .agent/:
git diff .agent/
git add .agent/ && git commit -m "chore: upgrade spec-driven-dev v{old} → v{new}"
```

---

### Q: `.agent/` nên commit vào git không?

**A:** **Có, nên commit.** Đây là điểm khác biệt so với cách cài cũ:
- `.agent/` chứa toàn bộ framework files (commands, steps, rules, templates)
- Khi commit, mọi developer trong team đều có framework ngay sau `git pull`
- Khi upgrade, chỉ cần `git diff .agent/` để xem framework thay đổi gì

---

### Q: Uninstall commands như nào?

**Mac/Linux:**
```bash
rm ~/.claude/commands/{setup-ai-first,define-product,generate-prd,refine-prd,review-context,generate-design-spec,generate-bdd,generate-tech-docs,review-tech-docs,generate-code,review-code,generate-tests,run-tests,smoke-test,fix-bug,debug,validate-traces}.md
```

**Windows (PowerShell):**
```powershell
"setup-ai-first","define-product","generate-prd","refine-prd","review-context","generate-design-spec","generate-bdd","generate-tech-docs","review-tech-docs","generate-code","review-code","generate-tests","run-tests","smoke-test","fix-bug","debug","validate-traces" | ForEach-Object { Remove-Item "$env:USERPROFILE\.claude\commands\$_.md" -ErrorAction SilentlyContinue }
```

---

## Liên Kết

- [spec-driven-dev plugin](https://github.com/TranHongAnh2k/spec-driven-dev)
- [spec-driven-dev extension](https://github.com/TranHongAnh2k/review-board) (Review Board + Living Docs)
- [Claude Code Documentation](https://docs.anthropic.com/claude/claude-code)
